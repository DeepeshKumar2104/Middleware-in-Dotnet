### JWT Authentication Setup in .NET 6+ (Hinglish Explanation)

Is README me hum JWT authentication ka setup karenge .NET 6+ ke simplified structure me. Har step aur code line ka deep explanation diya gaya hai taaki aapko har line ka purpose samajh aaye. Yahaan hum `Program.cs` me JWT generation aur validation ka code setup karenge.

---

### Step 1: `Program.cs` Setup

.NET 6+ me `Startup` ka concept nahi hota aur saari configurations `Program.cs` me hoti hai.

```csharp
var builder = WebApplication.CreateBuilder(args);
```

**Explanation**: `WebApplication.CreateBuilder(args)` ek `builder` object banata hai jo ASP.NET Core application ke configurations ko hold karta hai, jaise services, middleware, etc. `args` yaha command-line arguments pass karta hai jo application startup ke waqt use hote hain.

---

#### JWT Authentication Configure karna:

```csharp
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
```

**Explanation**: `AddAuthentication` method JWT authentication ko configure karta hai. `DefaultAuthenticateScheme` aur `DefaultChallengeScheme` dono ko `JwtBearerDefaults.AuthenticationScheme` set kiya gaya hai, jo batata hai ki JWT bearer token ka use hoga authentication aur authorization ke liye. Matlab, hum `Authorization` header me `Bearer <token>` format ka token check karenge.

```csharp
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        ValidAudience = builder.Configuration["Jwt:Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
    };
});
```

**Explanation**: `AddJwtBearer` method JWT token validation parameters set karta hai:
- `ValidateIssuer`: Ensure karta hai ki token valid issuer (token banane wala) se aaya hai.
- `ValidateAudience`: Ensure karta hai ki token valid audience ke liye hai.
- `ValidateLifetime`: Expiry check karta hai taaki expired token accept na ho.
- `ValidateIssuerSigningKey`: Token ki integrity check karta hai jo authorized signing key se sign hota hai.
- `ValidIssuer`, `ValidAudience`, `IssuerSigningKey`: Yeh values `appsettings.json` se uthayi jaati hain.

**`IssuerSigningKey`** ek symmetric key hai jo token sign aur validate karne ke liye use hoti hai.

---

### Services Register Karna:

```csharp
builder.Services.AddAuthorization();
builder.Services.AddControllers();
```

**Explanation**: `AddAuthorization()` authorization services ko register karta hai aur `AddControllers()` method API controllers ko register karta hai, jo API endpoints ko define karte hain.

---

### Application Build aur Middleware Pipeline Setup:

```csharp
var app = builder.Build();
```

**Explanation**: `builder.Build()` likh ke hum application ko build karte hain jo ab middleware ke saath configure ho gaya hai.

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

**Explanation**:
- `UseAuthentication()`: Yeh middleware pipeline me request ke saath incoming token ko validate karta hai.
- `UseAuthorization()`: Yeh check karta hai ki user ke paas required permissions hain jo API endpoints ko access karne ke liye chahiye.

---

### Routes Map aur Application Start karna:

```csharp
app.MapControllers();
app.Run();
```

**Explanation**:
- `MapControllers()`: Controllers ke endpoints ko route karta hai.
- `app.Run()`: Application start karne ke liye ye command use hoti hai.

---

### Step 2: `AuthController` me JWT Token Generate karna

#### `AuthController` Code:

```csharp
[Route("api/[controller]")]
[ApiController]
public class AuthController : ControllerBase
{
    private readonly IConfiguration _configuration;

    public AuthController(IConfiguration configuration)
    {
        _configuration = configuration;
    }
}
```

**Explanation**:
- `[Route("api/[controller]")]`: Controller ka route set karta hai (e.g., `api/auth`).
- `[ApiController]`: Batata hai ki ye Web API controller hai jo HTTP requests handle karta hai.
- `IConfiguration`: Configuration values ko inject karta hai, jaise `appsettings.json` me JWT ke keys.

---

#### `Login` Method:

```csharp
[HttpPost("login")]
public IActionResult Login([FromBody] UserLogin user)
{
    if (user.Username == "testuser" && user.Password == "password")
    {
        var token = GenerateJwtToken();
        return Ok(new { token });
    }
    return Unauthorized();
}
```

**Explanation**:
- `[HttpPost("login")]`: Is method ka route `api/auth/login` hai aur ye POST request handle karta hai.
- `Login` method user credentials accept karta hai. Agar credentials valid hain toh `GenerateJwtToken` method ko call karta hai aur token return karta hai; agar nahi toh `Unauthorized()` response return hota hai.

---

#### `GenerateJwtToken` Method:

```csharp
private string GenerateJwtToken()
{
    var tokenHandler = new JwtSecurityTokenHandler();
    var key = Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]);
    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.Name, "testuser"),
            new Claim(ClaimTypes.Role, "Admin")
        }),
        Expires = DateTime.UtcNow.AddMinutes(30),
        Issuer = _configuration["Jwt:Issuer"],
        Audience = _configuration["Jwt:Audience"],
        SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature)
    };
    var token = tokenHandler.CreateToken(tokenDescriptor);
    return tokenHandler.WriteToken(token);
}
```

**Explanation**:
- `JwtSecurityTokenHandler`: JWT tokens ko create aur validate karne ka kaam karta hai.
- `key`: Secret key ko UTF-8 bytes me convert karta hai.
- `SecurityTokenDescriptor`: Token structure ko define karta hai, jaise `Claims` (user ka name aur role), `Expires`, `Issuer`, `Audience`, `SigningCredentials`.
- `CreateToken`: Token generate karta hai.
- `WriteToken`: Token ko string format me convert karke client ko bhejta hai.

---

### Step 3: Protected Endpoint Setup in `TestController`

```csharp
[Authorize]
[Route("api/[controller]")]
[ApiController]
public class TestController : ControllerBase
{
    [HttpGet("private")]
    public IActionResult PrivateEndpoint()
    {
        return Ok("Yeh private data hai jo sirf authenticated users ke liye hai!");
    }
}
```

**Explanation**:
- `[Authorize]`: Is attribute se ensure hota hai ki ye endpoint sirf authenticated users ke liye accessible hai.
- `PrivateEndpoint`: Ye ek private data return karta hai jo sirf JWT authenticated users access kar sakte hain.

---

### appsettings.json

Aapko JWT token ke issuer, audience aur key define karne ke liye `appsettings.json` file me settings add karni hogi:

```json
"Jwt": {
  "Issuer": "YourIssuer",
  "Audience": "YourAudience",
  "Key": "YourSecretKey"
}
```

Ye `Issuer`, `Audience`, aur `Key` aapke JWT token generation aur validation ke liye use hote hain.

--- 

Yeh tha JWT authentication ka complete setup with .NET 6+ me, har line ka explanation ke saath.
