# Ügyfélszolgálati jegykezelő rendszer 

## 1. Áttekintés

Az ügyfélszolgálati jegykezelő rendszer egy Laravel alapú REST API, amely lehetővé teszi
felhasználók számára hibajegyek (ticketek) létrehozását, nyomon követését és
megválaszolását.

A backend elsődleges célja egy frontend alkalmazás kiszolgálása, amelyet az ügyfelek,
ügyfélszolgálati munkatársak és adminisztrátorok használnak.

A rendszer role-alapú működést alkalmaz:
- `user` – ügyfél
- `admin` – adminisztrátor

---

## 2. Alapadatok

- **Base URL**
  - `http://127.0.0.1:8000/api` vagy `http://127.0.0.1/ugyfelszolgalat/public/api`

- **Adatbázis neve:** `ugyfelszolgalat`

---

## 3. Funkciók

### 3.1 Autentikáció
- Regisztráció
- Bejelentkezés
- Token alapú autentikáció (Laravel Sanctum)
- Kijelentkezés (token törlés)

### 3.2 Jegykezelés
- Jegy létrehozása
- Saját jegyek listázása
- Összes jegy listázása (agent/admin)
- Jegy státuszának módosítása
- Jegy prioritásának módosítása

### 3.3 Válaszkezelés
- Válasz írása egy jegyhez
- Jegyhez tartozó válaszok megjelenítése

---

## 4. Hibakezelés

| HTTP kód | Jelentés |
|--------|---------|
| 400 Bad Request | Hibás kérés vagy hiányzó mezők |
| 401 Unauthorized | Érvénytelen vagy hiányzó token |
| 403 Forbidden | Jogosultság hiánya |
| 404 Not Found | Erőforrás nem található |
| 503 Service Unavailable | Szerverhiba |

### 401 Unauthorized – példa
```json
{
  "message": "Invalid token"
}
```
### 5. Nem védett végpontok
**GET** `/ping`
API elérhetőségének ellenőrzése.

Válasz: 200 OK

**POST** `/register`

Új felhasználó regisztrálása.
Kérés törzse:
```json
{
  "name": "Teszt User",
  "email": "user@example.com",
  "password": "Jelszo_2025",
  "password_confirmation": "Jelszo_2025"
}
```
Válasz: 201 Created

```json
{
  "message": "User created successfully",
  "user": {
    "id": 5,
    "name": "Teszt User",
    "email": "user@example.com",
    "role": "user"
  }
}
```
**POST** `/login`
Felhasználó bejelentkezése.

Válasz: 200 OK

```json
{
  "message": "Login successful",
  "user": {
    "id": 5,
    "name": "Teszt User",
    "email": "user@example.com",
    "role": "user"
  },
  "access": {
    "token": "1|abcdef123456",
    "token_type": "Bearer"
  }
}
```
6. Authentikált végpontok
A következő végpontokhoz kötelező az Authorization header:

makefile
```
Authorization: Bearer <token>
POST /logout
Kijelentkezés, token törlése.
```
Válasz: 200 OK

```json
{
  "message": "Logout successful"
}
```
### 7. Jegykezelés
**POST** `/tickets`
Új jegy létrehozása.
Kérés törzse:

```json
{
  "subject": "Nem működik a bejelentkezés",
  "description": "A rendszer hibát dob.",
  "priority": "high"
}
```
Válasz: 201 Created

```json
{
  "message": "Ticket created successfully"
}
```
**GET** `/tickets`
Jegyek listázása:
- user: csak saját jegyek
- admin: minden jegy

Válasz: 200 OK

json
```
{
  "tickets": [
    {
      "id": 1,
      "subject": "Nem működik a bejelentkezés",
      "priority": "high",
      "status": "open",
      "created_at": "2025-03-10 10:00:00"
    }
  ]
}
```
**GET** `/tickets/{id}`
Egy jegy részleteinek lekérése válaszokkal együtt.

Válasz: 200 OK

```json
{
  "ticket": {
    "id": 1,
    "subject": "Nem működik a bejelentkezés",
    "description": "A rendszer hibát dob",
    "priority": "high",
    "status": "open"
  },
  "replies": [
    {
      "user": "Admin",
      "message": "Dolgozunk a megoldáson",
      "created_at": "2025-03-10 11:00:00"
    }
  ]
}
```
**PATCH** `/tickets/{id}`
Jegy státuszának és prioritásának módosítása (agent/admin).
Kérés törzse:

```json
{
  "status": "in_progress",
  "priority": "medium"
}
```
### 8. Válaszkezelés
**POST** `/tickets/{id}/replies`
Válasz hozzáadása egy jegyhez.
Kérés törzse:

```json
{
  "message": "A hibát javítottuk, kérjük ellenőrizze."
}
```
Válasz: 201 Created

```json
{
  "message": "Reply added successfully"
}
```
### 9. Adatbázis terv

```
+---------------------------+      +---------------------+      +------------------------+
|   personal_access_tokens  |      |        users        |      |        tickets         |
+---------------------------+      +---------------------+      +------------------------+
| id (PK)                   |   _1 | id (PK)             | 1__  | id (PK)                |
| tokenable_id (FK)         | K_/  | name                |     \| user_id (FK)           |
| tokenable_type            |      | email (unique)      |      | subject                |
| name                      |      | password            |      | description            |
| token (unique)            |      | role (admin/user)   |      | priority               |
| abilities                 |      | updated_at          |      | status                 |
| last_used_at              |      | created_at          |      | created_at             |
| created_at                |      +---------------------+      | updated_at             |
| updated_at                |                                   +------------------------+
+---------------------------+                                          |
                                                                       | 1
                                                                       |
                                                                       | N
                                                               +------------------------+
                                                               |     ticket_replies     |
                                                               +------------------------+
                                                               | id (PK)                |
                                                               | ticket_id (FK)         |
                                                               | user_id (FK)           |
                                                               | message                |
                                                               | created_at             |
                                                               +------------------------+
                                                                

```
### 10. Összefoglaló végponttábla

