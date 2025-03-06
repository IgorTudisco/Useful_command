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
dotnet ef dbcontext scaffold "Host=SEU_HOST;Database=SEU_BANCO;Username=SEU_USUARIO;Password=SUA_SENHA" Npgsql.EntityFrameworkCore.PostgreSQL -o Models --context-dir Data --force

Explicação do comando:

scaffold → Gera as classes do banco automaticamente.
Host=SEU_HOST;Database=SEU_BANCO;Username=SEU_USUARIO;Password=SUA_SENHA → String de conexão com o banco PostgreSQL.
Npgsql.EntityFrameworkCore.PostgreSQL → Provedor do PostgreSQL.
-o Models → Salva as entidades na pasta Models/.
--context-dir Data → Salva o DbContext na pasta Data/.

