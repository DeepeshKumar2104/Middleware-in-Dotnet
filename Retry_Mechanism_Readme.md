# Retry Mechanism in Web API: Complete Guide (Hinglish)

## 1. Retry Mechanism ka Parichay
Retry mechanism ek strategy hai jiska use automatically operation ko dubara attempt karne ke liye hota hai, jab operation fail ho jaye. Yeh usually tab use hota hai jab transient errors ho jayein, jaise ki network issues, temporary server unavailability, ya timeouts. Retry mechanism ka goal yeh hai ki system ki reliability aur resilience ko improve kiya ja sake, jisse multiple attempts mein operation successful ho sake.

## 2. Retry Mechanism kyun Use Karte Hain?
Jab client kisi Web API se communicate karta hai, toh network problems ya server issues ho sakte hain. Yeh problems kabhi-kabhi transient hote hain, matlab woh thode time baad khud se theek ho jate hain. Direct fail hone aur user experience ko disrupt karne ke bajaye, retry mechanism allow karta hai ki system in issues ko automatically handle kar sake. Iske faayde hain:

1. Temporary issues se better resilience.
2. User experience ko enhance karna errors kam karke.
3. Transient issues ke waqt manual intervention ki zaroorat kam karna.

## 3. Retry Mechanism Kaise Kaam Karta Hai?
Retry mechanism mein ek policy define ki jaati hai jo batati hai ki kitni baar operation ko retry kiya jayega aur har attempt ke beech kitna delay hona chahiye. Yeh steps hote hain:

1. **Failure Detection**: Jab operation fail hota hai, toh failure detect hota hai aur retry policy trigger hoti hai.
2. **Retry Logic**: System kuch time ke liye wait karta hai aur phir operation ko dubara try karta hai.
3. **Maximum Retry Attempts**: Retry mechanism fixed retries attempt karta hai, uske baad fail ho jata hai.

## 4. Retry Strategies ke Types
Alag-alag scenarios ke hisaab se different retry strategies use ho sakti hain:

1. **Fixed Interval**: Har attempt ke beech fixed time ka delay (e.g., har 2 seconds ke baad).
2. **Exponential Backoff**: Har retry pe progressively zyada delay (e.g., 2, 4, 8 seconds).
3. **Jitter**: Delay mein randomness add karna taki multiple clients ek saath retry karke overload na karein.

## 5. Real-time Implementation Polly ke Saath
Polly ek popular .NET library hai jo resilience aur transient-fault handling capabilities provide karti hai, jismein retry mechanism bhi shamil hai. Isse developers flexible aur readable tareeke se retry policies define kar sakte hain.

## 6. Example: Registration API with Retry Mechanism
Neeche ek example hai jisme registration API ASP.NET Core mein banaya gaya hai jo MySQL database mein user data save karta hai, aur retry mechanism Polly ke saath implement kiya gaya hai.

### Code Explanation:

1. **Retry Policy Definition**:
    ```csharp
    var retryPolicy = Policy.Handle<Exception>().WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));
    ```
    Yeh code ek retry policy define karta hai jo maximum 3 baar retry karta hai, exponential backoff ke saath (2, 4, 8 seconds). Yeh general exceptions ko handle karta hai, jaise ki database errors.

2. **Registration Logic**:
    ```csharp
    await retryPolicy.ExecuteAsync(async () => {
        // Logic jo user data ko database mein save karta hai
        await _context.Users.AddAsync(newUser);
        await _context.SaveChangesAsync();
    });
    ```
    Yeh logic retry policy ke andar wrap kiya gaya hai taaki transient errors, jaise connectivity problems, handle ho sakein.

3. **Handling Failures**:
    Agar saari retry attempts fail ho jayein, toh `catch` block ek `500 Internal Server Error` return karega ek descriptive message ke saath. Isse users ko pata chalega jab operation multiple retries ke baad fail ho jaye.

