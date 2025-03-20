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
-----
dotnet add package Microsoft.AspNetCore.Authentication --version 2.3.0
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer --version 9.0.3

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
```sh
export Jwt__SecretKey="G6jT2XkqfhdP4JZmDlL0+N9sU6K0Zw=="
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
Front Angular

```sh
ng new frontend --routing
cd frontend
ng add @angular-eslint/schematics
npm install bootstrap
```
Add no angular.json
```json
"styles": [
  "node_modules/bootstrap/dist/css/bootstrap.min.css",
  "src/styles.css"
]
```
Componente
```sh
ng g c pages/cadastro
ng g c pages/login
ng g c pages/pesquisa
ng g s services/auth
ng g s services/pesquisa
```
app-routing.module.ts
```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { CadastroComponent } from './pages/cadastro/cadastro.component';
import { LoginComponent } from './pages/login/login.component';
import { PesquisaComponent } from './pages/pesquisa/pesquisa.component';

const routes: Routes = [
  { path: '', redirectTo: 'login', pathMatch: 'full' },
  { path: 'cadastro', component: CadastroComponent },
  { path: 'login', component: LoginComponent },
  { path: 'pesquisa', component: PesquisaComponent }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```
auth.service.ts
```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private apiUrl = 'http://localhost:5000/api/auth'; // Altere para sua API

  constructor(private http: HttpClient) { }

  cadastrar(dados: any): Observable<any> {
    return this.http.post(`${this.apiUrl}/register`, dados);
  }

  login(dados: any): Observable<any> {
    return this.http.post(`${this.apiUrl}/login`, dados);
  }
}
```
pesquisa.service.ts
```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class PesquisaService {
  private apiUrl = 'http://localhost:5000/api/pesquisa'; // Altere para sua API

  constructor(private http: HttpClient) { }

  pesquisarCpf(cpf: string): Observable<any> {
    return this.http.get(`${this.apiUrl}/${cpf}`);
  }
}
```
cadastro.component.ts
```ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';
import { AuthService } from 'src/app/services/auth.service';
import { Router } from '@angular/router';

@Component({
  selector: 'app-cadastro',
  templateUrl: './cadastro.component.html',
  styleUrls: ['./cadastro.component.css']
})
export class CadastroComponent {
  cadastroForm: FormGroup;

  constructor(private fb: FormBuilder, private authService: AuthService, private router: Router) {
    this.cadastroForm = this.fb.group({
      nome: [''],
      email: [''],
      senha: ['']
    });
  }

  cadastrar() {
    this.authService.cadastrar(this.cadastroForm.value).subscribe(() => {
      this.router.navigate(['/login']);
    });
  }
}
```
cadastro.component.html
```ts
<div class="container mt-5">
  <h2>Cadastro</h2>
  <form [formGroup]="cadastroForm" (ngSubmit)="cadastrar()">
    <div class="mb-3">
      <label>Nome Completo</label>
      <input class="form-control" formControlName="nome">
    </div>
    <div class="mb-3">
      <label>E-mail</label>
      <input class="form-control" type="email" formControlName="email">
    </div>
    <div class="mb-3">
      <label>Senha</label>
      <input class="form-control" type="password" formControlName="senha">
    </div>
    <button class="btn btn-primary w-100" type="submit">Cadastrar</button>
  </form>
</div>
```
login.component.ts
```ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';
import { AuthService } from 'src/app/services/auth.service';
import { Router } from '@angular/router';

@Component({
  selector: 'app-login',
  templateUrl: './login.component.html',
  styleUrls: ['./login.component.css']
})
export class LoginComponent {
  loginForm: FormGroup;

  constructor(private fb: FormBuilder, private authService: AuthService, private router: Router) {
    this.loginForm = this.fb.group({
      email: [''],
      senha: ['']
    });
  }

  login() {
    this.authService.login(this.loginForm.value).subscribe(() => {
      this.router.navigate(['/pesquisa']);
    });
  }
}
```
login.component.html
```html
<div class="container mt-5">
  <h2>Login</h2>
  <form [formGroup]="loginForm" (ngSubmit)="login()">
    <div class="mb-3">
      <label>E-mail</label>
      <input class="form-control" type="email" formControlName="email">
    </div>
    <div class="mb-3">
      <label>Senha</label>
      <input class="form-control" type="password" formControlName="senha">
    </div>
    <button class="btn btn-primary w-100" type="submit">Entrar</button>
  </form>