|HTTP metódus|	Útvonal	             |Jogosultság	| Rövid leírás                                 |
|------------|-----------------------|--------------|----------------------------------------------|
|GET	     | /ping	             | Nyilvános	| API teszteléshez                             |
|POST	     | /register	         | Nyilvános	| Regisztráció                                 |
|POST	     | /login	             | Nyilvános	| Bejelentkezés e-maillel és jelszóval         |
|POST	     | /logout	             | Hitelesített | Kijelentkezés                                |
|POST	     | /tickets  	         | Hitelesített | Jegy létrehozása                             |
|GET         | /tickets 	         | Hitelesített | Jegyek listázása                             |
|GET	     | /tickets/{id}         | Hitelesített | Jegy részletei                               |
|PATCH	     | /tickets/{id          | Admin	    | Jegy frissítése                              |
|POST	     | /tickets/{id}/replies | Hitelesített	| Válasz küldése                               |

# I. Modul – Telepítés és alapkonfiguráció
---
### 1. Projekt létrehozása
---
```
composer create-project laravel/laravel --prefer-dist ugyfelszolgalat
cd ugyfelszolgalat
```
### 2. .env konfiguráció
---
```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=ugyfelszolgalat
DB_USERNAME=root
DB_PASSWORD=
```
### 3. Időzóna beállítása
---
config/app.php:

`'timezone' => 'Europe/Budapest',`

### 4. Sanctum telepítése
---
```
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan install:api
```
### 5. Teszt végpont
---
routes/api.php:
```
Route::get('/ping', function () {
    return response()->json(['message' => 'API works'], 200);
});
```

Szerver indítása:

`php artisan serve`


Teszt (Postman / böngésző):
`GET http://127.0.0.1:8000/api/ping`

# II. Modul – Modellek és migrációk
---
### 1. Users tábla módosítása
---
```bash
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('password');
    $table->enum('role', ['user', 'agent', 'admin']);
    $table->timestamps();
});
```
### 2. User modell
---
```bash
class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    protected $fillable = ['name', 'email', 'password', 'role'];

    protected $hidden = ['password', 'remember_token'];

    public function tickets()
    {
        return $this->hasMany(Ticket::class);
    }

    public function replies()
    {
        return $this->hasMany(TicketReply::class);
    }
}
```
### 3. Ticket modell és migráció
---
`php artisan make:model Ticket -m`
```bash
Schema::create('tickets', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('subject');
    $table->text('description');
    $table->enum('priority', ['low', 'medium', 'high']);
    $table->enum('status', ['open', 'in_progress', 'closed'])->default('open');
    $table->timestamps();
});
```
```bash
class Ticket extends Model
{
    protected $fillable = ['subject', 'description', 'priority', 'status'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function replies()
    {
        return $this->hasMany(TicketReply::class);
    }
}
```
### 4. TicketReply modell és migráció
---
`php artisan make:model TicketReply -m`
```bash
Schema::create('ticket_replies', function (Blueprint $table) {
    $table->id();
    $table->foreignId('ticket_id')->constrained()->cascadeOnDelete();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->text('message');
    $table->timestamps();
});
```
```bash
class TicketReply extends Model
{
    protected $fillable = ['ticket_id', 'user_id', 'message'];

    public function ticket()
    {
        return $this->belongsTo(Ticket::class);
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

Migrációk futtatása:

`php artisan migrate`

# III. Modul – Seedelés (tesztadatok)
---
### 1. UserSeeder
---
`php artisan make:seeder UserSeeder`
```bash
User::create([
    'name' => 'Admin',
    'email' => 'admin@example.com',
    'password' => Hash::make('admin'),
    'role' => 'admin',
]);
```
### 2. TicketSeeder
---
`php artisan make:seeder TicketSeeder`
```bash
Ticket::create([
    'user_id' => 1,
    'subject' => 'Nem működik a bejelentkezés',
    'description' => 'Hibát kapok belépéskor',
    'priority' => 'high',
]);
```

DatabaseSeeder.php:
```bash
$this->call([
    UserSeeder::class,
    TicketSeeder::class,
]);
```
`php artisan db:seed`

# IV. Modul – Controllerek és végpontok
---
### 1. AuthController
---
`php artisan make:controller AuthController`


Funkciók:
- register
- login
- logout

### 2. TicketController
---
`php artisan make:controller TicketController`


Funkciók:
- store() – jegy létrehozás
- index() – jegyek listázása (role alapján)
- show() – jegy részletek
- update() – státusz/prioritás módosítás (agent/admin)

### 3. TicketReplyController
---
`php artisan make:controller TicketReplyController`

Funkció:
store() – válasz hozzáadása jegyhez

# V. Modul – Route-ok
---
routes/api.php:
```bash
Route::get('/ping', fn () => response()->json(['message'=>'API works']));

Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);

    Route::post('/tickets', [TicketController::class, 'store']);
    Route::get('/tickets', [TicketController::class, 'index']);
    Route::get('/tickets/{id}', [TicketController::class, 'show']);
    Route::patch('/tickets/{id}', [TicketController::class, 'update']);

    Route::post('/tickets/{id}/replies', [TicketReplyController::class, 'store']);
});
```
# VI. Modul – Tesztelés
---
`php artisan test`
