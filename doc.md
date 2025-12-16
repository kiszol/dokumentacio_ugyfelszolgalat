# 20feladatBearer – Ügyfélszolgálati jegykezelő rendszer (REST API, JWT Bearer)

Ez a dokumentum egy **JWT Bearer**-rel védett, egyszerű **ügyfélszolgálati jegykezelő (ticketing)** REST API projekt létrehozását és használatát írja le.

**Téma / adatmodell:**
- `users(id, name, email, role, created_at, updated_at)`
- `tickets(id, user_id, subject, description, priority, status, created_at, updated_at)`
- `ticket_replies(id, ticket_id, user_id, message, created_at)`

---

## I. Előkészítés

### 1) Követelmények
- .NET SDK (ajánlott: .NET 8 LTS)
- Entity Framework Core CLI (ha nincs):  
  ```bash
  dotnet tool install --global dotnet-ef
  ```

### 2) Projekt létrehozása
```bash
dotnet new webapi -n TicketingBearerApi
cd TicketingBearerApi
```

### 3) NuGet csomagok
SQLite + EF Core + JWT Bearer:
```bash
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.IdentityModel.Tokens
```

### 4) Könyvtárstruktúra (ajánlott)
```
TicketingBearerApi/
  Controllers/
  Data/
  Dtos/
  Models/
  Services/
  Program.cs
  appsettings.json
```

### 5) Konfiguráció (appsettings.json)
Hozz létre (vagy bővíts) a következőkkel:

```json
{
  "ConnectionStrings": {
    "Default": "Data Source=ticketing.db"
  },
  "Jwt": {
    "Issuer": "TicketingBearerApi",
    "Audience": "TicketingBearerApiClient",
    "Key": "CHANGE_THIS_TO_A_LONG_RANDOM_SECRET_KEY_32+",
    "ExpiresMinutes": 60
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

**Fontos:** a `Jwt:Key` minimum 32+ karakter hosszú, véletlen titok legyen.

---

## II. Controllerek és Végpontok

### 1) Modellek

#### Models/User.cs
```csharp
namespace TicketingBearerApi.Models;

public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public string Role { get; set; } = "customer"; // admin | agent | customer

    // Egyszerűsített auth: jelszó hash tárolása (demo célra)
    public string PasswordHash { get; set; } = "";

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;

    public List<Ticket> Tickets { get; set; } = new();
    public List<TicketReply> TicketReplies { get; set; } = new();
}
```

#### Models/Ticket.cs
```csharp
namespace TicketingBearerApi.Models;

public class Ticket
{
    public int Id { get; set; }
    public int UserId { get; set; }             // aki létrehozta
    public User? User { get; set; }

    public string Subject { get; set; } = "";
    public string Description { get; set; } = "";

    public string Priority { get; set; } = "medium"; // low | medium | high
    public string Status { get; set; } = "open";      // open | in_progress | resolved | closed

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;

    public List<TicketReply> Replies { get; set; } = new();
}
```

#### Models/TicketReply.cs
```csharp
namespace TicketingBearerApi.Models;

public class TicketReply
{
    public int Id { get; set; }

    public int TicketId { get; set; }
    public Ticket? Ticket { get; set; }

    public int UserId { get; set; }         // aki írta (agent vagy customer)
    public User? User { get; set; }

    public string Message { get; set; } = "";

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

---

### 2) Adatbázis réteg (EF Core)

#### Data/AppDbContext.cs
```csharp
using Microsoft.EntityFrameworkCore;
using TicketingBearerApi.Models;

namespace TicketingBearerApi.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) {}

    public DbSet<User> Users => Set<User>();
    public DbSet<Ticket> Tickets => Set<Ticket>();
    public DbSet<TicketReply> TicketReplies => Set<TicketReply>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>()
            .HasIndex(u => u.Email)
            .IsUnique();

        modelBuilder.Entity<Ticket>()
            .HasOne(t => t.User)
            .WithMany(u => u.Tickets)
            .HasForeignKey(t => t.UserId)
            .OnDelete(DeleteBehavior.Cascade);

        modelBuilder.Entity<TicketReply>()
            .HasOne(r => r.Ticket)
            .WithMany(t => t.Replies)
            .HasForeignKey(r => r.TicketId)
            .OnDelete(DeleteBehavior.Cascade);

        modelBuilder.Entity<TicketReply>()
            .HasOne(r => r.User)
            .WithMany(u => u.TicketReplies)
            .HasForeignKey(r => r.UserId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

---

### 3) DTO-k (ajánlott – tiszta API)

#### Dtos/AuthDtos.cs
```csharp
namespace TicketingBearerApi.Dtos;

public record RegisterRequest(string Name, string Email, string Password, string Role);
public record LoginRequest(string Email, string Password);
public record AuthResponse(string Token);
```

#### Dtos/TicketDtos.cs
```csharp
namespace TicketingBearerApi.Dtos;

public record TicketCreateRequest(string Subject, string Description, string Priority);
public record TicketUpdateStatusRequest(string Status);
public record TicketUpdatePriorityRequest(string Priority);

public record TicketReplyCreateRequest(string Message);
```

---

### 4) JWT token szolgáltatás

#### Services/JwtTokenService.cs
```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;
using TicketingBearerApi.Models;

namespace TicketingBearerApi.Services;

public class JwtOptions
{
    public string Issuer { get; set; } = "";
    public string Audience { get; set; } = "";
    public string Key { get; set; } = "";
    public int ExpiresMinutes { get; set; } = 60;
}

public class JwtTokenService
{
    private readonly JwtOptions _jwt;

