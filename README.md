{
  "Jwt": {
    "Key": "YourSuperSecretKeyHere",
    "Issuer": "YourIssuer",
    "Audience": "YourAudience",
    "ExpiresInMinutes": 30
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=BankingCRM;Trusted_Connection=True;MultipleActiveResultSets=true"
  }
}

// Models/User.cs
public class User
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string Password { get; set; } // In a real application, passwords should be hashed and salted
}

// Models/LoginRequest.cs
public class LoginRequest
{
    public string Username { get; set; }
    public string Password { get; set; }
}

// Models/RegisterRequest.cs
public class RegisterRequest
{
    public string Username { get; set; }
    public string Password { get; set; }
}

// Models/Customer.cs
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public List<Card> Cards { get; set; }
    public List<Loan> Loans { get; set; }
    public List<Deposit> Deposits { get; set; }
}

// Models/Card.cs
public class Card
{
    public int Id { get; set; }
    public string CardNumber { get; set; }
    public string Type { get; set; } // Debit or Credit
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
}

// Models/Loan.cs
public class Loan
{
    public int Id { get; set; }
    public string Type { get; set; } // Personal or Education
    public decimal Amount { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
}

// Models/Deposit.cs
public class Deposit
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
    public DateTime Date { get; set; }
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
}


// Data/BankingContext.cs
public class BankingContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Card> Cards { get; set; }
    public DbSet<Loan> Loans { get; set; }
    public DbSet<Deposit> Deposits { get; set; }

    public BankingContext(DbContextOptions<BankingContext> options) : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Define relationships and constraints if necessary
    }
}

// Services/IAuthService.cs
public interface IAuthService
{
    Task<string> Authenticate(LoginRequest loginRequest);
    Task Register(RegisterRequest registerRequest);
}

// Services/AuthService.cs
public class AuthService : IAuthService
{
    private readonly IConfiguration _configuration;
    private readonly BankingContext _context;

    public AuthService(IConfiguration configuration, BankingContext context)
    {
        _configuration = configuration;
        _context = context;
    }

    public async Task<string> Authenticate(LoginRequest loginRequest)
    {
        var user = await _context.Users.FirstOrDefaultAsync(u => u.Username == loginRequest.Username && u.Password == loginRequest.Password);

        if (user == null)
        {
            return null;
        }

        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.ASCII.GetBytes(_configuration["Jwt:Key"]);
        var tokenDescriptor = new SecurityTokenDescriptor
        {
            Subject = new ClaimsIdentity(new Claim[]
            {
                new Claim(ClaimTypes.Name, user.Username),
                new Claim(ClaimTypes.NameIdentifier, user.Id.ToString())
            }),
            Expires = DateTime.UtcNow.AddMinutes(Convert.ToInt32(_configuration["Jwt:ExpiresInMinutes"])),
            Issuer = _configuration["Jwt:Issuer"],
            Audience = _configuration["Jwt:Audience"],
            SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature)
        };
        var token = tokenHandler.CreateToken(tokenDescriptor);
        return tokenHandler.WriteToken(token);
    }

    public async Task Register(RegisterRequest registerRequest)
    {
        var user = new User
        {
            Username = registerRequest.Username,
            Password = registerRequest.Password // Hash the password in a real application
        };

        _context.Users.Add(user);
        await _context.SaveChangesAsync();
    }
}


public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbContext<BankingContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));

        services.AddControllers();
        services.AddSwaggerGen();

        services.AddTransient<IAuthService, AuthService>();

        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer = Configuration["Jwt:Issuer"],
                    ValidAudience = Configuration["Jwt:Audience"],
                    IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Configuration["Jwt:Key"]))
                };
            });

        services.AddAuthorization();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseHttpsRedirection();

        app.UseRouting();

        app.UseAuthentication();
        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });

        app.UseSwagger();
        app.UseSwaggerUI(c =>
        {
            c.SwaggerEndpoint("/swagger/v1/swagger.json", "BankingCRM API V1");
        });
    }
}


// Controllers/AuthController.cs
[Route("api/[controller]")]
[ApiController]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;

    public AuthController(IAuthService authService)
    {
        _authService = authService;
    }

    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequest loginRequest)
    {
        var token = await _authService.Authenticate(loginRequest);
        if (token == null)
        {
            return Unauthorized();
        }
        return Ok(new { Token = token });
    }

    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterRequest registerRequest)
    {
        await _authService.Register(registerRequest);
        return Ok();
    }
}

// Controllers/CardsController.cs
[Route("api/[controller]")]
[ApiController]
public class CardsController : ControllerBase
{
    private readonly BankingContext _context;

