Ügyfélszolgálati jegykezelő REST API – Fejlesztői kézikönyv
1. Bevezetés
Az ügyfélszolgálati jegykezelő rendszer célja, hogy strukturált módon kezelje a felhasználói problémákat és kérdéseket. A REST API lehetővé teszi:

Jegyek létrehozását, listázását, módosítását és törlését.

Jegyekhez kapcsolódó válaszok kezelését.

Felhasználói profilok kezelését.

Adminisztrátori jogosultságokkal a jegyek státuszának és prioritásának módosítását.

A rendszer Laravel + Sanctum alapokon épül, így biztosítja a biztonságos authentikációt és a token alapú hozzáférést.

Célközönség
Fejlesztők: akik integrálni szeretnék az API‑t saját alkalmazásukba.

Ügyfélszolgálati rendszergazdák: akik szeretnék automatizálni a jegykezelést.

QA tesztelők: akik validálni akarják az API működését.

2. Architektúra áttekintés
A rendszer három fő komponensből áll:

Felhasználók (users)

Azonosítják a jegyek tulajdonosait és a válaszokat adó személyeket.

Szerepkörök: user, admin.

Jegyek (tickets)

Egy felhasználó által létrehozott problémát vagy kérdést reprezentálnak.

Attribútumok: subject, description, priority, status.

Jegyválaszok (ticket_replies)

Kommunikációs szálak a jegyhez kapcsolódóan.

Tartalmazzák a felhasználó vagy admin üzenetét.

Adatáramlás
User → Ticket: egy felhasználó létrehoz egy jegyet.

Ticket → Replies: a jegyhez válaszok érkeznek.

Admin → Ticket: admin módosíthatja a státuszt és prioritást.

User/Admin → Replies: mindkét szerepkör adhat választ.

Példa folyamat
Felhasználó regisztrál és bejelentkezik.

Létrehoz egy új jegyet: „Nem működik a bejelentkezés”.

Admin megtekinti a jegyet, státuszt pending‑re állítja.

Admin válaszol: „Kérlek próbáld meg újra, frissítve a böngészőt.”

Felhasználó visszajelez: „Most már működik.”

Admin lezárja a jegyet (closed).

3. Adatbázis séma és relációk
Kód
+--------------------+       +--------------------+       +--------------------+
| users              |       | tickets            |       | ticket_replies     |
+--------------------+       +--------------------+       +--------------------+
| id (PK)            |   1--<| user_id (FK)       |   1--<| ticket_id (FK)     |
| name               |       | subject            |       | user_id (FK)       |
| email (unique)     |       | description (text) |       | message (text)     |
| role (user/admin)  |       | priority (enum)    |       | created_at         |
| password           |       | status (enum)      |       +--------------------+
| created_at         |       | created_at         |
| updated_at         |       | updated_at         |
+--------------------+       +--------------------+
Relációk
User – Ticket: 1:N (egy user több jegyet hozhat létre).

Ticket – Reply: 1:N (egy jegyhez több válasz tartozhat).

User – Reply: 1:N (egy user több választ adhat).

4. Authentikáció és jogosultságkezelés
Az API Laravel Sanctum segítségével biztosítja a token alapú authentikációt.

Regisztráció
Új felhasználó létrehozása.

Alapértelmezett szerepkör: user.

Bejelentkezés
Email + jelszó páros.

Sikeres bejelentkezéskor a rendszer visszaad egy Bearer tokent.

Jogosultságok
User: saját jegyek létrehozása, módosítása, törlése (ha nem closed), válaszok hozzáadása.

Admin: minden jegy megtekintése, státusz és prioritás módosítása, jegyek törlése.

👉 Következő lépésben kidolgozom a végpontok részletes leírását (request/response példákkal, státuszkódokkal), ami önmagában több száz sor lesz. Ezután jönnek a hibakezelési minták, telepítési útmutató, seeding, tesztelés, Swagger definíció, best practice‑ek és biztonsági megfontolások.

Szeretnéd, hogy a következő üzenetben a végpontok részletes leírását kezdjem el kidolgozni, minden egyes endpointhoz külön példával és JSON mintával?

Igen és ahol lehet ott irj kódokat
Szuper, akkor most elkezdem a végpontok részletes leírását, mindenhol ahol lehet kódrészletekkel és JSON példákkal. Ez lesz a dokumentáció legnagyobb része, és fejezetenként haladunk, hogy összeálljon az ~1000 soros kézikönyv.

5. Végpontok részletes leírása
5.1 Auth végpontok
POST /register – Regisztráció
Leírás: Új felhasználó létrehozása. Alapértelmezett szerepkör: user.

Request body:

json
{
  "name": "Teszt Elek",
  "email": "teszt@example.com",
  "password": "Jelszo_2025",
  "password_confirmation": "Jelszo_2025"
}
Validációs szabályok:

name: required, string, max:255

email: required, unique, valid email

password: required, min:8, confirmed

Sikeres válasz (201):

json
{
  "message": "User created successfully",
  "user": {
    "id": 1,
    "name": "Teszt Elek",
    "email": "teszt@example.com",
    "role": "user",
    "created_at": "2025-12-15T08:40:00Z"
  }
}
Hibás válasz (422):

