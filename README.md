# Useful_command
Código C#

Criar um Novo Projeto Web API no .NET 9

```sh
dotnet new webapi -n BiroAPI
cd BiroAPI
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Swashbuckle.AspNetCore
dotnet add package Microsoft.AspNetCore.Cors

```

ConnectionStrings
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=SeuBanco;Username=SeuUsuario;Password=SuaSenha"
  },
  "AllowedHosts": "*"
}
```

# 🚀 Mapeando uma Tabela Existente do PostgreSQL no .NET 9  

Este guia mostra como mapear automaticamente uma tabela existente no PostgreSQL para um projeto **.NET 9** usando **Entity Framework Core**.  

## 📌 **Pré-requisitos**  

Antes de começar, certifique-se de que seu projeto tem os pacotes necessários:  

```sh
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet tool install --global dotnet-ef
```

# Verificando a Tabela no PostgreSQL
\d clientes
# Lista as tabelas no DB.
\dt

Gerando as Classes do Banco Automaticamente
```sh
dotnet ef dbcontext scaffold "Host=SEU_HOST;Database=SEU_BANCO;Username=SEU_USUARIO;Password=SUA_SENHA" Npgsql.EntityFrameworkCore.PostgreSQL -o Models --context-dir Data --force
```
Explicação do comando:

scaffold → Gera as classes do banco automaticamente.
Host=SEU_HOST;Database=SEU_BANCO;Username=SEU_USUARIO;Password=SUA_SENHA → String de conexão com o banco PostgreSQL.
Npgsql.EntityFrameworkCore.PostgreSQL → Provedor do PostgreSQL.
-o Models → Salva as entidades na pasta Models/.
--context-dir Data → Salva o DbContext na pasta Data/.
--force → Sobrescreve arquivos existentes (caso necessário).

-------------------------------
=> JWT

```sh
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.IdentityModel.Tokens
dotnet add package System.IdentityModel.Tokens.Jwt
```
Model
```c#
using Microsoft.AspNetCore.Identity;

public class ApplicationUser : IdentityUser
{
    public string NomeCompleto { get; set; }
}
```
Context
```C#
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

public class BackendDbContext : IdentityDbContext<ApplicationUser>
{
    public BackendDbContext(DbContextOptions<BackendDbContext> options) : base(options) { }
}
```
Program.cs
```C#
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

// Configurar banco de dados PostgreSQL
builder.Services.AddDbContext<BackendDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// Configurar Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole>()
    .AddEntityFrameworkStores<BackendDbContext>()
    .AddDefaultTokenProviders();

// Configurar autenticação JWT
var key = Encoding.UTF8.GetBytes(builder.Configuration["Jwt:SecretKey"]);
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.RequireHttpsMetadata = false;
        options.SaveToken = true;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(key),
            ValidateIssuer = false,
            ValidateAudience = false
        };
    });

// Adicionar autorização
builder.Services.AddAuthorization();

var app = builder.Build();

// Configurar autenticação e autorização
app.UseAuthentication();
app.UseAuthorization();

app.Run();
```

Service
```C#
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Identity;
using Microsoft.IdentityModel.Tokens;

