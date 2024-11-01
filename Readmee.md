JWT authentication setup ko ekdum depth me samjhte hain. Main har ek code line ka explanation dunga taaki aapko har line ka purpose samajh aaye. Hum yaha Program.cs file aur JWT generation aur validation ka code samjhenge jo humne upar likha hai.

Step 1: Program.cs Setup
Ye simplified structure .NET 6 aur uske upar versions me hota hai, jisme startup ka concept nahi hota aur saari configuration Program.cs me hoti hai.

Code
csharp
Copy code
var builder = WebApplication.CreateBuilder(args);
Explanation: Yaha WebApplication.CreateBuilder(args) ek builder object banata hai jo ASP.NET Core application ke liye configurations hold karta hai, jaise services, middleware, etc. args command-line arguments pass karte hain jo application ke startup ke waqt use hote hain.
csharp
Copy code
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
Explanation: AddAuthentication method ko call karke, hum JWT authentication ko configure kar rahe hain.
DefaultAuthenticateScheme aur DefaultChallengeScheme dono ko JwtBearerDefaults.AuthenticationScheme set kiya gaya hai, jo batata hai ki JWT bearer token ka use hoga authentication aur authorization ke liye.
JwtBearerDefaults.AuthenticationScheme ka matlab hai ki hum JWT tokens ko verify karenge jo client ke Authorization header me aayenge Bearer <token> format me.
csharp
Copy code
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
Explanation: AddJwtBearer method JWT Bearer Authentication scheme ko add karta hai. Isme hum JWT token validation ki settings define karte hain:
ValidateIssuer: Ye ensure karta hai ki token valid issuer (token banane wala) se aaya hai.
ValidateAudience: Ye ensure karta hai ki token valid audience (token ka intended user) ke liye hi hai.
ValidateLifetime: Ye token ki expiry check karta hai, taaki expired tokens accept na kiye jaayein.
ValidateIssuerSigningKey: Ye token ki integrity check karta hai, aur ensure karta hai ki token kisi authorized signing key ke sath sign kiya gaya hai.
ValidIssuer, ValidAudience, aur IssuerSigningKey: Ye values appsettings.json file se read ki jaati hain, aur ye configuration ke liye secret key aur issuer/audience ki values set karti hain.
IssuerSigningKey
IssuerSigningKey ek symmetric key hoti hai jo JWT ko sign aur validate karne ke liye use hoti hai. Yaha Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]) likh kar hum key ko UTF-8 bytes me convert karte hain jo token signing ke liye chahiye hoti hai.

csharp
Copy code
builder.Services.AddAuthorization();
builder.Services.AddControllers();
Explanation: AddAuthorization() se hum authorization services ko application me add kar rahe hain.
AddControllers() method controllers ko register karta hai, jo humare API endpoints define karte hain.
csharp
Copy code
var app = builder.Build();
Explanation: builder.Build() likh kar hum application ko build kar rahe hain jo ab middleware pipeline ke sath configure ho gaya hai aur run ke liye ready hai.
csharp
Copy code
app.UseAuthentication();
app.UseAuthorization();
Explanation: Yaha pe hum middleware pipeline me Authentication aur Authorization ko add kar rahe hain:
UseAuthentication(): Ye request ke saath incoming token ko validate karta hai.
UseAuthorization(): Ye check karta hai ki user ke paas required permissions hai ya nahi jo API ke endpoints access karne ke liye chahiye.
csharp
Copy code
app.MapControllers();
app.Run();
Explanation: MapControllers() humare controllers ke endpoints ko route karta hai, aur app.Run() likh kar application ko start karte hain.
Step 2: AuthController me JWT Token Generate Karna
AuthController Code:
csharp
Copy code
[Route("api/[controller]")]
[ApiController]
public class AuthController : ControllerBase
{
    private readonly IConfiguration _configuration;

    public AuthController(IConfiguration configuration)
    {
        _configuration = configuration;
    }
Explanation:
Route("api/[controller]"): Ye controller ke route ko specify karta hai. [controller] ko AuthController ke sath replace kiya jayega, toh endpoint banega api/auth.
ApiController: Ye attribute batata hai ki ye ek Web API controller hai jo HTTP requests handle karta hai.
IConfiguration: Ye dependency injection ke zariye configuration values (jese appsettings.json me JWT keys) ko controller me access karne me madad karta hai.
Login Method:
csharp
Copy code
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
Explanation:
HttpPost("login"): Is method ka route hai api/auth/login, jo POST request ke liye accessible hai.
Login method client se user credentials accept karta hai (UserLogin model me).
if condition se username aur password ko check kiya jaata hai. Agar ye valid hote hain, toh token generate karne ke liye GenerateJwtToken method ko call karte hain.
Ok(new { token }): Agar login successful hota hai toh token return karte hain.
Unauthorized(): Agar credentials galat hote hain, toh 401 Unauthorized response return hota hai.
GenerateJwtToken Method
csharp
Copy code
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
Explanation:
JwtSecurityTokenHandler: Ye class JWT tokens ko create aur validate karne ka kaam karti hai.
key: appsettings.json se secret key ko read karke UTF-8 bytes me convert karte hain.
SecurityTokenDescriptor: Ye JWT token ka descriptor banata hai jo token ka structure define karta hai:
Subject: Isme user ke claims define kiye gaye hain jaise Name aur Role. Ye claims token me store honge aur identify karenge ki user kaun hai aur uska role kya hai.
Expires: Token ki expiry time 30 minutes set ki gayi hai.
Issuer aur Audience: Ye token ka issuer aur audience specify karte hain, jo appsettings.json se load ki gayi hai.
SigningCredentials: Ye batata hai ki token ko sign karne ke liye kaun si key aur algorithm use kiya jaayega. Yaha HmacSha256Signature algorithm use hua hai.
CreateToken(tokenDescriptor): Ye method JWT token create karta hai, jo ki tokenDescriptor pe based hota hai.
WriteToken(token): Ye method token ko string format me convert karke return karta hai, jo client ko bheja ja sakta hai.
Step 3: TestController - Protected Endpoint
Code:
csharp
Copy code
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
Explanation:
[Authorize]: Ye attribute batata hai ki ye endpoint sirf tabhi access hoga jab request me valid JWT token ho.
Route("api/[controller]"): Iska route api/test/private hai.
PrivateEndpoint(): Ye method ek private data return karta hai, jo sirf authenticated users ke liye accessible hai.
