# PRODUCTION KOD AUDITI VA XAVFSIZLIK HISOBOTI

**Ob'ekt:** FastAPI / SQLAlchemy foydalanuvchilar va premium xaridlar servisi

**Auditor roli:** Senior Backend Engineer & Security Auditor

**Hujjat maqomi:** Yakuniy texnik tahlil hisoboti

---

## 1. MA'LUMOTLAR KLASSIFIKATSIYASI

Kodning taqdim etilgan qismini tahlil qilish asosida faktlar, taxminlar va noma'lumlar alohida ajratildi:

* **Tasdiqlangan faktlar (Koddagi aniq ko'ringan holatlar):**
* `User` modeli SQLAlchemy ORM `Base` sinfidan meros olgan, lekin `Base` import qilinmagan.
* FastAPI yo'nalishlarida (`endpoints`) avtorizatsiya va autentifikatsiya middleware/guard mexanizmi umuman yo'q.
* `profile()` funksiyasida `user` ob'ekti mavjudligi tekshirilmaydi.
* `db: Session` parametri FastAPI'ning `Depends()` mexanizmiga biriktirilmagan.
* `purchase()` funksiyasida qat'iy ma'lumotlar bazasi blokirovkasi (`FOR UPDATE`) qo'llanilmagan.


* **Noma'lumlar (Koddagi yo'q va tashqi konfiguratsiyaga bog'liq holatlar):**
* `Base` ob'ekti qaysi fayldan deklaratsiya qilingani va bazaga qanday bog'langani.
* Ma'lumotlar bazasi sessiyasini boshqaruvchi dependency generator (`get_db`) mavjudligi.
* Tizimda JWT yoki Telegram Mini App `initData` tekshiruvi tashqi gateway/proxy darajasida bor-yo'qligi.
* Baza va server vaqt mintaqasi (UTC yoki Local Timezone) konfiguratsiyasi.



---

## 2. KODDAGI FAKTIK VA MANTIQIY XATOLAR

### 1. Model va Import Sintaksis Xatosi (`NameError`)

* **Koddagi o'rni:**
```python
class User(Base):

```


* **Sababi:** `Base` deklaratsiyasi yoki importi (`from database import Base`) mavjud emas.
* **Real oqibati:** Dastur ishga tushish bosqichida (`startup phase`) darhol `NameError: name 'Base' is not defined` xatosi bilan to'xtaydi (crash qiladi).

### 2. FastAPI Dependency Injection Xatosi

* **Koddagi o'rni:**
```python
def give_premium(user_id: int, days: int, db: Session):

```


* **Sababi:** `db: Session` parametri `Depends()` yordamida uzatilmagan.
* **Real oqibati:** FastAPI bu parametrni so'rovning Body (JSON) yoki Query parametri deb qabul qiladi. Mijoz tomonidan `db` obyektini JSON shaklida yuborib bo'lmagani sababli, barcha API so'rovlari **HTTP 422 Unprocessable Entity** xatosini qaytaradi.

### 3. Nollik qiymatni (`None`) Stringga o'girish xatosi

* **Koddagi o'rni:**
```python
"premium_expires_at": str(user.premium_expires_at)

```


* **Sababi:** Agar foydalanuvchida Premium obuna bo'lmasa, `user.premium_expires_at` qiymati `None` ga teng bo'ladi.
* **Real oqibati:** Python `str(None)` natijasida `"None"` string matnini qaytaradi. JSON javobida `null` o'rniga `"None"` matni chiqadi. Mijoz (frontend) ushbu matnni sana deb parse qilishga urinib, JavaScript/Dart darajasida xatolikka uchraydi.

### 4. Manfiy Qiymatlar Validatsiyasi Yo'qligi

* **Koddagi o'rni:**
```python
def give_premium(user_id: int, days: int, db: Session):
def purchase(user_id: int, price: int, db: Session):

```


* **Sababi:** `days` va `price` parametrlari pydantic schema (DTO) yoki `Path/Query` validatsiyasi orqali tekshirilmagan.
* **Real oqibati:**
* `days = -10` yuborilsa, foydalanuvchining Premium muddati o'tmishga o'tib ketadi.
* `price = -100` yuborilsa, `user.balance -= (-100)` amali bajarilib, foydalanuvchining balansi kamayish o'rniga **ko'payib ketadi** (balansni noqonuniy to'ldirish).



---

## 3. XAVFSIZLIK ZAIFLIKLARI (SECURITY AUDIT)

### 1. Admin Huquqlarini Aylanib O'tish (Broken Access Control)

* **Koddagi o'rni:** `@app.post("/admin/users/{user_id}/premium")`
* **Zaiflik turi:** CWE-285: Improper Authorization.
* **Tahlil:** Yo'nalish nomida `/admin/` so'zi borligiga qaramay, funksiya ichida so'rov yuborgan shaxsning admin ekanligi (`is_admin == True`) yoki sessiya tokeni tekshirilmaydi.
* **Real oqibati:** Har qanday oddiy foydalanuvchi yoki havaskor xaker ushbu API manziliga so'rov yuborib, o'ziga yoki ixtiyoriy ID'ga ega foydalanuvchiga tekinga cheksiz Premium vaqtini berib oladi.

### 2. IDOR (Insecure Direct Object Reference)

* **Koddagi o'rni:** `@app.post("/purchase/{user_id}")` va `@app.get("/profile/{user_id}")`
* **Zaiflik turi:** CWE-639: Authorization Bypass Through User-Controlled Key.
* **Tahlil:** API endpoint to'g'ridan-to'g'ri URL'dan `user_id` qabul qiladi. So mezon yuborayotgan shaxs rostdan ham shu `user_id` egasi ekanligi tekshirilmaydi (autentifikatsiya yo'q).
* **Real oqibati:**
* Hujumchi boshqa foydalanuvchining `user_id`sini ko'rsatib, **boshqaning balansi hisobidan** o'ziga xarid o'tkazishi yoki boshqaning balansini nolga tushirishi mumkin.
* Boshqa foydalanuvchilarning shaxsiy profil va balans ma'lumotlarini ruxsatsiz ko'rish imkoniyati mavjud.



---

## 4. RACE CONDITION VA MA'LUMOTLAR YAXLITLIGI

### 1. Xarid Jarayonidagi Poyga Holati (Race Condition / Double Spending)

* **Koddagi o'rni:**
```python
if user.balance >= price:
    user.balance -= price
    # ...
    db.commit()

```


* **Tahlil (TOCTOU - Time-Of-Check to Time-Of-Use):**
1. Foydalanuvchining balansi `100` birlik. Xizmat narxi `100` birlik.
2. Foydalanuvchi bir vaqtning o'zida parallel ravishda **2 ta** `/purchase/` so'rovini yuboradi.
3. Thread 1 bazadan o'qiydi: `balance = 100`. Tekshiruv: `100 >= 100` (To'g'ri).
4. Thread 2 parallel ravishda bazadan o'qiydi: `balance = 100`. Tekshiruv: `100 >= 100` (To'g'ri).
5. Thread 1 balansdan 100 ayiradi, `balance = 0` qiladi va `commit` bajaradi.
6. Thread 2 ham balansdan 100 ayiradi, `balance = -100` ga tushadi yoki `balance = 0` holatda qayta yozib, **2 marotaba xaridni bajartiradi**.


* **Real oqibati:** Balans manfiyga kirib ketadi yoki bitta balans evaziga ikki barobar ko'p xizmat olinadi (Double Spending).

### 2. Mantiqiy Status va Vaqt Nomutanosibligi (Stale State)

* **Koddagi o'rni:** `user.is_premium = True` va `user.premium_expires_at`
* **Tahlil:** `is_premium` — bu static boolean bayroq. `premium_expires_at` vaqti o'tib ketganidan so'ng, `is_premium` avtomatik ravishda `False` ga o'zgarmaydi.
* **Real oqibati:** Vaqti tugagan obunachilar ham bazada `is_premium = True` bo'lib qolaveradi va tizimdan umrbod tekin foydalanadi.

---

## 5. HTTP 500 (INTERNAL SERVER ERROR) MANBALARI

| Holat | Kod qismi | Sababi |
| --- | --- | --- |
| **Mavjud bo'lmagan foydalanuvchi profili** | `profile()` endpoint: `return {"id": user.id, ...}` | Baza `user = None` qaytaradi. Nollik ob'ektdan attribute o'qishda `AttributeError: 'NoneType' object has no attribute 'id'` otiladi va HTTP 500 yuzaga keladi. |
| **Ma'lumotlar bazasi uzilishi** | `db.query(...)` | Baza bilan aloqa uzilganda unhandled DB error otiladi. |
| **`premium_expires_at` `None` bo'lganda tip xatosi (muayyan drayverlarda)** | `str(user.premium_expires_at)` | Ba'zi maxsus datetime wrapper'larda `None` qiymat berilganda exception otilishi mumkin. |

---

## 6. MUAMMOLARNING JIDDIYLIK DARAJASI (PRIORITY MATRIX)

```
[P0 - KRITIK]  ──> Admin & IDOR Avtorizatsiya teshiklari + Profile HTTP 500 + Purchase Race Condition
[P1 - YUQORI]   ──> Import/Syntax Error (Base missing) + FastAPI Dependency Injection (db missing)
[P2 - O'RTA]    ──> Manfiy qiymatlar validatsiyasi yo'qligi + Premium muddati avto-invalidation yo'qligi
[P3 - QUBUR]    ──> str(None) string xatosi + Local timezone o'rniga UTC ishlatilmagani

```

---

## 7. TEXNIK TUZATISH REJASI (REMEDIATION PLAN)

Kodni qayta yozmasdan, bajarilishi kerak bo'lgan arxitektura va refaktoring qadamlari:

1. **Infrastruktura va Importlar:**
* `Base` obyektini deklaratsiya qilish yoki to'g'ri moduldan import qilish.
* `get_db` generator funksiyasini yozish va FastAPI `Depends(get_db)` yordamida har bir endpointga DB sessiyasini xavfsiz ulash.


2. **Xavfsizlik va Avtorizatsiya (Auth Layer):**
* Pydantic modellar (Schemas) orqali kiruvchi ma'lumotlarni (`price > 0`, `days > 0`) qat'iy validatsiya qilish.
* Auth Middleware / Security Guard joriy etish (JWT yoki Telegram `initData` tekshiruvi).
* URL ichidagi `{user_id}` o'rniga joriy autentifikatsiyadan o'tgan foydalanuvchini so'rov kontekstidan (`current_user`) olish. Admin route'larni faqat `current_user.is_admin == True` bo'lganda ochish.


3. **Poyga Holati (Race Condition) va Tranzaksiyalar:**
* `purchase()` ichidagi SQL so'roviga row-level lock (`SELECT ... FOR UPDATE` yoki `.with_for_update()`) qo'shish. Alternativ: Atomar yangilash bajarish (`User.balance = User.balance - price`).


4. **Profil va Status Ishonchliligi:**
* `profile()` funksiyasida `if not user:` shartini qo'shib, topilmaganda **HTTP 404** otish.
* `is_premium` qiymatini static DB ustunidan emas, `premium_expires_at > datetime.now(timezone.utc)` dinamik solishtirish shaklida hisoblash.



---

## 8. TEST SSENARIYLARI VA VERIFIKATSIYA

Tuzatishlar kiritilgandan so'ng auditni tasdiqlash uchun talab qilinadigan testlar:

### 1. Avtorizatsiya va Xavfsizlik Testlari (Security Unit/Integration Tests)

* **Ssenariy 1.1:** Oddiy foydalanuvchi tokeni bilan `/admin/users/{id}/premium` endpointiga so'rov yuborish.
*Kutiladigan natija:* `HTTP 403 Forbidden`.
* **Ssenariy 1.2:** Autentifikatsiya tokensisiz `/profile/1` yoki `/purchase/1` so'rovlarini yuborish.
*Kutiladigan natija:* `HTTP 401 Unauthorized`.

### 2. Concurrency va Race Condition Testlari (Load / Stress Test)

* **Ssenariy 2.1:** Foydalanuvchi balansi = 100. Bir vaqtning o'zida (simultan ravishda 50 ms ichida) narxi 100 bo'lgan xarid uchun 10 ta parallel HTTP POST request yuborish (Locust yoki `asyncio.gather` vositasida).
*Kutiladigan natija:* Faqat **1 ta** so'rov `HTTP 200 OK` qaytarishi, qolgan 9 tasi `HTTP 400 Insufficient balance` qaytarishi hamda yakuniy balans **0** bo'lishi kerak.

### 3. Edge Case va HTTP 500 Testlari

* **Ssenariy 3.1:** Bazada yo'q bo'lgan `user_id = 99999` uchun `/profile/99999` so'rovini yuborish.
*Kutiladigan natija:* `HTTP 404 Not Found` (HTTP 500 emas).
* **Ssenariy 3.2:** `purchase` so'rovida `price = -500` yoki `days = -5` yuborish.
*Kutiladigan natija:* `HTTP 422 Unprocessable Entity` yoki `HTTP 400 Bad Request`.