public class AuthService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly IConfiguration _configuration;

    public AuthService(UserManager<ApplicationUser> userManager, IConfiguration configuration)
    {
        _userManager = userManager;
        _configuration = configuration;
    }

    public async Task<string> GenerateJwtToken(ApplicationUser user)
    {
        var userRoles = await _userManager.GetRolesAsync(user);
        var claims = new List<Claim>
        {
            new Claim(JwtRegisteredClaimNames.Sub, user.Id),
            new Claim(JwtRegisteredClaimNames.Email, user.Email)
        };

        claims.AddRange(userRoles.Select(role => new Claim(ClaimTypes.Role, role)));

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:SecretKey"]));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var token = new JwtSecurityToken(
            expires: DateTime.UtcNow.AddHours(2),
            claims: claims,
            signingCredentials: creds
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

Controller
```C#
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/auth")]
public class AuthController : ControllerBase
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly AuthService _authService;

    public AuthController(UserManager<ApplicationUser> userManager, SignInManager<ApplicationUser> signInManager, AuthService authService)
    {
        _userManager = userManager;
        _signInManager = signInManager;
        _authService = authService;
    }

    [HttpPost("register")]
    public async Task<IActionResult> Register([FromBody] RegisterRequest model)
    {
        var user = new ApplicationUser
        {
            UserName = model.Email,
            Email = model.Email,
            NomeCompleto = model.NomeCompleto
        };

        var result = await _userManager.CreateAsync(user, model.Password);
        if (!result.Succeeded)
            return BadRequest(result.Errors);

        return Ok("Usuário registrado com sucesso!");
    }

    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequest model)
    {
        var user = await _userManager.FindByEmailAsync(model.Email);
        if (user == null)
            return Unauthorized("Credenciais inválidas");

        var result = await _signInManager.PasswordSignInAsync(user, model.Password, false, false);
        if (!result.Succeeded)
            return Unauthorized("Credenciais inválidas");

        var token = await _authService.GenerateJwtToken(user);
        return Ok(new { Token = token });
    }
}
```
Request
```C#
public class LoginRequest
{
    public string Email { get; set; }
    public string Password { get; set; }
}

public class RegisterRequest
{
    public string NomeCompleto { get; set; }
    public string Email { get; set; }
    public string Password { get; set; }
}

```

appsettings.json
```C#
{
  "Jwt": {
    "SecretKey": "minha-chave-secreta-ultra-segura-para-o-jwt"
  },
  "ConnectionStrings": {
    "DefaultConnection": "Host=IP_DO_BANCO;Database=SEU_BANCO;Username=SEU_USER;Password=SUA_SENHA"
  }
}
```
PesquisaController.cs
```C#
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/pesquisa")]
[Authorize] // 🔒 Apenas usuários autenticados podem acessar
public class PesquisaController : ControllerBase
{
    private readonly PesquisaService _pesquisaService;

    public PesquisaController(PesquisaService pesquisaService)
    {
        _pesquisaService = pesquisaService;
    }

    [HttpGet("/{cpf}")]
    public async Task<IActionResult> GetPesquisaClienteCpf([FromRoute] string cpf)
    {
        var pesquisa = await _pesquisaService.GetClientePorCpf(cpf);
        return pesquisa is not null ? Ok(pesquisa) : NotFound();
    }
}
```
- Se quiser que apenas admins acessem, use [Authorize(Roles = "Admin")].


Capturar o Usuário Logado
```C#
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;

[ApiController]
[Route("api/pesquisa")]
[Authorize] // 🔒 Apenas usuários autenticados podem acessar
public class PesquisaController : ControllerBase
{
    private readonly PesquisaService _pesquisaService;

    public PesquisaController(PesquisaService pesquisaService)
    {
        _pesquisaService = pesquisaService;
    }

    [HttpGet("/{cpf}")]
    public async Task<IActionResult> GetPesquisaClienteCpf([FromRoute] string cpf)
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value; // Captura o ID do usuário logado
        if (userId == null)
        {
            return Unauthorized("Usuário não autenticado.");
        }

        var pesquisa = await _pesquisaService.GetClientePorCpf(cpf, userId); // Passa o ID para registrar
        return pesquisa is not null ? Ok(pesquisa) : NotFound();
    }
}
```
Modificar o Serviço para Registrar a Pesquisa

```C#
using Microsoft.EntityFrameworkCore;

public class PesquisaService
{
    private readonly BackendDbContext _context;

    public PesquisaService(BackendDbContext context)
    {
        _context = context;
    }

    public async Task<Cliente?> GetClientePorCpf(string cpf, string userId)
    {
        var cliente = await _context.Clientes.FirstOrDefaultAsync(c => c.Cpf == cpf);
        
        if (cliente != null)
        {
            // Registrar a pesquisa vinculada ao usuário logado
            var pesquisa = new Pesquisa
            {
                ClienteId = cliente.Id,
                UsuarioId = userId, // Quem fez a pesquisa
                DataHoraPesquisa = DateTime.UtcNow
            };

            _context.Pesquisas.Add(pesquisa);
            await _context.SaveChangesAsync();
        }

        return cliente;
    }
}
```
Tabela de Pesquisa

```C#
public class Pesquisa
{
    public int Id { get; set; }
    public int ClienteId { get; set; } // Cliente pesquisado
    public string UsuarioId { get; set; } // Quem fez a pesquisa
    public DateTime DataHoraPesquisa { get; set; }
}
```
Context
```C#
public DbSet<Pesquisa> Pesquisas { get; set; }
```

```sh
dotnet ef migrations add AddTabelaPesquisa
dotnet ef database update
```
Consultar Pesquisas Feitas pelo Usuário
```C#
[HttpGet("minhas-pesquisas")]
public async Task<IActionResult> GetMinhasPesquisas()
{
    var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    if (userId == null)
    {
        return Unauthorized("Usuário não autenticado.");
    }

    var pesquisas = await _pesquisaService.GetPesquisasPorUsuario(userId);
    return Ok(pesquisas);
}
```
Services/PesquisaService.cs
```C#
public async Task<List<Pesquisa>> GetPesquisasPorUsuario(string userId)
{
    return await _context.Pesquisas
        .Where(p => p.UsuarioId == userId)
        .OrderByDescending(p => p.DataHoraPesquisa)
        .ToListAsync();
}
```

```txt
Testando
Agora você pode:
Fazer login e obter um token JWT (POST /api/auth/login)
Usar o token no header das requisições Authorization: Bearer {seu-token}
Consultar um CPF (GET /api/pesquisa/{cpf}) e ele será registrado com o usuário logado
Ver todas as pesquisas feitas pelo usuário (GET /api/pesquisa/minhas-pesquisas)
```









