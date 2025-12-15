# Ügyfélszolgálati jegykezelő REST API

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

### Kapcsolatok

- User → Ticket (1:N)
- Ticket → TicketReply (1:N)
- User → TicketReply (1:N)

---

### Authentikáció
```bash
Laravel Sanctum biztosítja a token alapú authentikációt.

Regisztráció: új user, role=user.

Login: email+jelszó, token visszaadás.

Jogosultságok: user saját jegyek, admin minden jegy.
```
### Postman
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
###6. Hibakódok

400 Bad Request – hibás formátum

401 Unauthorized – token hiba

403 Forbidden – jogosultság hiba

404 Not Found – erőforrás nem található

422 Unprocessable Entity – validációs hiba

7. Seeding
UserSeeder
```bash
php
User::create([
  'name' => 'Admin',
  'email' => 'admin@example.com',
  'password' => Hash::make('Admin123'),
  'role' => 'admin',
]);
User::factory(10)->create();
```
TicketSeeder
```bash
php
Ticket::create([
  'user_id' => $user->id,
  'subject' => 'Teszt jegy',
  'description' => 'Leírás',
  'priority' => 'medium',
  'status' => 'open',
]);
```
TicketReplySeeder
```bash
php
TicketReply::create([
  'ticket_id' => $ticket->id,
  'user_id' => $admin->id,
  'message' => 'Admin válasz',
]);
```
DatabaseSeeder
```bash
php
$this->call([
  UserSeeder::class,
  TicketSeeder::class,
  TicketReplySeeder::class,
]);
```
Futtatás:

```bash
php artisan migrate:fresh --seed
```
