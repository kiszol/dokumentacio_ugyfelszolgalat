# Ügyfélszolgálati jegykezelő REST API

Laravel + Sanctum alapú REST API ügyfélszolgálati jegykezelő rendszerhez.  
A projekt célja egy egyszerű, átlátható backend megvalósítása jegyek és válaszok kezelésére, szerepkör alapú jogosultságkezeléssel.

---

## Fő funkciók

- Felhasználói regisztráció és bejelentkezés
- Token alapú autentikáció (Laravel Sanctum)
- Jegyek létrehozása, listázása, módosítása, törlése
- Jegyekhez tartozó válaszok kezelése
- Szerepkörök: `user`, `admin`
- Automatikus adatbázis seedelés
- Feature tesztek

---

## Technológiai stack

- PHP 8.1+
- Laravel 10+
- Laravel Sanctum
- MySQL / PostgreSQL
- PHPUnit

---

## Adatmodell

### Táblák
```bash
├── app/
│   ├── Models/
│   │   ├── User.php
│   │   ├── Ticket.php
│   │   └── TicketReply.php
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── AuthController.php
│   │   │   ├── TicketController.php
│   │   │   └── TicketReplyController.php
│   │   └── Middleware/
│   └── Policies/
│       └── TicketPolicy.php
│
├── database/
│   ├── migrations/
│   ├── seeders/
│   │   ├── UserSeeder.php
│   │   ├── TicketSeeder.php
│   │   └── TicketReplySeeder.php
│
├── routes/
│   └── api.php
│
├── tests/
│   ├── Feature/
│   │   ├── AuthTest.php
│   │   ├── TicketTest.php
│   │   └── TicketReplyTest.php
│
├── .env.example
├── README.md
└── composer.json
```
Kód másolása

### Kapcsolatok

- User → Ticket (1:N)
- Ticket → TicketReply (1:N)
- User → TicketReply (1:N)

---

## Telepítés

### Repository klónozása

4. Authentikáció
```bash
Laravel Sanctum biztosítja a token alapú authentikációt.

Regisztráció: új user, role=user.

Login: email+jelszó, token visszaadás.

Jogosultságok: user saját jegyek, admin minden jegy.
```
5. Végpontok részletes leírása
Auth
POST /register
```bash
json
{
  "name": "Teszt Elek",
  "email": "teszt@example.com",
  "password": "Jelszo_2025",
  "password_confirmation": "Jelszo_2025"
}
```
Sikeres válasz (201):
```bash
json
{ "message":"User created successfully", "user":{...} }
POST /login
```
```bash
json
{
  "email": "teszt@example.com",
  "password": "Jelszo_2025"
}
```
Sikeres válasz (200):
```bash
json
{ "message":"Login successful", "user":{...}, "access":{"token":"..."} }
POST /logout
```
```bash
json
{ "message":"Logout successful" }
```
User
GET /users/me → Saját profil + statisztika.

PUT /users/me → Profil frissítése.

Tickets
GET /tickets → Saját jegyek (user), minden jegy (admin).

POST /tickets
```bash
json
{
  "subject": "Nem működik a bejelentkezés",
  "description": "A rendszer hibát dob.",
  "priority": "high"
}
```
GET /tickets/:id → Jegy részletei + válaszok.

PATCH /tickets/:id → User subject/description, admin status/priority.

DELETE /tickets/:id → Jegy törlése.

Ticket replies
GET /tickets/:id/replies → Válaszok listázása.

POST /tickets/:id/replies
```bash
json
{
  "message": "Most már működik, köszönöm!"
}
```