    public JwtTokenService(IOptions<JwtOptions> jwtOptions)
    {
        _jwt = jwtOptions.Value;
    }

    public string CreateToken(User user)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new(ClaimTypes.Name, user.Email),
            new(ClaimTypes.Role, user.Role)
        };

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwt.Key));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwt.Issuer,
            audience: _jwt.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwt.ExpiresMinutes),
            signingCredentials: creds
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

### 5) Program.cs (Auth + EF + Swagger)

#### Program.cs (minta)
```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using TicketingBearerApi.Data;
using TicketingBearerApi.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// EF Core
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlite(builder.Configuration.GetConnectionString("Default"))
);

// JWT options
builder.Services.Configure<JwtOptions>(builder.Configuration.GetSection("Jwt"));
builder.Services.AddScoped<JwtTokenService>();

// AuthN/AuthZ
var jwtSection = builder.Configuration.GetSection("Jwt");
var jwtKey = jwtSection["Key"]!;

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwtSection["Issuer"],
            ValidAudience = jwtSection["Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtKey))
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

---

### 6) AuthController

#### Controllers/AuthController.cs
- `POST /api/auth/register` – felhasználó létrehozása
- `POST /api/auth/login` – token kérés

```csharp
using System.Security.Cryptography;
using System.Text;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using TicketingBearerApi.Data;
using TicketingBearerApi.Dtos;
using TicketingBearerApi.Models;
using TicketingBearerApi.Services;

namespace TicketingBearerApi.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly JwtTokenService _jwt;

    public AuthController(AppDbContext db, JwtTokenService jwt)
    {
        _db = db;
        _jwt = jwt;
    }

    [HttpPost("register")]
    public async Task<ActionResult> Register(RegisterRequest req)
    {
        var email = req.Email.Trim().ToLowerInvariant();

        if (await _db.Users.AnyAsync(u => u.Email == email))
            return Conflict(new { message = "Email már foglalt." });

        var user = new User
        {
            Name = req.Name.Trim(),
            Email = email,
            Role = string.IsNullOrWhiteSpace(req.Role) ? "customer" : req.Role.Trim().ToLowerInvariant(),
            PasswordHash = Hash(req.Password)
        };

        _db.Users.Add(user);
        await _db.SaveChangesAsync();

        return CreatedAtAction(nameof(Register), new { id = user.Id }, new { user.Id, user.Name, user.Email, user.Role });
    }

    [HttpPost("login")]
    public async Task<ActionResult<AuthResponse>> Login(LoginRequest req)
    {
        var email = req.Email.Trim().ToLowerInvariant();
        var user = await _db.Users.FirstOrDefaultAsync(u => u.Email == email);

        if (user == null || user.PasswordHash != Hash(req.Password))
            return Unauthorized(new { message = "Hibás email vagy jelszó." });

        var token = _jwt.CreateToken(user);
        return Ok(new AuthResponse(token));
    }

    private static string Hash(string input)
    {
        using var sha = SHA256.Create();
        var bytes = sha.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Convert.ToHexString(bytes);
    }
}
```

Megjegyzés: oktatási projekthez elfogadható, de éles rendszerben **ASP.NET Identity / BCrypt / Argon2** javasolt.

---

### 7) UsersController (admin-only)

#### Jogosultság
- Csak `admin`

#### Controllers/UsersController.cs (váz)
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using TicketingBearerApi.Data;

namespace TicketingBearerApi.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize(Roles = "admin")]
public class UsersController : ControllerBase
{
    private readonly AppDbContext _db;
    public UsersController(AppDbContext db) => _db = db;

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var users = await _db.Users
            .Select(u => new { u.Id, u.Name, u.Email, u.Role, u.CreatedAt, u.UpdatedAt })
            .ToListAsync();

        return Ok(users);
    }

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        var u = await _db.Users
            .Where(x => x.Id == id)
            .Select(x => new { x.Id, x.Name, x.Email, x.Role, x.CreatedAt, x.UpdatedAt })
            .FirstOrDefaultAsync();

        return u == null ? NotFound() : Ok(u);
    }
}
```

