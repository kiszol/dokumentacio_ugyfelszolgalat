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

Kód másolása

### Kapcsolatok

- User → Ticket (1:N)
- Ticket → TicketReply (1:N)
- User → TicketReply (1:N)

---

## Telepítés

### Repository klónozása

```bash
git clone https://github.com/felhasznalo/ticketing-api.git
cd ticketing-api
Függőségek telepítése
bash
Kód másolása
composer install
Környezeti változók
bash
Kód másolása
cp .env.example .env
php artisan key:generate
.env fájlban állítsd be az adatbázist:

env
Kód másolása
DB_DATABASE=ticketing
DB_USERNAME=root
DB_PASSWORD=
Migráció és seedelés
bash
Kód másolása
php artisan migrate:fresh --seed
Szerver indítása
bash
Kód másolása
php artisan serve
Authentikáció
A védett végpontok Bearer token alapú hitelesítést használnak.

css
Kód másolása
Authorization: Bearer {token}
API végpontok
Auth
Method	Endpoint	Leírás
POST	/api/register	Regisztráció
POST	/api/login	Bejelentkezés
POST	/api/logout	Kijelentkezés

User
Method	Endpoint	Leírás
GET	/api/users/me	Saját profil

Tickets
Method	Endpoint	Jogosultság
GET	/api/tickets	user, admin
POST	/api/tickets	user
GET	/api/tickets/{id}	user, admin
PATCH	/api/tickets/{id}	user, admin
DELETE	/api/tickets/{id}	admin

Ticket replies
Method	Endpoint
GET	/api/tickets/{id}/replies
POST	/api/tickets/{id}/replies

Hibakódok
Kód	Jelentés
400	Hibás kérés
401	Nem autentikált
403	Jogosultság megtagadva
404	Nem található
422	Validációs hiba

Tesztelés
bash
Kód másolása
php artisan test
A tesztek lefedik:

Regisztráció

Bejelentkezés

Jegy létrehozás

Jegyválasz létrehozás

Alap admin felhasználó (seed)
makefile
Kód másolása
Email: admin@example.com
Jelszó: Admin123
