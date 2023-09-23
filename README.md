# TodoApp

This is a refresher app for asp.net core.

##### Inital setup
1. Create a new "ToDoApp" web api
   
   `dotnet new webapi -n "TodoApp"`
2. Add Entity Framework packages
   
    `dotnet add package Microsoft.EntityFrameworkCore.Tools`

    `dotnet add package Microsoft.EntityFrameworkCore.Sqlite`

##### Create ToDo Model
1. Create a `Models` folder
2. Add `TodoItems.cs` file with following code
```csharp
namespace TodoApp.Models
{
    public class ToDoItem
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public string Description { get; set; }
        public bool IsComplete { get; set; }
    }
}
```

##### Create AppDbContext
1. Create `Data` folder
2. Add `AppDbContext.cs` file with following code
```csharp
using Microsoft.EntityFrameworkCore;
using TodoApp.Models;

namespace TodoApp
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions options) : base(options)
        {
        }

        public virtual DbSet<ToDoItem> ToDoItems { get; set; }
    }
}
```
3. Add ConnectionString in `appsettings.json` 
```json
"ConnectionStrings": {
    "DefaultConnection": "DataSource=app.db; Cache=Shared"
  },
```

##### Create Migration
1. Register `AppDbContext` in `program.cs` file with the Dependency Injection container
```csharp
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection") ?? throw new InvalidOperationException("Connection string 'DefaultConnection' not found.");
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(connectionString));
```
2. Open PackageManagerConsle and use the following commands
   `add-migration "Inital"`
   `update-database`


##### Create ToDoController
1. Add `ToDoController.cs` in Controller folder
2. Add following code:
   ```csharp
   
   private readonly AppDbContext _appDbContext;

        public ToDoController(AppDbContext appDbContext)
        {
            _appDbContext = appDbContext;
        }

        [HttpGet]
        public async Task<IActionResult> GetItems()
        {
            var toDoItem = await _appDbContext.ToDoItems.ToListAsync();
            return Ok(toDoItem);
        }

        [HttpPost]
        public async Task<IActionResult> CreateItem(ToDoItem toDoItem)
        {
            if (ModelState.IsValid)
            {
                await _appDbContext.ToDoItems.AddAsync(toDoItem);
                await _appDbContext.SaveChangesAsync();

                return CreatedAtAction("GetItemById", new { toDoItem.Id }, toDoItem);
            }
            return BadRequest("Something went wrong");
        }

        [HttpGet("{Id:int}")]
        public async Task<IActionResult> GetItemById(int Id)
        {
            var item = await _appDbContext.ToDoItems.FirstOrDefaultAsync(x => x.Id == Id);

            if(item == null)
            {
                return NotFound();
            }

            return Ok(item);
        }

        [HttpPut("{Id:int}")]
        public async Task<IActionResult> UpdateItem(int Id, ToDoItem itemToUpdate)
        {
            if(Id != itemToUpdate.Id)
            {
                return BadRequest();
            }

            var item = await _appDbContext.ToDoItems.FirstOrDefaultAsync(x => x.Id == Id);
            if (item == null)
            {
                return NotFound();
            }

            if (ModelState.IsValid)
            {
                item.Title = itemToUpdate.Title;
                item.Description = itemToUpdate.Description;
                item.IsComplete = itemToUpdate.IsComplete;

                await _appDbContext.SaveChangesAsync();

                return NoContent();
            }

            return BadRequest();
        }

        [HttpDelete("{Id:int}")]
        public async Task<IActionResult> DeleteItem(int Id)
        {
            var item = await _appDbContext.ToDoItems.FirstOrDefaultAsync(x => x.Id == Id);

            if (item == null)
            {
                return NotFound();
            }
            _appDbContext.ToDoItems.Remove(item);
            await _appDbContext.SaveChangesAsync();
            return Ok(item);
        }
   
   ```

## Functionality for Authentication using JWT

##### Add Packages
1. `Microsoft.AspNetCore.Identity.EntityFrameworkCore`
2. `Microsoft.AspNetCore.Authentication.JwtBearer`

##### JWT configuration
1. Add <b><i>Secret key</i></b> in **aspsettings.json**
```json
 "JWTConfig": {
  "Secret": "Super secret string used for encryption"
}
```

2. Add a **Configuration folder** folder.
3. Add **JWTConfig.cs** file which will have same sturcture as `JWTConfig` in **aspsettings.json**
```csharp
public class JWTConfig
    {
        public string Secret { get; set; }
    }
```
4. Make changes in **Program.cs** file for JWt Configuration and setup
```csharp
builder.Services.Configure<JWTConfig>(builder.Configuration.GetSection("JWTConfig"));

//existing 
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection") ?? throw new InvalidOperationException("Connection string 'DefaultConnection' not found.");
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(connectionString));

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
}).AddJwtBearer(jwt =>
{
    var key = Encoding.ASCII.GetBytes(builder.Configuration["JWTConfig:Secret"]);

    jwt.SaveToken = true;
    jwt.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(key),
        ValidateIssuer = false,
        ValidateAudience = false,
        ValidateLifetime = true,
        RequireExpirationTime = false,

    };
});

builder.Services.AddDefaultIdentity<IdentityUser>(options =>
{
    // Default Lockout settings.
    options.SignIn.RequireConfirmedAccount = true;
}).AddEntityFrameworkStores<AppDbContext>();

...
app.UseAuthentication();
app.UseAuthorization();

```

