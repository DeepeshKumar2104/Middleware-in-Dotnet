
---

# JWT Authentication in .NET 6+ Web API

This is a sample setup for implementing JWT (JSON Web Token) authentication in a .NET 6+ Web API project. This guide provides only the essential code and configurations.

## Prerequisites

- .NET 6 SDK or later
- Basic understanding of Web API and JWT

## Project Setup

1. Create a new Web API project in .NET 6:
   ```bash
   dotnet new webapi -n JwtAuthDemo
   cd JwtAuthDemo
   ```

2. Install the necessary NuGet packages:
   ```bash
   dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
   dotnet add package System.IdentityModel.Tokens.Jwt
   ```

## Configuration

### appsettings.json

Add JWT settings to `appsettings.json`:
```json
"Jwt": {
  "Key": "YourSecretKeyHere",       // Replace with your secret key
  "Issuer": "YourIssuer",
  "Audience": "YourAudience"
}
```

### Program.cs

Configure JWT authentication in `Program.cs`:
```csharp
var builder = WebApplication.CreateBuilder(args);

// Add Authentication and JWT Bearer configuration
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
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

builder.Services.AddAuthorization();
builder.Services.AddControllers();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

### AuthController.cs

This controller provides a login endpoint and generates a JWT token:
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
}
```

### TestController.cs

This controller contains a protected endpoint accessible only with a valid JWT token:
```csharp
[Authorize]
[Route("api/[controller]")]
[ApiController]
public class TestController : ControllerBase
{
    [HttpGet("private")]
    public IActionResult PrivateEndpoint()
    {
        return Ok("This is a protected data accessible only with a valid JWT token.");
    }
}
```

### Models/UserLogin.cs

Define a `UserLogin` model to capture user credentials:
```csharp
public class UserLogin
{
    public string Username { get; set; }
    public string Password { get; set; }
}
```

## Testing the API

1. **Login**: Send a `POST` request to `/api/auth/login` with `Username` and `Password` in the request body. You’ll receive a JWT token if the credentials are correct.

2. **Access Protected Endpoint**: Use the token in the `Authorization` header with the `Bearer` scheme to access the protected `/api/test/private` endpoint.