</div>
```
pesquisa.component.ts
```ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';
import { PesquisaService } from 'src/app/services/pesquisa.service';

@Component({
  selector: 'app-pesquisa',
  templateUrl: './pesquisa.component.html',
  styleUrls: ['./pesquisa.component.css']
})
export class PesquisaComponent {
  pesquisaForm: FormGroup;
  resultado: any = null;

  constructor(private fb: FormBuilder, private pesquisaService: PesquisaService) {
    this.pesquisaForm = this.fb.group({ cpf: [''] });
  }

  pesquisar() {
    const cpf = this.pesquisaForm.value.cpf;
    this.pesquisaService.pesquisarCpf(cpf).subscribe(data => {
      this.resultado = data;
    });
  }
}
```
pesquisa.component.html
```html
<div class="container mt-5">
  <h2>Pesquisa de CPF</h2>
  <input class="form-control" placeholder="Digite o CPF" formControlName="cpf">
  <button class="btn btn-primary mt-2" (click)="pesquisar()">Pesquisar</button>

  <div *ngIf="resultado" class="mt-3">
    <h4>Resultado:</h4>
    <p>Nome: {{ resultado.nome }}</p>
    <p>CPF: {{ resultado.cpf }}</p>
  </div>
</div>
```
Autenticados acessem a tela de pesquisa.

```sh
ng g g guards/auth
```
/auth.guard.ts
```ts
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {
  constructor(private router: Router) {}

  canActivate(): boolean {
    const token = localStorage.getItem('token');
    if (!token) {
      this.router.navigate(['/login']);
      return false;
    }
    return true;
  }
}
```
 Aplique o AuthGuard na rota de pesquisa. Edite src/app/app-routing.module.ts:
```ts
const routes: Routes = [
  { path: '', redirectTo: 'login', pathMatch: 'full' },
  { path: 'cadastro', component: CadastroComponent },
  { path: 'login', component: LoginComponent },
  { path: 'pesquisa', component: PesquisaComponent, canActivate: [AuthGuard] }
];
```
Ajustar o AuthService para armazenar o token
- auth.service.ts
```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { Router } from '@angular/router';

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private apiUrl = 'http://localhost:5000/api/auth';

  constructor(private http: HttpClient, private router: Router) { }

  cadastrar(dados: any): Observable<any> {
    return this.http.post(`${this.apiUrl}/register`, dados);
  }

  login(dados: any): Observable<any> {
    return this.http.post(`${this.apiUrl}/login`, dados).pipe(
      tap((res: any) => {
        localStorage.setItem('token', res.token);
        this.router.navigate(['/pesquisa']);
      })
    );
  }

  logout() {
    localStorage.removeItem('token');
    this.router.navigate(['/login']);
  }

  estaLogado(): boolean {
    return !!localStorage.getItem('token');
  }
}
```
login.component.html
```html
<div class="container mt-5">
  <h2>Login</h2>
  <form [formGroup]="loginForm" (ngSubmit)="login()">
    <div class="mb-3">
      <label>E-mail</label>
      <input class="form-control" type="email" formControlName="email">
    </div>
    <div class="mb-3">
      <label>Senha</label>
      <input class="form-control" type="password" formControlName="senha">
    </div>
    <button class="btn btn-primary w-100" type="submit">Entrar</button>
  </form>

  <p class="mt-3">
    Ainda não tem uma conta? <a routerLink="/cadastro">Cadastre-se aqui</a>.
  </p>
</div>
```
 Botão de Logout
 pesquisa.component.html
 ```html
<div class="container mt-5">
  <div class="d-flex justify-content-between align-items-center">
    <h2>Pesquisa de CPF</h2>
    <button class="btn btn-danger" (click)="logout()">Sair</button>
  </div>
  
  <input class="form-control mt-3" placeholder="Digite o CPF" formControlName="cpf">
  <button class="btn btn-primary mt-2" (click)="pesquisar()">Pesquisar</button>

  <div *ngIf="resultado" class="mt-3">
    <h4>Resultado:</h4>
    <p>Nome: {{ resultado.nome }}</p>
    <p>CPF: {{ resultado.cpf }}</p>
  </div>
</div>
```
Edite src/app/pages/pesquisa/pesquisa.component.ts
```ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';
import { PesquisaService } from 'src/app/services/pesquisa.service';
import { AuthService } from 'src/app/services/auth.service';

@Component({
  selector: 'app-pesquisa',
  templateUrl: './pesquisa.component.html',
  styleUrls: ['./pesquisa.component.css']
})
export class PesquisaComponent {
  pesquisaForm: FormGroup;
  resultado: any = null;