---

### 8) TicketsController (ticket CRUD + státusz/prioritás)

#### Jogosultsági szabály (javasolt)
- `customer`: csak a saját ticketjeit látja/módosíthatja.
- `agent` és `admin`: minden ticketet láthat, státuszt/prioritást állíthat.
- Ticket létrehozás: bármely bejelentkezett user.

#### Controllers/TicketsController.cs (minta)
```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using TicketingBearerApi.Data;
using TicketingBearerApi.Dtos;
using TicketingBearerApi.Models;

namespace TicketingBearerApi.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class TicketsController : ControllerBase
{
    private readonly AppDbContext _db;
    public TicketsController(AppDbContext db) => _db = db;

    private int CurrentUserId => int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
    private string CurrentRole => User.FindFirstValue(ClaimTypes.Role) ?? "customer";

    [HttpPost]
    public async Task<IActionResult> Create(TicketCreateRequest req)
    {
        var ticket = new Ticket
        {
            UserId = CurrentUserId,
            Subject = req.Subject.Trim(),
            Description = req.Description.Trim(),
            Priority = string.IsNullOrWhiteSpace(req.Priority) ? "medium" : req.Priority.Trim().ToLowerInvariant(),
            Status = "open"
        };

        _db.Tickets.Add(ticket);
        await _db.SaveChangesAsync();

        return CreatedAtAction(nameof(GetById), new { id = ticket.Id }, ticket);
    }

    [HttpGet]
    public async Task<IActionResult> List()
    {
        var q = _db.Tickets
            .Include(t => t.User)
            .AsQueryable();

        if (CurrentRole == "customer")
            q = q.Where(t => t.UserId == CurrentUserId);

        var items = await q
            .OrderByDescending(t => t.CreatedAt)
            .Select(t => new
            {
                t.Id,
                t.Subject,
                t.Priority,
                t.Status,
                t.CreatedAt,
                t.UpdatedAt,
                Owner = new { t.UserId, Name = t.User!.Name, Email = t.User!.Email }
            })
            .ToListAsync();

        return Ok(items);
    }

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id)
    {
        var t = await _db.Tickets
            .Include(x => x.User)
            .Include(x => x.Replies).ThenInclude(r => r.User)
            .FirstOrDefaultAsync(x => x.Id == id);

        if (t == null) return NotFound();

        if (CurrentRole == "customer" && t.UserId != CurrentUserId)
            return Forbid();

        return Ok(new
        {
            t.Id, t.Subject, t.Description, t.Priority, t.Status, t.CreatedAt, t.UpdatedAt,
            Owner = new { t.UserId, Name = t.User!.Name, Email = t.User!.Email },
            Replies = t.Replies
                .OrderBy(r => r.CreatedAt)
                .Select(r => new { r.Id, r.Message, r.CreatedAt, Author = new { r.UserId, r.User!.Name, r.User!.Role } })
        });
    }

    [HttpPatch("{id:int}/status")]
    [Authorize(Roles = "admin,agent")]
    public async Task<IActionResult> UpdateStatus(int id, TicketUpdateStatusRequest req)
    {
        var t = await _db.Tickets.FirstOrDefaultAsync(x => x.Id == id);
        if (t == null) return NotFound();

        t.Status = req.Status.Trim().ToLowerInvariant();
        t.UpdatedAt = DateTime.UtcNow;

        await _db.SaveChangesAsync();
        return Ok(t);
    }

    [HttpPatch("{id:int}/priority")]
    [Authorize(Roles = "admin,agent")]
    public async Task<IActionResult> UpdatePriority(int id, TicketUpdatePriorityRequest req)
    {
        var t = await _db.Tickets.FirstOrDefaultAsync(x => x.Id == id);
        if (t == null) return NotFound();

        t.Priority = req.Priority.Trim().ToLowerInvariant();
        t.UpdatedAt = DateTime.UtcNow;

        await _db.SaveChangesAsync();
        return Ok(t);
    }
}
```

---

### 9) TicketRepliesController (válaszok)

#### Jogosultság (javasolt)
- Ticket tulajdonosa (`customer`) válaszolhat a saját ticketjére.
- `agent` és `admin` bármely ticketre válaszolhat.

