# Ügyfélszolgálati jegykezelő rendszer dokumentációja

**base_url:** `http://127.0.0.1/ugyfelszolgalat/public/api` vagy `http://127.0.0.1:8000/api`

## Funkciók:

**1. Autentikáció (Login, token kezelés)**
---
- A rendszer biztosítja a felhasználók azonosítását és hitelesítését.
- Funkcionális követelmények
- A felhasználó felhasználónév és jelszó segítségével jelentkezhet be.

Sikeres bejelentkezés után:
- a rendszer authentikációs tokent generál,
- a token azonosítja a felhasználót a további műveletek során.
- Hibás bejelentkezési adatok esetén a rendszer megtagadja a hozzáférést.

**2. Felhasználó beiratkozása kurzusra**
---
- A student felhasználók kurzusokra iratkozhatnak fel.
- A rendszer listázza az elérhető kurzusokat.

Egy felhasználó:
- kiválaszthat egy kurzust, beiratkozhat rá.
- ugyanarra a kurzusra csak egyszer iratkozhat fel.

**3. Admin funkciók**
---
- Az admin felhasználó a rendszer karbantartásáért felel.

Admin jogosultsága:
- felhasználók kezelése (pl. törlése)
- kurzusok létrehozása
- beiratkozások és teljesítések áttekinthetőek
- Az admin nem tanuló, nem szükséges kurzusokra beiratkoznia.

**4. Adatbázis logikai felépítés (röviden)**
---
### A rendszer az alábbi fő entitásokat kezeli:
- Felhasználók (users)
- Kurzusok (courses)
- Beiratkozások (enrollments)
- Teljesítési állapot 

Az adatbázis neve:
```bash
ugyfelszolgalat_db
```