  constructor(
    private fb: FormBuilder,
    private pesquisaService: PesquisaService,
    private authService: AuthService
  ) {
    this.pesquisaForm = this.fb.group({ cpf: [''] });
  }

  pesquisar() {
    const cpf = this.pesquisaForm.value.cpf;
    this.pesquisaService.pesquisarCpf(cpf).subscribe(data => {
      this.resultado = data;
    });
  }

  logout() {
    this.authService.logout();
  }
}
```
=> Verificação 3: Interceptor Global (Caso Precise de Autenticação)

```sh
ng generate service interceptors/auth
```

```ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler } from '@angular/common/http';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const token = localStorage.getItem('token');

    if (token) {
      const clonedReq = req.clone({
        headers: req.headers.set('Authorization', `Bearer ${token}`)
      });
      return next.handle(clonedReq);
    }

    return next.handle(req);
  }
}
```

 Adicione no app.module.ts

 ```ts
import { HTTP_INTERCEPTORS } from '@angular/common/http';
import { AuthInterceptor } from './interceptors/auth.interceptor';

@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
  ]
})
```
Se ainda der erro 404, tente rodar o front com o proxy do Angular
```sh
ng serve --proxy-config proxy.conf.json
```
Se nada funcionar, tente chamar a API diretamente no navegador
```txt
http://localhost:5000/api/pesquisa/xxxxxxxxxx
```
Código APÓS o Interceptor - pesquisarCpf

```ts
pesquisarCpf(cpf: string): Observable<any> {
  return this.http.get(`${this.apiUrl}/${cpf}`);  // O Interceptor já adiciona o token automaticamente
}
```
Modifique o auth.interceptor.ts para não adicionar o token em requisições para "login" e "cadastro":

```ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler } from '@angular/common/http';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const token = localStorage.getItem('token');

    // URLs que NÃO precisam de token (ajuste conforme necessário)
    if (req.url.includes('/auth/login') || req.url.includes('/auth/cadastro')) {
      return next.handle(req);  // 🔥 Deixa a requisição seguir sem modificar headers
    }

    if (token) {
      const clonedReq = req.clone({
        headers: req.headers.set('Authorization', `Bearer ${token}`)
      });
      return next.handle(clonedReq);
    }

    return next.handle(req);
  }
}
```
O ideal é armazenar o token no localStorage para que ele permaneça salvo mesmo ao recarregar a página

```ts
localStorage.setItem('token', tokenRecebido);
```
Se quiser que o token seja apagado ao fechar a aba, use sessionStorage:
```ts
sessionStorage.setItem('token', tokenRecebido);
```
Crie um método no serviço de autenticação (auth.service.ts) para checar se o token ainda está válido
```ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root',
})
export class AuthService {
  isAuthenticated(): boolean {
    const token = localStorage.getItem('token');
    return !!token; // Retorna true se o token existir, senão false
  }

  logout(): void {
    localStorage.removeItem('token');
  }
}
```
Agora, no AppComponent, podemos redirecionar para o login se o usuário não estiver autenticado:

```ts
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from './services/auth.service';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
})
export class AppComponent implements OnInit {
  constructor(private authService: AuthService, private router: Router) {}

  ngOnInit() {
    if (!this.authService.isAuthenticated()) {
      this.router.navigate(['/login']); // Redireciona para o login se não estiver autenticado
    }
  }
}
```
ng generate guard guards/auth
```sh
ng generate guard guards/auth
```
Edite auth.guard.ts:

```ts
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

@Injectable({
  providedIn: 'root',
})
export class AuthGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}

  canActivate(): boolean {
    if (!this.authService.isAuthenticated()) {
      this.router.navigate(['/login']);
      return false;
    }
    return true;
  }
}
```

Agora, aplique o AuthGuard nas rotas protegidas (app-routing.module.ts):

```ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { LoginComponent } from './components/login/login.component';
import { CadastroComponent } from './components/cadastro/cadastro.component';
import { PesquisaComponent } from './components/pesquisa/pesquisa.component';
import { AuthGuard } from './guards/auth.guard'; // Importação do AuthGuard

const routes: Routes = [
  { path: 'login', component: LoginComponent }, // Login é público
  { path: 'cadastro', component: CadastroComponent }, // Cadastro é público
  { path: 'pesquisa', component: PesquisaComponent, canActivate: [AuthGuard] }, // Protegido pelo AuthGuard
  { path: '**', redirectTo: 'login' }, // Redireciona qualquer rota inválida para o login
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}

