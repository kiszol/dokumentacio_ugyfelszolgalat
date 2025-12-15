Ügyfélszolgálati jegykezelő REST API – Fejlesztői kézikönyv
1. Bevezetés
Az ügyfélszolgálati jegykezelő rendszer célja, hogy strukturált módon kezelje a felhasználói problémákat és kérdéseket. A REST API lehetővé teszi jegyek létrehozását, listázását, módosítását és törlését, valamint válaszok kezelését. A rendszer Laravel + Sanctum alapokon épül.

2. Architektúra áttekintés
Felhasználók (users): jegyek tulajdonosai és válaszadók. Szerepkörök: user, admin.

Jegyek (tickets): problémák, kérdések. Attribútumok: subject, description, priority, status.

Jegyválaszok (ticket_replies): kommunikációs szálak.

3. Adatbázis séma
text
users(id, name, email, role, password, created_at, updated_at)
tickets(id, user_id, subject, description, priority, status, created_at, updated_at)
ticket_replies(id, ticket_id, user_id, message, created_at)
Relációk:

User–Ticket 1:N

Ticket–Reply 1:N

User–Reply 1:N

4. Authentikáció
Laravel Sanctum biztosítja a token alapú authentikációt.

Regisztráció: új user, role=user.

Login: email+jelszó, token visszaadás.

Jogosultságok: user saját jegyek, admin minden jegy.

5. Végpontok részletes leírása
Auth
POST /register

json
{
  "name": "Teszt Elek",
  "email": "teszt@example.com",
  "password": "Jelszo_2025",
  "password_confirmation": "Jelszo_2025"
}
Sikeres válasz (201):

json
{ "message":"User created successfully", "user":{...} }
POST /login

json
{
  "email": "teszt@example.com",
  "password": "Jelszo_2025"
}
Sikeres válasz (200):

json
{ "message":"Login successful", "user":{...}, "access":{"token":"..."} }
POST /logout

json
{ "message":"Logout successful" }
User
GET /users/me → Saját profil + statisztika.

PUT /users/me → Profil frissítése.

Tickets
GET /tickets → Saját jegyek (user), minden jegy (admin).

POST /tickets

json
{
  "subject": "Nem működik a bejelentkezés",
  "description": "A rendszer hibát dob.",
  "priority": "high"
}
GET /tickets/:id → Jegy részletei + válaszok.

PATCH /tickets/:id → User subject/description, admin status/priority.

DELETE /tickets/:id → Jegy törlése.

Ticket replies
GET /tickets/:id/replies → Válaszok listázása.

POST /tickets/:id/replies

json
{
  "message": "Most már működik, köszönöm!"
}
6. Hibakódok
400 Bad Request – hibás formátum

401 Unauthorized – token hiba

403 Forbidden – jogosultság hiba

404 Not Found – erőforrás nem található

422 Unprocessable Entity – validációs hiba

7. Seeding
UserSeeder
php
User::create([
  'name' => 'Admin',
  'email' => 'admin@example.com',
  'password' => Hash::make('Admin123'),
  'role' => 'admin',
]);
User::factory(10)->create();
TicketSeeder
php
Ticket::create([
  'user_id' => $user->id,
  'subject' => 'Teszt jegy',
  'description' => 'Leírás',
  'priority' => 'medium',
  'status' => 'open',
]);
TicketReplySeeder
php
TicketReply::create([
  'ticket_id' => $ticket->id,
  'user_id' => $admin->id,
  'message' => 'Admin válasz',
]);
DatabaseSeeder
php
$this->call([
  UserSeeder::class,
  TicketSeeder::class,
  TicketReplySeeder::class,
]);
Futtatás:

bash
php artisan migrate:fresh --seed
8. Tesztelés
AuthTest
php
$response = $this->postJson('/api/register', [
  'name' => 'Teszt Elek',
  'email' => 'teszt@example.com',
  'password' => 'Jelszo_2025',
  'password_confirmation' => 'Jelszo_2025',
]);
$response->assertStatus(201)->assertJsonStructure(['message','user']);
TicketTest
php
$response = $this->postJson('/api/tickets', [
  'subject' => 'Teszt jegy',
  'description' => 'Leírás',
  'priority' => 'high',
], ['Authorization' => 'Bearer '.$token]);

$response->assertStatus(201)->assertJsonStructure(['message','ticket']);
TicketReplyTest
php
$response = $this->postJson("/api/tickets/{$ticket->id}/replies", [
  'message' => 'Ez egy válasz.',
], ['Authorization' => 'Bearer '.$token]);

$response->assertStatus(201)->assertJsonStructure(['message','reply']);
Futtatás:

bash
php artisan test