> [!IMPORTANT]
> Will need to install an additional packages `Microsoft.AspNetCore.Identity.UI`

## Add AuthenticationEndPoints

#### Inital Configuration
1.Add `AuthResult.cs` file
```csharp
    public class AuthResult
    {
        public string Token { get; set; }
        public bool Success { get; set; }
        public List<string> Error { get; set; }
    }
```
2. Create **DTO** folder inside **Models** for Request and Response
3. Create `UserRegistrationDTO.cs` file inside Request folder
```csharp
public class UserRegistrationDTO
    {
        [Required]
        public string UserName { get; set; }
        [Required]
        [EmailAddress]
        public string Email { get; set; }
        [Required]
        public string Password { get; set; }
    }
```
4. Create `UserLoginRequestDTO.cs` file inside Request folder
```csharp
public class UserLoginRequestDto
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }

        [Required]
        public string Password { get; set; }

    }
```
5. Create `RegistrationResponseDTO.cs` file inside Response folder
```csharp
public class RegistrationResponseDTO: AuthResult
    {
    }
```
#### Add AuthenticationController Method
1. Create file `AuthManagerController.cs` file
2. Add `Register` and `Login` EndPoint
```csharp
[Route("api/[controller]")]
    [ApiController]
    public class AuthManagerController : ControllerBase
    {
        private readonly UserManager<IdentityUser> _userManager;
        private readonly JWTConfig _jwtConfig;

        public AuthManagerController(UserManager<IdentityUser> userManager, IOptionsMonitor<JWTConfig> optionsMonitor)
        {
            _userManager = userManager;
            _jwtConfig = optionsMonitor.CurrentValue;
        }

        [HttpPost]
        [Route("Register")]
        public async Task<ActionResult> Register([FromBody] UserRegistrationDTO user)
        {
            if(ModelState.IsValid)
            {
                var existingUser = await _userManager.FindByEmailAsync(user.Email);
                if (existingUser != null)
                {
                    return BadRequest(new RegistrationResponseDTO()
                    {
                        Error = new List<string>() { "Email already exists." },
                        Success = false
                    });
                }

                var newUser = new IdentityUser() { Email = user.Email, UserName = user.UserName };
                var isCreated = await _userManager.CreateAsync(newUser, user.Password);

                if (isCreated.Succeeded)
                {
                    var jwtToken = GenerateJWTToken(newUser);

                    return Ok(new RegistrationResponseDTO()
                    {
                        Success = true,
                        Token = jwtToken
                    });
                }
                else
                {
                    return BadRequest(new RegistrationResponseDTO()
                    {
                        Error = isCreated.Errors.Select(x=> x.Description).ToList(),
                        Success = false
                    });
                }

            }

            return BadRequest(new RegistrationResponseDTO()
            {
                Error = new List<string>() { "Invalid Payload"},
                Success = false
            });
        }

        [HttpPost]
        [Route("Login")]
        public async Task<IActionResult> Login([FromBody] UserLoginRequestDto user)
        {
            if(ModelState.IsValid) 
            {
                var userExists = await _userManager.FindByEmailAsync(user.Email);

                if(userExists == null)
                {
                    return BadRequest(new RegistrationResponseDTO()
                    {
                        Error = new List<string>() { "No Such user exist" },
                        Success = false
                    });
                }

                var isCorrect = await _userManager.CheckPasswordAsync(userExists, user.Password);

                if (!isCorrect) {
                    return BadRequest(new RegistrationResponseDTO()
                    {
                        Error = new List<string>() { "Invalid Password" },
                        Success = false
                    });
                }

                var jwtToken = GenerateJWTToken(userExists);

                return Ok(new RegistrationResponseDTO()
                {
                    Success = true,
                    Token = jwtToken
                });
            }

            return BadRequest(new RegistrationResponseDTO()
            {
                Error = new List<string>() { "Invalid Payload" },
                Success = false
            });
        }

        private string GenerateJWTToken(IdentityUser user)
        {
            var jwtTokenHandler = new JwtSecurityTokenHandler();

            var key = Encoding.ASCII.GetBytes(_jwtConfig.Secret);

            var tokenDescriptor = new SecurityTokenDescriptor { 
                Subject = new ClaimsIdentity(new[]
                {
                    new Claim("Id", user.Id),
                    new Claim(JwtRegisteredClaimNames.Email , user.Email),
                    new Claim(JwtRegisteredClaimNames.Sub, user.Email),
                    new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
                }),
                Expires= DateTime.UtcNow.AddHours(6),
                SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature)
            };

            var token = jwtTokenHandler.CreateToken(tokenDescriptor);
            var JwtToken = jwtTokenHandler.WriteToken(token);

            return JwtToken;
        }

    }
```
3. Update `ToDoController.cs` file to add **Authorization** to the `Todo` endpoints
   ```csharp
   
    [Route("api/[controller]")]
    [ApiController]
    [Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]
    public class ToDoController : ControllerBase
   ```