#### Controllers/TicketRepliesController.cs
```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using TicketingBearerApi.Data;
using TicketingBearerApi.Dtos;
using TicketingBearerApi.Models;

namespace TicketingBearerApi.Controllers;

[ApiController]
[Route("api/tickets/{ticketId:int}/replies")]
[Authorize]
public class TicketRepliesController : ControllerBase
{
    private readonly AppDbContext _db;
    public TicketRepliesController(AppDbContext db) => _db = db;

    private int CurrentUserId => int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
    private string CurrentRole => User.FindFirstValue(ClaimTypes.Role) ?? "customer";

    [HttpPost]
    public async Task<IActionResult> AddReply(int ticketId, TicketReplyCreateRequest req)
    {
        var ticket = await _db.Tickets.FirstOrDefaultAsync(t => t.Id == ticketId);
        if (ticket == null) return NotFound();

        var isOwner = ticket.UserId == CurrentUserId;
        var isStaff = CurrentRole is "admin" or "agent";

        if (!isOwner && !isStaff) return Forbid();

        var reply = new TicketReply
        {
            TicketId = ticketId,
            UserId = CurrentUserId,
            Message = req.Message.Trim()
        };

        _db.TicketReplies.Add(reply);

        ticket.UpdatedAt = DateTime.UtcNow;
        if (ticket.Status == "open" && isStaff)
            ticket.Status = "in_progress";

        await _db.SaveChangesAsync();

        return Created("", new { reply.Id, reply.TicketId, reply.UserId, reply.Message, reply.CreatedAt });
    }
}
```

---

### 10) Végpontok összefoglaló táblázata

| Terület | Metódus | Útvonal | Auth | Leírás |
|---|---:|---|---|---|
| Auth | POST | `/api/auth/register` | nyilvános | Regisztráció |
| Auth | POST | `/api/auth/login` | nyilvános | JWT token |
| Users | GET | `/api/users` | admin | Felhasználók listája |
| Users | GET | `/api/users/{id}` | admin | Felhasználó lekérése |
| Tickets | POST | `/api/tickets` | bejelentkezett | Ticket létrehozása |
| Tickets | GET | `/api/tickets` | bejelentkezett | Ticket lista (customer: csak saját) |
| Tickets | GET | `/api/tickets/{id}` | bejelentkezett | Ticket részletek + válaszok |
| Tickets | PATCH | `/api/tickets/{id}/status` | admin/agent | Státusz módosítás |
| Tickets | PATCH | `/api/tickets/{id}/priority` | admin/agent | Prioritás módosítás |
| Replies | POST | `/api/tickets/{ticketId}/replies` | bejelentkezett | Válasz hozzáadása |

---

## III. Tesztelés és dokumentáció

### 1) Migráció és adatbázis létrehozása
```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

### 2) Indítás
```bash
dotnet run
```

Fejlesztői módban a Swagger UI tipikusan elérhető:
- `https://localhost:xxxx/swagger`

### 3) Tipikus tesztelési folyamat (Postman / curl)

#### a) Regisztráció (customer)
```bash
curl -X POST "https://localhost:xxxx/api/auth/register" \
  -H "Content-Type: application/json" \
  -d '{"name":"Teszt User","email":"user@test.com","password":"Pass1234!","role":"customer"}'
```

#### b) Bejelentkezés és token
```bash
curl -X POST "https://localhost:xxxx/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"user@test.com","password":"Pass1234!"}'
```

Válasz:
```json
{ "token": "eyJhbGciOi..." }
```

#### c) Ticket létrehozás (Authorization: Bearer)
```bash
curl -X POST "https://localhost:xxxx/api/tickets" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"subject":"Nem működik a belépés","description":"A jelszó reset nem érkezik meg.","priority":"high"}'
```

#### d) Ticket lista
```bash
curl -X GET "https://localhost:xxxx/api/tickets" \
  -H "Authorization: Bearer <TOKEN>"
```

#### e) Válasz hozzáadása
```bash
curl -X POST "https://localhost:xxxx/api/tickets/1/replies" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"message":"További infó: a spam mappában sincs."}'
```

#### f) Státusz módosítás (agent/admin)
```bash
curl -X PATCH "https://localhost:xxxx/api/tickets/1/status" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <AGENT_OR_ADMIN_TOKEN>" \
  -d '{"status":"in_progress"}'
```

---

### 4) Swagger – Bearer Auth beállítás (opcionális)
Ha szeretnéd, hogy a Swagger UI-ban legyen **Authorize** gomb, egészítsd ki a Swagger konfigurációt.

> Röviden: `AddSwaggerGen`-ben add hozzá a `SecurityDefinition`-t és `SecurityRequirement`-et (Bearer).

---

### 5) Hibakezelés (minimum elvárás)
Javasolt egységes hibaválaszok:
- `400 BadRequest` – rossz input
- `401 Unauthorized` – nincs/rossz token
- `403 Forbidden` – nincs jogosultság
- `404 NotFound` – nem létező erőforrás
- `409 Conflict` – pl. foglalt email

---

## Kész
A fenti leírás alapján a projekt teljesíti a feladatot:
- 3 táblás adatmodell (users, tickets, ticket_replies)
- JWT Bearer autentikáció
- Role-alapú jogosultság (admin/agent/customer)
- CRUD jellegű ticket kezelés + válaszok
- Swagger és Postman/curl tesztelési útmutató
