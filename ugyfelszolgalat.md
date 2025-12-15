Ügyfélszolgálati jegykezelő REST API megvalósítása Laravel környezetben
base_url: http://127.0.0.1/supportTicketBearer/public/api vagy http://127.0.0.1:8000/api

Az API nyilvános elérésre tervezett backend, ami ügyfélszolgálati jegyek kezelését biztosítja: felhasználók jegyet hozhatnak létre, megtekinthetik és válaszolhatnak rá, adminok pedig priorizálhatják és kezelhetik a státuszokat. A tartalom- és hibakezelési minták, valamint a bearer tokenes authentikáció (Sanctum) a hivatkozott mintát követik, a fejlesztői dokumentáció felépítése és stílusa is arra rímel.

Funkciók:

Auth: regisztráció, bejelentkezés, bearer token kezelés.

Jegykezelés: jegy létrehozása, listázása, megtekintése, módosítása, törlése.

Válaszok: jegyválaszok létrehozása és listázása.

Admin műveletek: jegy státusz és prioritás módosítása, jegyek összesített listázása.

Kötelező headerek:

Content-Type: application/json

Accept: application/json

Végpontok
Érvénytelen vagy hiányzó token esetén a backendnek 401 Unauthorized választ kell adnia. Response: 401 Unauthorized { "message": "Invalid token" }

Nem védett végpontok
GET: /ping

Leírás: Egyszerű elérhetőségi teszt.

Válaszok: 200 OK, { "message": "API works!" }

POST: /register

Leírás: Új felhasználó regisztrációja (alapértelmezett role: "user").

Törzs:

name (string, required)

email (string, required, unique)

password (string, required, min:8)

password_confirmation (string, required, egyezzen)

Siker: 201 Created { "message": "User created successfully", "user": { "id": 13, "name": "Név", "email": "valami@example.com", "role": "user" } }

Hiba: 422 Unprocessable Entity, mezőhiba részletekkel

POST: /login

Leírás: Bejelentkezés email + jelszó.

Törzs: { "email": "valami@example.com", "password": "Jelszo_2025" }

Siker: 200 OK + bearer token { "message": "Login successful", "user": { ... }, "access": { "token": "<token>", "token_type": "Bearer" } }

Hiba: 401 Unauthorized, { "message": "Invalid email or password" }

Authorization header minta minden további autentikált végpontra: Authorization: "Bearer <token>"

Hibák
400 Bad Request: hibás formátum vagy hiányzó mezők.

401 Unauthorized: hiányzó/érvénytelen token.

403 Forbidden: nincs jogosultság (pl. user próbál admin-only műveletet).

404 Not Found: erőforrás nem található (jegy, válasz).

409 Conflict: ütközés (pl. már létező állapot).

422 Unprocessable Entity: validációs hibák.

503 Service Unavailable: nem elérhető szolgáltatás / váratlan hiba.

Jegykezelés
Authentikált végpontok (auth:sanctum)
GET: /users/me

Leírás: Saját profil és jegy-statisztikák.

Válasz: 200 OK { "user": { id, name, email, role }, "stats": { "ticketsOpened": X, "ticketsClosed": Y } }

PUT: /users/me

Leírás: Saját profil frissítése (name, email, password+confirmation).

Siker: 200 OK, { "message": "Profile updated successfully", "user": { ... } }

Hiba: 422 (validáció), 401 (token)

POST: /logout

Leírás: Aktuális felhasználó kijelentkeztetése (token törlése).

Siker: 200 OK, { "message": "Logout successful" }

Tickets
GET: /tickets

Leírás: Jegyek listázása.

User: csak saját jegyek.

Admin: minden jegy.

Query opciók: status, priority, user_id (admin), search (subject).

Siker: 200 OK, { "data": [ { id, subject, priority, status, created_at, user: { id, name } } ... ] }

POST: /tickets

Leírás: Új jegy létrehozása.