```

Configuring application environments

=> link: https://v17.angular.io/guide/build

=> https://github.com/IgorTudisco/Eco-merce/blob/main/frontEnd/src/environments/environment.prod.ts

=> https://github.com/IgorTudisco/Eco-merce/blob/main/frontEnd/src/app/service/auth.service.ts

=> https://www.here.com/platform/map-data?cid=Developer_Location-Services-Api-Google-YT-0-Dev-LATAM-BR-matchtype=p&utm_source=Google&utm_medium=ppc&utm_campaign=Products_AlwaysOn&utm_term=real%20time%20location%20api&gad_source=1&gclid=Cj0KCQjw1um-BhDtARIsABjU5x6HBWL78Sas2fM5eBUUw05XSXvc24auTFzTe1etq7_VRS6kq-HVmxgaAhzfEALw_wcB&gclsrc=aw.ds

=> https://storepoint.co/developers?utm_source=google&utm_medium=cpc&utm_campaign=18882060419&utm_adgroup=176902485072&utm_term=location%20management%20api&utm_content=738600204230&gad_source=1&gclid=Cj0KCQjw1um-BhDtARIsABjU5x5Nt_i3gWU3zLFwNP5-4AiVI8SyBkLOdu09HEboHxJwu9LKfhRSOg4aAr22EALw_wcB

=> https://opencagedata.com/api
Google Maps Geolocation + Places API (Gratuita até certo limited)

```sql
SELECT *, 
       ST_DistanceSphere(
           ST_MakePoint(longitude, latitude),
           ST_MakePoint(-23.5505, -46.6333)
       ) AS distancia
FROM usuarios
WHERE ST_DistanceSphere(
          ST_MakePoint(longitude, latitude),
          ST_MakePoint(-23.5505, -46.6333)
      ) <= 2500;  -- 2,5 km em metros
