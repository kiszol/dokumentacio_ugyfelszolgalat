Ügyfélszolgálati jegykezelő REST API

Laravel + Sanctum alapú ügyfélszolgálati jegykezelő rendszer REST API megvalósítással.
A rendszer támogatja a felhasználók regisztrációját, autentikációját, jegyek és jegyválaszok kezelését szerepkör alapú jogosultságkezeléssel.

Funkcionalitás

Token alapú autentikáció (Laravel Sanctum)

Szerepkörök: user, admin

Jegyek kezelése (CRUD)

Jegyekhez tartozó válaszok

Jogosultságvezérelt hozzáférés

Seederek és automatizált tesztek

Technológiai stack

PHP 8.1+

Laravel 10+

Laravel Sanctum

MySQL / PostgreSQL

PHPUnit

Könyvtárstruktúra (releváns részek)
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

Telepítés és futtatás
1. Repository klónozása
git clone https://github.com/felhasznalo/ticketing-api.git
cd ticketing-api

2. Függőségek telepítése
composer install

3. Környezeti változók
cp .env.example .env
php artisan key:generate


Állítsd be az adatbázis kapcsolatot a .env fájlban:

DB_DATABASE=ticketing
DB_USERNAME=root
DB_PASSWORD=

4. Adatbázis migráció és seedelés
php artisan migrate:fresh --seed

5. Szerver indítása
php artisan serve

Authentikáció

A rendszer Laravel Sanctumot használ.
Minden védett végpont Authorization: Bearer <token> fejlécet vár.

API végpontok összefoglaló
Auth
Metódus	Végpont	Leírás
POST	/api/register	Regisztráció
POST	/api/login	Bejelentkezés
POST	/api/logout	Kijelentkezés
User
Metódus	Végpont	Leírás
GET	/api/users/me	Saját profil
Tickets
Metódus	Végpont	Jogosultság
GET	/api/tickets	user / admin
POST	/api/tickets	user
GET	/api/tickets/{id}	user / admin
PATCH	/api/tickets/{id}	user / admin
DELETE	/api/tickets/{id}	admin
Ticket replies
Metódus	Végpont
GET	/api/tickets/{id}/replies
POST	/api/tickets/{id}/replies
Hibakezelés
HTTP kód	Jelentés
400	Hibás kérés
401	Nem autentikált
403	Jogosultság megtagadva
404	Nem található
422	Validációs hiba
Tesztelés
php artisan test


A tesztek lefedik:

Regisztráció

Bejelentkezés

Jegy létrehozás

Jegyválasz létrehozás

Admin felhasználó (seed)
Email: admin@example.com
Jelszó: Admin123
