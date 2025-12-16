# Ügyfélszolgálati jegykezelő rendszer dokumentációja

**base_url:** `http://127.0.0.1/ugyfelszolgalat/public/api` vagy `http://127.0.0.1:8000/api`

**Funkciók:**

1. Autentikáció (Login, token kezelés)
Funkció leírása

A rendszer biztosítja a felhasználók azonosítását és hitelesítését.

Funkcionális követelmények

A felhasználó felhasználónév és jelszó segítségével jelentkezhet be.

Sikeres bejelentkezés után:

a rendszer authentikációs tokent generál,

a token azonosítja a felhasználót a további műveletek során.

Hibás bejelentkezési adatok esetén a rendszer megtagadja a hozzáférést.

Szerepkörök

admin

student

2. Felhasználó beiratkozása kurzusra
Funkció leírása

A student felhasználók kurzusokra iratkozhatnak fel.

Funkcionális követelmények

A rendszer listázza az elérhető kurzusokat.

A student:

kiválaszthat egy kurzust,

beiratkozhat rá.

Egy felhasználó:

ugyanarra a kurzusra csak egyszer iratkozhat fel.

A beiratkozás állapota kezdetben nem teljesített.

3. Kurzus teljesítésének jelölése
Funkció leírása

A felhasználó egy kurzus elvégzése után jelölheti annak teljesítését.

Funkcionális követelmények

Csak olyan kurzus jelölhető teljesítettnek:

amelyre a felhasználó fel van iratkozva.

A rendszer eltárolja:

a teljesítés tényét,

a teljesítés időpontját (opcionális).

A teljesített kurzus státusza megjelenik a felhasználó kurzuslistájában.

4. Admin funkciók
Funkció leírása

Az admin felhasználó a rendszer karbantartásáért felel.

Funkcionális követelmények

Admin jogosultsággal:

felhasználók kezelhetők,

kurzusok létrehozhatók és listázhatók,

beiratkozások és teljesítések áttekinthetők.

Az admin nem tanuló, nem szükséges kurzusokra beiratkoznia.

5. Tesztadatok (Seeder alapú kezdeti állapot)

A rendszer indulásakor az adatbázis az alábbi tesztadatokat tartalmazza.

5.1 Felhasználók

1 admin felhasználó

felhasználónév: admin

jelszó: admin

szerepkör: admin

9 student felhasználó

jelszó: Jelszo_2025

szerepkör: student

5.2 Kurzusok

3 releváns kurzus

egyedi névvel és leírással

aktív állapotban

5.3 Beiratkozások

2 véletlenszerű student:

mindkettő legalább egy kurzusra beiratkozva

a beiratkozások közül:

1 kurzus teljesített

1 kurzus nem teljesített

6. Adatbázis logikai felépítés (röviden)

A rendszer az alábbi fő entitásokat kezeli:

Felhasználók (users)

Kurzusok (courses)

Beiratkozások (enrollments)

Teljesítési állapot (enrollment status / completed flag)

Az adatbázis neve minden esetben:
learning_platform
