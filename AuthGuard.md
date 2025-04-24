# ✅ Adaptação do AuthGuard para SSR (Server-Side Rendering) no Angular

## 🧠 Objetivo
Evitar erros ao usar `localStorage` diretamente no `AuthGuard`, pois no ambiente SSR (Server-Side Rendering) o código é executado no servidor, onde não existem objetos como `window`, `document` ou `localStorage`.

---

## 📦 Etapa 1: Criar o AuthService

O `AuthService` encapsula a lógica de autenticação e garante que o código só interage com o `localStorage` quando está no browser.

```ts
// auth.service.ts
import { Injectable, Inject, PLATFORM_ID } from '@angular/core';
import { isPlatformBrowser } from '@angular/common';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private isBrowser: boolean;

  constructor(@Inject(PLATFORM_ID) platformId: any) {
    this.isBrowser = isPlatformBrowser(platformId);
  }

  isLoggedIn(): boolean {
    if (!this.isBrowser) return false; // SSR: evita erro
    const token = localStorage.getItem('auth_token');
    return !!token;
  }

  getToken(): string | null {
    if (!this.isBrowser) return null;
    return localStorage.getItem('auth_token');
  }
}
```

---

## 🛠️ Etapa 2: Adaptar o AuthGuard

O `AuthGuard` é responsável por proteger rotas específicas, garantindo que apenas usuários autenticados possam acessá-las. Ele utiliza o `AuthService` para verificar se o usuário está autenticado.

### Passos para adaptar o AuthGuard:

1. **Importe as dependências necessárias**:
   O `AuthGuard` precisa implementar a interface `CanActivate` do Angular Router. Além disso, ele utiliza o `Router` para redirecionar usuários não autenticados.

2. **Injete o `AuthService` e o `Router` no construtor**:
   O `AuthService` verifica o estado de autenticação, enquanto o `Router` é usado para redirecionar o usuário para a página de login, caso necessário.

3. **Implemente o método `canActivate`**:
   Este método é chamado automaticamente pelo Angular Router antes de ativar uma rota protegida. Ele deve retornar:
   - `true` se o usuário estiver autenticado.
   - Um `UrlTree` (gerado pelo `Router`) para redirecionar o usuário, caso contrário.

### Exemplo de implementação:

```ts
// auth.guard.ts
import { Injectable } from '@angular/core';
import {
  CanActivate,
  Router,
  UrlTree,
  ActivatedRouteSnapshot,
  RouterStateSnapshot
} from '@angular/router';
import { AuthService } from './auth.service';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean | UrlTree {
    // Verifica se o usuário está autenticado
    if (this.authService.isLoggedIn()) {
      return true; // Permite o acesso à rota
    }

    // Redireciona para a página de login
    return this.router.createUrlTree(['/login']);
  }
}
```

---

## 🌐 Etapa 3: Configurar as rotas com o AuthGuard

Agora que o `AuthGuard` está implementado, você pode usá-lo para proteger rotas específicas no arquivo de configuração de rotas (`app-routing.module.ts`).

### Exemplo de configuração:

```ts
// app-routing.module.ts
{
  path: 'dashboard', // Rota protegida
  canActivate: [AuthGuard], // Aplica o AuthGuard
  loadChildren: () => import('./dashboard/dashboard.module').then(m => m.DashboardModule)
}
```

### O que acontece aqui:
- Quando o usuário tenta acessar a rota `/dashboard`, o Angular Router chama o método `canActivate` do `AuthGuard`.
- Se o método retornar `true`, a rota é ativada.
- Caso contrário, o usuário é redirecionado para `/login`.

---

## 🧪 Etapa 4: Teste com SSR

Execute o projeto com SSR (Server-Side Rendering) para garantir que o `AuthGuard` funcione corretamente, mesmo durante a renderização no servidor.

```bash
npm run dev:ssr
```

### Verifique:
- Acesse uma rota protegida.
- Confirme que não há erros relacionados ao `localStorage` no console do servidor.

---

## ✅ Resultado Esperado

- O `AuthGuard` protege as rotas corretamente, mesmo durante a renderização no servidor.
- A lógica de autenticação está centralizada no `AuthService`, tornando o código mais limpo e testável.
- O projeto está preparado para futuras melhorias, como:
  - Uso de cookies com `HttpOnly`.
  - Tokens JWT.
  - Expiração automática de tokens.

---

## 🧩 Extras (futuros)

- Armazenar o token em cookies (`HttpOnly`) para maior segurança no SSR.
- Adicionar lógica de expiração do token.
- Implementar decodificação de JWT no `AuthService`.

