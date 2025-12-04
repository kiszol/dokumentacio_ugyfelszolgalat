# Ügyfélszolgálati jegykezelő REST API

## 1. Cél és áttekintés

A **20feladatBearer** egy egyszerű, Laravel alapú REST API projekt, amely ügyfélszolgálati jegyek kezelésére szolgál.  
A rendszerben:

- felhasználók (**users**) hozhatnak létre jegyeket (**tickets**),
- a jegyekhez válaszok (**ticket_replies**) fűzhetők,
- a hozzáférés **Bearer token** alapú autentikációval történik.

A dokumentáció célja, hogy bemutassa:

- az adatbázis struktúrát,
- a főbb Laravel komponenseket (modellek, kontrollerek, route-ok),
- az elérhető REST végpontokat,
- példákat a kérésekre és válaszokra.

---

## 2. Használt technológiák

- **PHP** >= 8.1  
- **Laravel** 10+  
- **MySQL / MariaDB** (vagy bármely támogatott SQL adatbázis)  
- **Composer** (függőségkezelő)  
- **Laravel Sanctum / Passport / saját Bearer guard** a token alapú autentikációhoz  
- Fejlesztéshez: **Postman / Insomnia** vagy más REST kliens

> Megjegyzés: a példák Laravel-alapúak, de a REST API elvek más backend környezetben is ugyanígy alkalmazhatók.

---

## 3. Adatbázis séma

A feladatban megadott táblák:

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('customer', 'agent', 'admin') DEFAULT 'customer',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);

CREATE TABLE tickets (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    subject VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    priority ENUM('low', 'medium', 'high') DEFAULT 'medium',
    status ENUM('open', 'in_progress', 'resolved', 'closed') DEFAULT 'open',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_tickets_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);

CREATE TABLE ticket_replies (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    ticket_id BIGINT UNSIGNED NOT NULL,
    user_id BIGINT UNSIGNED NOT NULL,
    message TEXT NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_replies_ticket FOREIGN KEY (ticket_id)
        REFERENCES tickets(id) ON DELETE CASCADE,
    CONSTRAINT fk_replies_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE CASCADE
);
```

### 3.1 Laravel migrációk

**`database/migrations/xxxx_xx_xx_create_users_table.php`** – csak a lényeges mezők:

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('password');
    $table->enum('role', ['customer', 'agent', 'admin'])->default('customer');
    $table->timestamps();
});
```

**`create_tickets_table.php`**:

```php
Schema::create('tickets', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->string('subject');
    $table->text('description');
    $table->enum('priority', ['low', 'medium', 'high'])->default('medium');
    $table->enum('status', ['open', 'in_progress', 'resolved', 'closed'])->default('open');
    $table->timestamps();
});
```

**`create_ticket_replies_table.php`**:

```php
Schema::create('ticket_replies', function (Blueprint $table) {
    $table->id();
    $table->foreignId('ticket_id')->constrained()->onDelete('cascade');
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->text('message');
    $table->timestamps();
});
```

---

## 4. Modellek és kapcsolatok

### 4.1 `User` modell

**`app/Models/User.php`**

```php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;

    protected $fillable = [
        'name',
        'email',
        'password',
        'role',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'email_verified_at' => 'datetime',
    ];

    public function tickets()
    {
        return $this->hasMany(Ticket::class);
    }

    public function ticketReplies()
    {
        return $this->hasMany(TicketReply::class);
    }
}
```

### 4.2 `Ticket` modell