Törzs:

subject (string, required, max:255)

description (string, required)

priority (enum: low|medium|high, default: medium)

Siker: 201 Created, { "message": "Ticket created", "ticket": { id, ... } }

Hiba: 422 Unprocessable Entity

GET: /tickets/:id

Leírás: Jegy részletei, válaszokkal.

User: csak a saját jegyét.

Admin: bármely jegy.

Siker: 200 OK { "ticket": { id, user_id, subject, description, priority, status }, "replies": [ { id, user_id, message, created_at } ... ] }

Hiba: 404 Not Found

PATCH: /tickets/:id

Leírás: Jegy módosítása.

User: subject, description módosítás ha status != closed.

Admin: status (open|pending|resolved|closed), priority módosítás.

Siker: 200 OK, { "message": "Ticket updated", "ticket": { ... } }

Hiba: 403 Forbidden (nem saját, vagy closed), 422 Unprocessable Entity

DELETE: /tickets/:id

Leírás: Jegy törlése.

User: saját jegy törlése, ha status != closed.

Admin: bármely jegy törlése (soft delete ajánlott).

Siker: 200 OK, { "message": "Ticket deleted successfully" }

Hiba: 404 Not Found, 403 Forbidden

Ticket replies
GET: /tickets/:id/replies

Leírás: Jegy válaszainak listája.

Siker: 200 OK, { "data": [ { id, user_id, message, created_at, user: { id, name, role } } ... ] }

Hiba: 404 Not Found

POST: /tickets/:id/replies

Leírás: Válasz hozzáadása egy jegyhez.

Törzs:

message (string, required)

Siker: 201 Created, { "message": "Reply added", "reply": { id, ... } }

Hiba: 403 Forbidden (hozzáférés), 422 Unprocessable Entity

Összefoglaló végpont táblázat
HTTP	Útvonal	Jogosultság	Státuszkódok	Rövid leírás
GET	/ping	Nyilvános	200 OK	API teszteléshez
POST	/register	Nyilvános	201, 422	Regisztráció
POST	/login	Nyilvános	200, 401	Bejelentkezés
POST	/logout	Hitelesített	200, 401	Kijelentkezés
GET	/users/me	Hitelesített	200, 401	Saját profil, statisztikák
PUT	/users/me	Hitelesített	200, 422, 401	Profil módosítása
GET	/tickets	Hitelesített	200, 401	Jegyek listázása
POST	/tickets	Hitelesített	201, 422, 401	Jegy létrehozása
GET	/tickets/:id	Hitelesített	200, 404, 403, 401	Jegy részletek
PATCH	/tickets/:id	Hitelesített	200, 403, 422, 401	Jegy módosítása
DELETE	/tickets/:id	Hitelesített	200, 404, 403, 401	Jegy törlése
GET	/tickets/:id/replies	Hitelesített	200, 404, 403, 401	Válaszok listája
POST	/tickets/:id/replies	Hitelesített	201, 422, 403, 401	Válasz hozzáadása
Sources:

Adatbázis terv és sémák
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
| deleted_at (soft?) |       +--------------------+
+--------------------+
Migrációk
users

id, name, email(unique), password, role(enum: user|admin), softDeletes(), timestamps()

tickets

id, user_id(foreign, cascade), subject(string), description(text), priority(enum: low|medium|high), status(enum: open|pending|resolved|closed), timestamps()

ticket_replies

id, ticket_id(foreign, cascade), user_id(foreign, cascade), message(text), created_at(timestamp, useCurrent)

Telepítés és konfiguráció
Projekt létrehozása:

Parancs: composer create-project laravel/laravel --prefer-dist supportTicketBearer

.env beállítás:

DB_CONNECTION: mysql

DB_HOST: 127.0.0.1

DB_PORT: 3306

DB_DATABASE: support_ticket

DB_USERNAME: root

DB_PASSWORD: (megfelelő érték)

Időzóna:

