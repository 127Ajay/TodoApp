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