**`app/Models/Ticket.php`**

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Ticket extends Model
{
    protected $fillable = [
        'user_id',
        'subject',
        'description',
        'priority',
        'status',
    ];

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

### 4.3 `TicketReply` modell

**`app/Models/TicketReply.php`**

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class TicketReply extends Model
{
    protected $fillable = [
        'ticket_id',
        'user_id',
        'message',
    ];

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

---

## 5. Autentikáció (Bearer token)

A rendszer **Bearer token** alapú autentikációt használ (pl. Laravel Sanctum).  
A felhasználó bejelentkezés után kap egy tokent, amelyet a következő HTTP headerben kell küldeni:

```http
Authorization: Bearer {token}
```

### 5.1 Auth route-ok

**`routes/api.php`** (részlet):

```php
use App\Http\Controllers\AuthController;
use App\Http\Controllers\TicketController;
use App\Http\Controllers\TicketReplyController;
use Illuminate\Support\Facades\Route;

Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);

    Route::get('/tickets', [TicketController::class, 'index']);
    Route::post('/tickets', [TicketController::class, 'store']);
    Route::get('/tickets/{ticket}', [TicketController::class, 'show']);
    Route::put('/tickets/{ticket}', [TicketController::class, 'update']);
    Route::delete('/tickets/{ticket}', [TicketController::class, 'destroy']);

    Route::get('/tickets/{ticket}/replies', [TicketReplyController::class, 'index']);
    Route::post('/tickets/{ticket}/replies', [TicketReplyController::class, 'store']);
});
```

### 5.2 `AuthController` (részlet)

**`app/Http/Controllers/AuthController.php`**

```php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function register(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users,email',
            'password' => 'required|min:6|confirmed',
        ]);

        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);

        return response()->json($user, 201);
    }

    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $user = User::where('email', $credentials['email'])->first();

        if (! $user || ! Hash::check($credentials['password'], $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'token' => $token,
            'token_type' => 'Bearer',
            'user' => $user,
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logged out']);
    }
}
```

---

## 6. Ticket API végpontok

### 6.1 Összefoglaló táblázat

| HTTP metódus | URL                           | Leírás                                  | Auth |
|--------------|-------------------------------|-----------------------------------------|------|
| POST         | `/api/register`               | Regisztráció                            | Nem  |
| POST         | `/api/login`                  | Bejelentkezés, token generálás         | Nem  |
| POST         | `/api/logout`                 | Kijelentkezés, token törlés            | Igen |
| GET          | `/api/tickets`                | Jegyek listázása (szűrve is)           | Igen |
| POST         | `/api/tickets`                | Új jegy létrehozása                    | Igen |
| GET          | `/api/tickets/{id}`           | Konkrét jegy lekérése                  | Igen |
| PUT/PATCH    | `/api/tickets/{id}`           | Jegy módosítása                         | Igen |
| DELETE       | `/api/tickets/{id}`           | Jegy törlése                            | Igen |
| GET          | `/api/tickets/{id}/replies`   | Jegy válaszainak listázása             | Igen |
| POST         | `/api/tickets/{id}/replies`   | Új válasz hozzáadása                   | Igen |

---

### 6.2 Jegyek listázása

**Kérés:**

```http
GET /api/tickets?status=open&priority=high&page=1 HTTP/1.1
Host: example.test
Authorization: Bearer {token}
Accept: application/json
```

Támogatott query paraméterek:

- `status` – `open`, `in_progress`, `resolved`, `closed`
- `priority` – `low`, `medium`, `high`
- `user_id` – adott felhasználó jegyei
- `page` – lapozás

**Példa válasz:**

```json
{
  "data": [
    {
      "id": 1,
      "user_id": 3,
      "subject": "Nem működik a belépés",
      "description": "Belépésnél hibát kapok...",
      "priority": "high",
      "status": "open",
      "created_at": "2025-05-01T10:15:00.000000Z",
      "updated_at": "2025-05-01T10:15:00.000000Z"
    }
  ],
  "links": {
    "first": "http://example.test/api/tickets?page=1",
    "last": "http://example.test/api/tickets?page=1",
    "prev": null,
    "next": null
  },
  "meta": {
    "current_page": 1,
    "from": 1,
    "last_page": 1,
    "path": "http://example.test/api/tickets",
    "per_page": 15,
    "to": 1,
    "total": 1
  }
}
```

### 6.3 Jegy létrehozása

**Kérés:**

```http
POST /api/tickets HTTP/1.1
Host: example.test
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json

{
  "subject": "Nem működik a belépés",
  "description": "Belépéskor 500-as hibakódot kapok.",
  "priority": "high"
}
```

**Válasz (201 Created):**

```json
{
  "id": 1,
  "user_id": 3,
  "subject": "Nem működik a belépés",
  "description": "Belépéskor 500-as hibakódot kapok.",
  "priority": "high",
  "status": "open",
  "created_at": "2025-05-01T10:15:00.000000Z",
  "updated_at": "2025-05-01T10:15:00.000000Z"
}
```

### 6.4 `TicketController` – főbb metódusok

**`app/Http/Controllers/TicketController.php`** (rövidítve):

```php
namespace App\Http\Controllers;

use App\Models\Ticket;
use Illuminate\Http\Request;

class TicketController extends Controller
{
    public function index(Request $request)
    {
        $query = Ticket::query()->with('user');

        if ($request->filled('status')) {
            $query->where('status', $request->status);
        }

        if ($request->filled('priority')) {
            $query->where('priority', $request->priority);
        }

        if ($request->filled('user_id')) {
            $query->where('user_id', $request->user_id);
        }

        return $query->paginate(15);
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'subject' => 'required|string|max:255',
            'description' => 'required|string',
            'priority' => 'nullable|in:low,medium,high',
        ]);

        $validated['user_id'] = $request->user()->id;

        $ticket = Ticket::create($validated);

        return response()->json($ticket, 201);
    }

    public function show(Ticket $ticket)
    {
        $ticket->load(['user', 'replies.user']);

        return $ticket;
    }

    public function update(Request $request, Ticket $ticket)
    {
        $this->authorize('update', $ticket);

        $validated = $request->validate([
            'subject' => 'sometimes|string|max:255',
            'description' => 'sometimes|string',
            'priority' => 'sometimes|in:low,medium,high',
            'status' => 'sometimes|in:open,in_progress,resolved,closed',
        ]);

        $ticket->update($validated);

        return $ticket;
    }

    public function destroy(Ticket $ticket)
    {
        $this->authorize('delete', $ticket);

        $ticket->delete();

        return response()->json(null, 204);
    }
}
```

---

## 7. Válaszok kezelése (`TicketReplyController`)

### 7.1 Válaszok listázása

```http
GET /api/tickets/1/replies HTTP/1.1
Host: example.test
Authorization: Bearer {token}
Accept: application/json
```

**Válasz:**

```json
[
  {
    "id": 1,
    "ticket_id": 1,
    "user_id": 2,
    "message": "Megvizsgáljuk a hibát, türelmét kérjük.",
    "created_at": "2025-05-01T11:00:00.000000Z",
    "updated_at": "2025-05-01T11:00:00.000000Z"
  }
]
```

### 7.2 Új válasz létrehozása

```http
POST /api/tickets/1/replies HTTP/1.1
Host: example.test
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json

{
  "message": "A hiba javítva lett, próbálja meg újra a belépést."
}
```

**`app/Http/Controllers/TicketReplyController.php`** (részlet):

```php
namespace App\Http\Controllers;

use App\Models\Ticket;
use App\Models\TicketReply;
use Illuminate\Http\Request;

class TicketReplyController extends Controller
{
    public function index(Ticket $ticket)
    {
        return $ticket->replies()->with('user')->get();
    }

    public function store(Request $request, Ticket $ticket)
    {
        $validated = $request->validate([
            'message' => 'required|string',
        ]);

        $reply = TicketReply::create([
            'ticket_id' => $ticket->id,
            'user_id' => $request->user()->id,
            'message' => $validated['message'],
        ]);

        return response()->json($reply, 201);
    }
}
```

---

## 8. Jogosultságkezelés (Policy példa)

A jegyek módosítása / törlése csak:
- az adott jegy tulajdonosa, vagy
- **admin / agent** szerepkörű felhasználó számára engedélyezett.

**`app/Policies/TicketPolicy.php`** (részlet):

```php
namespace App\Policies;

use App\Models\Ticket;
use App\Models\User;

class TicketPolicy
{
    public function update(User $user, Ticket $ticket)
    {
        return $user->id === $ticket->user_id
            || in_array($user->role, ['admin', 'agent']);
    }

    public function delete(User $user, Ticket $ticket)
    {
        return $user->id === $ticket->user_id
            || in_array($user->role, ['admin', 'agent']);
    }
}
```

**`AuthServiceProvider.php`** regisztráció:

```php
protected $policies = [
    \App\Models\Ticket::class => \App\Policies\TicketPolicy::class,
];
```

---

## 9. Hibakezelés és validáció

- Laravel beépített validációja használatban (`$request->validate([...])`).
- Hibás kérés esetén (422) a válasz például:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "subject": [
      "The subject field is required."
    ]
  }
}
```

- Jogosultság hiánya esetén (403):

```json
{
  "message": "This action is unauthorized."
}
```

- Nem létező erőforrás esetén (404):

```json
{
  "message": "No query results for model [App\\Models\\Ticket] 999"
}
```

---

## 10. Telepítés és futtatás (lépések)

1. **Repository klónozása**
   ```bash
   git clone https://github.com/felhasznalo/20feladatBearer.git
   cd 20feladatBearer
   ```

2. **Függőségek telepítése**
   ```bash
   composer install
   npm install   # ha szükséges a front-endhez
   ```

3. **.env fájl létrehozása**
   ```bash
   cp .env.example .env
   ```

   - Adatbázis beállítások (DB_DATABASE, DB_USERNAME, DB_PASSWORD)
   - APP_URL, APP_NAME stb.

4. **Application key generálása**
   ```bash
   php artisan key:generate
   ```

5. **Migrációk futtatása és (opcionális) seederek**
   ```bash
   php artisan migrate --seed
   ```

6. **Fejlesztői szerver indítása**
   ```bash
   php artisan serve
   ```

7. **API tesztelése**
   - Postman / Insomnia segítségével
   - vagy `curl` parancsokkal

---

## 11. Összefoglalás

A **20feladatBearer** projekt:

- bemutatja egy tipikus ügyfélszolgálati jegykezelő rendszer REST API megvalósítását,
- Laravel alapú, Bearer tokenes autentikációval,
- tiszta adatmodellt használ (`users`, `tickets`, `ticket_replies`),
- jól elkülönített felelősségeket valósít meg (AuthController, TicketController, TicketReplyController, Policy-k),
- alkalmas további fejlesztésre (pl. fájlcsatolmányok, kategóriák, SLA kezelése, értesítések stb.).

A fenti dokumentáció önmagában is elég ahhoz, hogy egy másik fejlesztő a kód megnyitása előtt is átlássa a rendszer működését és az API hívások menetét.