config/app.php: 'timezone' => 'Europe/Budapest'

Sanctum telepítése és API inicializálás:

Parancsok:

composer require laravel/sanctum

php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

php artisan install:api

Tesztútvonal:

routes/api.php: GET /ping -> { "message": "API works!" }

Futtatás:

Parancs: php artisan serve

POSTMAN teszt: GET http://127.0.0.1:8000/api/ping

Modellek, kontrollerek és útvonalak
Modellek
User

Fillable: name, email, password, role

Hidden: password, remember_token

Relációk: hasMany(Ticket), hasMany(TicketReply)

Segéd: isAdmin(): bool

Ticket

Fillable: user_id, subject, description, priority, status

Relációk: belongsTo(User), hasMany(TicketReply)

TicketReply

Timestamps: csak created_at (updated_at nem szükséges)

Fillable: ticket_id, user_id, message

Relációk: belongsTo(Ticket), belongsTo(User)

AuthController
register(Request): validáció, user létrehozás, 201 válasz

login(Request): email+jelszó ellenőrzés, token létrehozás, 200 válasz

logout(Request): user->tokens()->delete(), 200 válasz

UserController
me(Request): user és jegy statisztika (opened/closed)

updateMe(Request): name/email unique kivétellel, password+confirmation, 200 válasz

TicketController
index(Request):

User: saját jegyek szűrése

Admin: minden jegy, opcionális query szűrés

store(Request): subject/description/priority validáció, 201 válasz

show(Request, $id): hozzáférés ellenőrzés (saját vagy admin), válaszok betöltése

update(Request, $id):

User: subject, description módosítása, ha status != closed

Admin: status, priority módosítása

destroy(Request, $id):

User: saját jegy törlése, ha status != closed

Admin: soft delete bármely jegy

TicketReplyController
index(Request, $ticketId): jegyhez tartozó válaszok listázása (hozzáférés ellenőrzés)

store(Request, $ticketId): message validáció, 201 válasz

Útvonalak (routes/api.php)
Public:

GET: /ping

POST: /register

POST: /login

Authenticated (auth:sanctum):

POST: /logout

GET: /users/me

PUT: /users/me

GET: /tickets

POST: /tickets

GET: /tickets/{id}

PATCH: /tickets/{id}

DELETE: /tickets/{id}

GET: /tickets/{id}/replies

POST: /tickets/{id}/replies

Seeding és tesztelés
Seeding
UserSeeder:

1 admin: admin@example.com, jelszó: admin, role: admin

9 user: Faker hu_HU nevekkel, jelszó: Jelszo123, role: user

TicketSeeder:

3 releváns jegy: különböző subject/priority/status kombinációkkal, felhasználókhoz rendelve.

TicketReplySeeder:

Válaszok: minden jegyhez 1–2 reply, vegyesen admin és user által.

DatabaseSeeder:

call: UserSeeder, TicketSeeder, TicketReplySeeder

Feature tesztek (példák)
AuthTest:

ping returns ok: GET /api/ping -> 200, "API works!"

register creates user: POST /api/register -> 201, user létrejött

login success/failure: POST /api/login -> 200 tokennel / 401 invalid

UserTest:

/users/me requires auth: 401 "Unauthenticated."

/users/me returns profile: 200, helyes struktúra és email

update name/email/password: 200, DB állapot ellenőrzése

TicketTest:

index requires auth: 401 "Unauthenticated."

index returns list: 200, struktúra és elemszám

store creates ticket: 201, DB has ticket

show returns details and replies: 200, nested struktúra

update rules: 200 admin módosítások; 403 user closed esetben; 422 invalid

destroy rules: 200 törlés; 403 tiltott; 404 nem található

TicketReplyTest:

index requires auth: 401

store requires message: 422 ha hiányzik; 201 ha ok

access control: user csak saját jegyhez fér; admin bármihez

Futtatás:

Parancs: php artisan tes