json
{
  "message": "Validation error",
  "errors": {
    "email": ["The email has already been taken."]
  }
}
POST /login – Bejelentkezés
Request body:

json
{
  "email": "teszt@example.com",
  "password": "Jelszo_2025"
}
Sikeres válasz (200):

json
{
  "message": "Login successful",
  "user": {
    "id": 1,
    "name": "Teszt Elek",
    "email": "teszt@example.com",
    "role": "user"
  },
  "access": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer"
  }
}
Hibás válasz (401):

json
{
  "message": "Invalid email or password"
}
POST /logout – Kijelentkezés
Header:

Kód
Authorization: Bearer <token>
Sikeres válasz (200):

json
{
  "message": "Logout successful"
}
5.2 User végpontok
GET /users/me – Saját profil
Header:

Kód
Authorization: Bearer <token>
Sikeres válasz (200):

json
{
  "user": {
    "id": 1,
    "name": "Teszt Elek",
    "email": "teszt@example.com",
    "role": "user"
  },
  "stats": {
    "ticketsOpened": 3,
    "ticketsClosed": 1
  }
}
PUT /users/me – Profil frissítése
Request body:

json
{
  "name": "Teszt Elek Jr.",
  "email": "tesztjr@example.com",
  "password": "UjJelszo_2025",
  "password_confirmation": "UjJelszo_2025"
}
Sikeres válasz (200):

json
{
  "message": "Profile updated successfully",
  "user": {
    "id": 1,
    "name": "Teszt Elek Jr.",
    "email": "tesztjr@example.com",
    "role": "user"
  }
}
5.3 Tickets végpontok
GET /tickets – Jegyek listázása
User: csak saját jegyek. Admin: minden jegy.

Sikeres válasz (200):

json
{
  "data": [
    {
      "id": 5,
      "subject": "Nem működik a bejelentkezés",
      "priority": "high",
      "status": "open",
      "created_at": "2025-12-15T08:40:00Z",
      "user": {
        "id": 1,
        "name": "Teszt Elek"
      }
    },
    {
      "id": 6,
      "subject": "Lassú a rendszer",
      "priority": "medium",
      "status": "pending",
      "created_at": "2025-12-15T09:00:00Z",
      "user": {
        "id": 2,
        "name": "Kovács Béla"
      }
    }
  ]
}
POST /tickets – Jegy létrehozása
Request body:

json
{
  "subject": "Nem működik a bejelentkezés",
  "description": "A rendszer hibát dob, amikor próbálok belépni.",
  "priority": "high"
}
Sikeres válasz (201):

json
{
  "message": "Ticket created successfully",
  "ticket": {
    "id": 12,
    "user_id": 1,
    "subject": "Nem működik a bejelentkezés",
    "description": "A rendszer hibát dob, amikor próbálok belépni.",
    "priority": "high",
    "status": "open",
    "created_at": "2025-12-15T08:40:00Z",
    "updated_at": "2025-12-15T08:40:00Z"
  }
}
GET /tickets/:id – Jegy részletei
Sikeres válasz (200):

json
{
  "ticket": {
    "id": 12,
    "user_id": 1,
    "subject": "Nem működik a bejelentkezés",
    "description": "A rendszer hibát dob, amikor próbálok belépni.",
    "priority": "high",
    "status": "open",
    "created_at": "2025-12-15T08:40:00Z"
  },
  "replies": [
    {
      "id": 20,
      "user_id": 2,
      "message": "Kérlek próbáld meg újra, frissítve a böngészőt.",
      "created_at": "2025-12-15T08:45:00Z"
    }
  ]
}
PATCH /tickets/:id – Jegy módosítása
User: subject, description módosítható, ha status != closed. Admin: status és priority módosítható.

Request body (admin):

json
{
  "status": "pending",
  "priority": "medium"
}
Sikeres válasz (200):

json
{
  "message": "Ticket updated successfully",
  "ticket": {
    "id": 12,
    "subject": "Nem működik a bejelentkezés",
    "priority": "medium",
    "status": "pending"
  }
}
DELETE /tickets/:id – Jegy törlése
Sikeres válasz (200):

json
{
  "message": "Ticket deleted successfully"
}
5.4 Ticket replies végpontok
GET /tickets/:id/replies – Válaszok listázása
Sikeres válasz (200):

json
{
  "data": [
    {
      "id": 20,
      "user_id": 2,
      "message": "Kérlek próbáld meg újra, frissítve a böngészőt.",
      "created_at": "2025-12-15T08:45:00Z",
      "user": {
        "id": 2,
        "name": "Admin",
        "role": "admin"
      }
    }
  ]
}
POST /tickets/:id/replies – Válasz hozzáadása
Request body:

json
{
  "message": "Most már működik, köszönöm!"
}
Sikeres válasz (201):

json
{
  "message": "Reply added successfully",
  "reply": {
    "id": 21,
    "ticket_id": 12,
    "user_id": 1,
    "message": "Most már működik, köszönöm!",
    "created_at": "2025-12-15T08:50:00Z"
  }
}