```

--------------------------------------------------------------------------------------
--------------------------------------------------------------------------------------


Converter Endereço para Coordenadas (Latitude/Longitude)
Para transformar um endereço em coordenadas, você pode usar uma API de geocodificação, como o OpenCage Geocoder (grátis até 2.500 requisições/dia).

📌 Exemplo de requisição:

bash
Copy
Edit
https://api.opencagedata.com/geocode/v1/json?q=Rua+Augusta+1500,São+Paulo&key=SUA_CHAVE
📌 Exemplo de resposta:

json
Copy
Edit
{
    "results": [
        {
            "geometry": {
                "lat": -23.5565,
                "lng": -46.6581
            }
        }
    ]
}
📌 Exemplo de código em C# para buscar coordenadas a partir do endereço:

csharp
Copy
Edit
using System.Net.Http;
using System.Threading.Tasks;
using Newtonsoft.Json.Linq;

public class GeolocationService
{
    private readonly HttpClient _httpClient;
    private const string API_KEY = "SUA_CHAVE";

    public GeolocationService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<(double Latitude, double Longitude)> ObterCoordenadasAsync(string endereco)
    {
        string url = $"https://api.opencagedata.com/geocode/v1/json?q={Uri.EscapeDataString(endereco)}&key={API_KEY}";
        var response = await _httpClient.GetStringAsync(url);
        var json = JObject.Parse(response);

        double latitude = json["results"][0]["geometry"]["lat"].Value<double>();
        double longitude = json["results"][0]["geometry"]["lng"].Value<double>();

        return (latitude, longitude);
    }
}
📌 Como chamar o método para pegar as coordenadas do endereço do usuário:

csharp
Copy
Edit
var service = new GeolocationService(new HttpClient());
var coordenadas = await service.ObterCoordenadasAsync("Rua Augusta 1500, São Paulo");
Console.WriteLine($"Latitude: {coordenadas.Latitude}, Longitude: {coordenadas.Longitude}");
2️⃣ Armazenar as Coordenadas no Banco de Dados
Agora que você pegou a latitude e longitude do endereço, é importante armazenar essas informações no banco de dados.

📌 Exemplo de tabela no PostgreSQL com PostGIS para armazenar usuários e coordenadas:

sql
Copy
Edit
CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(255) NOT NULL,
    endereco VARCHAR(255) NOT NULL,
    latitude DECIMAL(9,6),
    longitude DECIMAL(9,6)
);
📌 Exemplo de como inserir um usuário com coordenadas:

sql
Copy
Edit
INSERT INTO usuarios (nome, endereco, latitude, longitude)
VALUES ('João da Silva', 'Rua Augusta 1500, São Paulo', -23.5565, -46.6581);
3️⃣ Buscar Pessoas Próximas dentro de um Raio de X km
Agora que cada usuário tem sua localização armazenada, podemos fazer uma consulta SQL para buscar pessoas próximas.

📌 Exemplo de SQL para encontrar usuários dentro de um raio de 2,5 km:

sql
Copy
Edit
SELECT id, nome, endereco, latitude, longitude,
       ST_DistanceSphere(
           ST_MakePoint(longitude, latitude),
           ST_MakePoint(-23.5565, -46.6581)  -- Coordenadas do usuário de referência
       ) AS distancia
FROM usuarios
WHERE ST_DistanceSphere(
          ST_MakePoint(longitude, latitude),
          ST_MakePoint(-23.5565, -46.6581)
      ) <= 2500;  -- 2,5 km em metros
📌 Como funciona?

ST_MakePoint(longitude, latitude): Cria um ponto geoespacial para o usuário
ST_DistanceSphere(): Calcula a distância entre dois pontos
WHERE ... <= 2500: Filtra apenas os usuários dentro do raio de 2,5 km
4️⃣ Criando um Endpoint na API para Buscar Pessoas Próximas
Agora, vamos criar um endpoint na API .NET 9 para buscar usuários dentro do raio de 2,5 km.

📌 Controller em C# para buscar usuários próximos:

csharp
Copy
Edit
[HttpGet("usuarios-proximos")]
public async Task<IActionResult> BuscarUsuariosProximos(double latitude, double longitude, double raio = 2500)
{
    var query = @"
        SELECT id, nome, endereco, latitude, longitude,
               ST_DistanceSphere(
                   ST_MakePoint(longitude, latitude),
                   ST_MakePoint(@longitude, @latitude)
               ) AS distancia
        FROM usuarios
        WHERE ST_DistanceSphere(
                  ST_MakePoint(longitude, latitude),
                  ST_MakePoint(@longitude, @latitude)
              ) <= @raio;";

    using (var connection = new NpgsqlConnection(_connectionString))
    {
        var usuarios = await connection.QueryAsync<Usuario>(query, new { latitude, longitude, raio });
        return Ok(usuarios);
    }
}
📌 Como chamar a API pelo Angular?

typescript
Copy
Edit
pesquisarUsuariosProximos(latitude: number, longitude: number): Observable<any> {
  return this.http.get(`http://localhost:5150/api/usuarios-proximos?latitude=${latitude}&longitude=${longitude}&raio=2500`);
}
5️⃣ Criando o Frontend no Angular
Agora, vamos criar um botão para pesquisar usuários próximos no Angular.

📌 Componente de Pesquisa (pesquisa.component.ts)

typescript
Copy
Edit
import { Component } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-pesquisa',
  templateUrl: './pesquisa.component.html'
})
export class PesquisaComponent {
  usuariosProximos: any[] = [];

  constructor(private http: HttpClient) {}

  buscarUsuariosProximos() {
    navigator.geolocation.getCurrentPosition((position) => {
      const latitude = position.coords.latitude;
      const longitude = position.coords.longitude;

      this.http.get(`http://localhost:5150/api/usuarios-proximos?latitude=${latitude}&longitude=${longitude}&raio=2500`)
        .subscribe((data: any) => {
          this.usuariosProximos = data;
        });
    });
  }
}
📌 Componente de Pesquisa (pesquisa.component.html)

html
Copy
Edit
<button (click)="buscarUsuariosProximos()" class="btn btn-primary">Buscar Usuários Próximos</button>

<ul *ngIf="usuariosProximos.length > 0">
  <li *ngFor="let usuario of usuariosProximos">
    {{ usuario.nome }} - {{ usuario.endereco }}
  </li>
</ul>
🔥 Conclusão
Agora você tem um sistema completo que:
✅ Obtém a localização do usuário a partir do endereço
✅ Armazena latitude/longitude no banco de dados
✅ Consulta pessoas próximas dentro de um raio de 2,5 km
✅ Cria um endpoint na API para retornar usuários próximos
✅ Faz o consumo da API no frontend Angular

🚀 Agora você pode expandir isso para exibir os usuários no Google Maps ou outro serviço de mapas!

Se precisar de ajustes ou quiser melhorar a interface, me avise! 💪🔥