    public CardsController(BankingContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Card>>> GetCards()
    {
        return await _context.Cards.ToListAsync();
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Card>> GetCard(int id)
    {
        var card = await _context.Cards.FindAsync(id);

        if (card == null)
        {
            return NotFound();
        }

        return card;
    }

    [HttpPost]
    public async Task<ActionResult<Card>> CreateCard(Card card)
    {
        _context.Cards.Add(card);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetCard), new { id = card.Id }, card);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateCard(int id, Card card)
    {
        if (id != card.Id)
        {
            return BadRequest();
        }

        _context.Entry(card).State = EntityState.Modified;

        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!CardExists(id))
            {
                return NotFound();
            }
            else
            {
                throw;
            }
        }

        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteCard(int id)
    {
        var card = await _context.Cards.FindAsync(id);
        if (card == null)
        {
            return NotFound();
        }

        _context.Cards.Remove(card);
        await _context.SaveChangesAsync();

        return NoContent();
    }

    private bool CardExists(int id)
    {
        return _context.Cards.Any(e => e.Id == id);
    }
}


// Controllers/LoansController.cs
[Route("api/[controller]")]
[ApiController]
public class LoansController : ControllerBase
{
    private readonly BankingContext _context;

    public LoansController(BankingContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Loan>>> GetLoans()
    {
        return await _context.Loans.ToListAsync();
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Loan>> GetLoan(int id)
    {
        var loan = await _context.Loans.FindAsync(id);

        if (loan == null)
        {
            return NotFound();
        }

        return loan;
    }

    [HttpPost]
    public async Task<ActionResult<Loan>> CreateLoan(Loan loan)
    {
        _context.Loans.Add(loan);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetLoan), new { id = loan.Id }, loan);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateLoan(int id, Loan loan)
    {
        if (id != loan.Id)
        {
            return BadRequest();
        }

        _context.Entry(loan).State = EntityState.Modified;

        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!LoanExists(id))
            {
                return NotFound();
            }
            else
            {
                throw;
            }
        }

        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteLoan(int id)
    {
        var loan = await _context.Loans.FindAsync(id);
        if (loan == null)
        {
            return NotFound();
        }

        _context.Loans.Remove(loan);
        await _context.SaveChangesAsync();

        return NoContent();
    }

    private bool LoanExists(int id)
    {
        return _context.Loans.Any(e => e.Id == id);
    }
}


// Controllers/DepositsController.cs
[Route("api/[controller]")]
[ApiController]
public class DepositsController : ControllerBase
{
    private readonly BankingContext _context;

    public DepositsController(BankingContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Deposit>>> GetDeposits()
    {
        return await _context.Deposits.ToListAsync();
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Deposit>> GetDeposit(int id)
    {
        var deposit = await _context.Deposits.FindAsync(id);

        if (deposit == null)
        {
            return NotFound();
        }

        return deposit;
    }

    [HttpPost]
    public async Task<ActionResult<Deposit>> CreateDeposit(Deposit deposit)
    {
        _context.Deposits.Add(deposit);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetDeposit), new { id = deposit.Id }, deposit);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateDeposit(int id, Deposit deposit)
    {
        if (id != deposit.Id)
        {
            return BadRequest();
        }

        _context.Entry(deposit).State = EntityState.Modified;

        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!DepositExists(id))
            {
                return NotFound();
            }
            else
            {
                throw;
            }
        }

        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteDeposit(int id)
    {
        var deposit = await _context.Deposits.FindAsync(id);
        if (deposit == null)
        {
            return NotFound();
        }

        _context.Deposits.Remove(deposit);
        await _context.SaveChangesAsync();

        return NoContent();
    }

    private bool DepositExists(int id)
    {
        return _context.Deposits.Any(e => e.Id == id);
    }
}



public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.UseStartup<Startup>();
            });
}

public class Startup
{
    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public IConfiguration Configuration { get; }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbContext<BankingContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));

        services.AddControllers();
        services.AddSwaggerGen();

        services.AddTransient<IAuthService, AuthService>();

        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer = Configuration["Jwt:Issuer"],
                    ValidAudience = Configuration["Jwt:Audience"],
                    IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Configuration["Jwt:Key"]))
                };
            });

        services.AddAuthorization();
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseHttpsRedirection();

        app.UseRouting();

        app.UseAuthentication();
        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });

        app.UseSwagger();
        app.UseSwaggerUI(c =>
        {
            c.SwaggerEndpoint("/swagger/v1/swagger.json", "BankingCRM API V1");
        });
    }
}

dotnet ef migrations add InitialCreate
dotnet ef database update

