# Useful_command
Código C#

Criar um Novo Projeto Web API no .NET 9

dotnet new webapi -n BiroAPI
cd BiroAPI
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package Swashbuckle.AspNetCore
dotnet add package Microsoft.AspNetCore.Cors


{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=SeuBanco;Username=SeuUsuario;Password=SuaSenha"
  },
  "AllowedHosts": "*"
}

 Gerar o código automaticamente usando Scaffold-DbContext
 dotnet ef dbcontext scaffold "Host=SEU_HOST;Database=SEU_BANCO;Username=SEU_USUARIO;Password=SUA_SENHA" Npgsql.EntityFrameworkCore.PostgreSQL -o Models --context-dir Data --force

  O que esse comando faz?

Gera automaticamente as classes das tabelas já existentes.
Cria o DbContext correspondente ao banco de dados.
Salva os arquivos na pasta Models e o DbContext na pasta Data.
--force substitui arquivos existentes (caso precise atualizar).


dotnet tool install --global dotnet-ef
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