## 7. Complete Code Example
```csharp
using Microsoft.AspNetCore.Mvc;
using Polly;
using System;
using System.Threading.Tasks;

[Route("api/[controller]")]
[ApiController]
public class RegistrationController : ControllerBase
{
    private readonly ApplicationDbContext _context;

    public RegistrationController(ApplicationDbContext context)
    {
        _context = context;
    }

    [HttpPost]
    [Route("register")]
    public async Task<IActionResult> Register([FromBody] UserRegistrationModel user)
    {
        var retryPolicy = Policy
            .Handle<Exception>()
            .WaitAndRetryAsync(3, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

        try
        {
            await retryPolicy.ExecuteAsync(async () =>
            {
                var existingUser = await _context.Users.FirstOrDefaultAsync(u => u.Email == user.Email);
                if (existingUser != null)
                {
                    throw new Exception("User with this email already exists.");
                }

                var newUser = new User
                {
                    Username = user.Username,
                    Password = user.Password, // Note: Store hashed password in a real application
                    Email = user.Email
                };

                await _context.Users.AddAsync(newUser);
                await _context.SaveChangesAsync();
            });

            return Ok("Registration successful.");
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Registration failed: {ex.Message}");
        }
    }
}


# Cross-Origin Resource Sharing (CORS) in ASP.NET Core

## 1. What is CORS?
**CORS (Cross-Origin Resource Sharing)** ek security feature hai jo web browsers enforce karte hain. Yeh ensure karta hai ki ek web page jo ek origin se load hua hai, dusre origin ke resources access nahi kar sakta bina permission ke. Origin ka matlab hota hai **protocol**, **domain**, aur **port** ka combination. Agar ek request ek dusre origin se aati hai, toh woh "cross-origin" request hoti hai.

CORS use hota hai jab ek web application ko dusre server ya domain se APIs ya resources access karne ho. Default security policy browsers ki yeh hoti hai ki cross-origin requests ko block karen, aur CORS ko use karke aap define kar sakte hain ki kaunse origins allowed hain.

## 2. Why Use CORS?
CORS isliye use kiya jaata hai kyunki modern web applications mein, ek client application (for example, ek React ya Angular application) backend API se communicate karta hai jo kisi dusre server ya domain par ho sakta hai. Jab yeh origin alag hote hain, toh browser request ko block kar deta hai. CORS policies set karke, aap specific origins ko allow kar sakte hain.

## 3. How to Use CORS in ASP.NET Core (Program.cs)?

ASP.NET Core latest version mein, CORS ko configure karne ke liye `Program.cs` file mein CORS services ko register kiya jaata hai. Aap CORS policy ko define karke global level, controller level, ya action level par apply kar sakte hain.

### 3.1 Register CORS in `Program.cs`
Sabse pehle CORS ko `Program.cs` mein register karna hota hai. Yeh `.AddCors()` method use karke kiya jaata hai.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register CORS service
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigin", policy =>
    {
        policy.WithOrigins("https://example.com") // Specific origin ko allow karega
              .AllowAnyHeader()
              .AllowAnyMethod();
    });

    options.AddPolicy("AllowAllOrigins", policy =>
    {
        policy.AllowAnyOrigin() // Sare origins ko allow karega
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});

builder.Services.AddControllers();

var app = builder.Build();

// Use CORS in middleware pipeline
app.UseCors("AllowAllOrigins"); // Apply global CORS policy

app.UseRouting();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

### 3.2 Apply CORS at Controller Level
Agar aap chahte hain ki CORS policy sirf specific controllers par apply ho, toh aap `[EnableCors]` attribute use kar sakte hain:

```csharp
using Microsoft.AspNetCore.Cors;

[ApiController]
[Route("api/[controller]")]
[EnableCors("AllowSpecificOrigin")] // Sirf specific origin ko allow karne wali policy
public class MyController : ControllerBase
{
    [HttpGet]
    public IActionResult GetData()
    {
        return Ok("Data fetched successfully");
    }
}
```

### 3.3 Apply CORS at Action Level
Aap CORS ko sirf ek specific action method par bhi apply kar sakte hain:

```csharp
[ApiController]
[Route("api/[controller]")]
public class MyController : ControllerBase
{
    [HttpGet]
    [EnableCors("AllowSpecificOrigin")] // Sirf iss action par CORS policy apply hogi
    public IActionResult GetData()
    {
        return Ok("Data fetched successfully");
    }

    [HttpPost]
    [DisableCors] // CORS ko is action par disable kar diya gaya hai
    public IActionResult PostData()
    {
        return Ok("Data posted successfully");
    }
}
```

### 3.4 Applying CORS for Different Scenarios
- **Allow Specific Headers**: Agar aap sirf kuch specific headers ko allow karna chahte hain:
  ```csharp
  builder.Services.AddCors(options =>
  {
      options.AddPolicy("AllowSpecificHeaders", policy =>
      {
          policy.WithOrigins("https://example.com")
                .WithHeaders("Content-Type", "Authorization");
      });
  });
  ```
  
- **Allow Credentials**: Agar aap client ke credentials allow karna chahte hain (like cookies, HTTP authentication):
  ```csharp
  builder.Services.AddCors(options =>
  {
      options.AddPolicy("AllowWithCredentials", policy =>
      {
          policy.WithOrigins("https://example.com")
                .AllowCredentials();
      });
  });
  ```

### 4. Register CORS with Dependency Injection
Agar aapne `IServiceCollection` ko DI pattern ke saath use kiya hai, toh CORS policy ko services ke saath register kar sakte hain. Yeh approach aapko flexibility deta hai ki CORS ko application ke different parts mein reuse kar sakein.

```csharp
var builder = WebApplication.CreateBuilder(args);

// Register CORS policy using DI
builder.Services.AddCors(options =>
{
    options.AddPolicy("DefaultPolicy", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});

builder.Services.AddControllers();

var app = builder.Build();

// Use CORS with DefaultPolicy globally
app.UseCors("DefaultPolicy");

app.UseRouting();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### 5. Summary
- **CORS** ek mechanism hai jo cross-origin requests ko allow ya block karta hai. Yeh web security mein important role play karta hai.
- **Registration**: CORS policies ko `Program.cs` mein register kiya jaata hai.
- **Global Level Application**: Use `app.UseCors("policyName")` in middleware pipeline.
- **Controller or Action Level Application**: Use `[EnableCors("policyName")]` attribute on controllers or actions.
- **Disable CORS**: Use `[DisableCors]` attribute to prevent applying any CORS policy.

### 6. Best Practices
1. **Limit Origins**: Sirf un origins ko allow karein jo trusted hain. `AllowAnyOrigin` use karne se avoid karein jab tak zaroorat na ho.
2. **Use HTTPS**: CORS ko secure tariqe se implement karne ke liye, hamesha HTTPS use karein.
3. **Sensitive Data**: Agar credentials ya sensitive data send kar rahe hain, toh `AllowCredentials` carefully use karein.

By implementing CORS correctly, aap apni application ko secure rakh sakte hain aur ensure kar sakte hain ki sirf allowed clients hi aapke APIs access kar saken.
```

This `README.md` content provides a comprehensive guide on using CORS in ASP.NET Core using the latest `Program.cs` approach. It covers the registration of CORS policies, global and specific usage, as well as best practices for ensuring secure implementations.
