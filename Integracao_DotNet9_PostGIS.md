# Integração do .NET 9 com PostGIS usando Npgsql e NetTopologySuite

## 1. Introdução
Este guia explica como configurar um projeto **.NET 9** para armazenar e consultar coordenadas geográficas em um banco **PostgreSQL** com **PostGIS**. Usaremos o pacote **Npgsql.EntityFrameworkCore.PostgreSQL.NetTopologySuite** para lidar com tipos espaciais.

## 2. Pré-requisitos
- **.NET 9** instalado
- **PostgreSQL 15+** com **PostGIS** habilitado
- **Docker** (opcional, mas recomendado)
- **Entity Framework Core**

## 3. Instalação dos pacotes necessários
Execute o comando abaixo para instalar o pacote que permite a integração com PostGIS:

```sh
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL.NetTopologySuite
```

## 4. Configuração do `DbContext`
Edite o seu `DbContext` para usar o **NetTopologySuite**:

```csharp
using Microsoft.EntityFrameworkCore;
using NetTopologySuite;
using NetTopologySuite.Geometries;

public class MeuDbContext : DbContext
{
    public DbSet<ClienteLocalizacao> ClienteLocalizacoes { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseNpgsql("Host=localhost;Database=meubanco;Username=usuario;Password=senha",
            o => o.UseNetTopologySuite());
    }
}
```

## 5. Criando a entidade `ClienteLocalizacao`
Esta entidade armazena informações sobre o cliente, incluindo **latitude** e **longitude**:

```csharp
using NetTopologySuite.Geometries;

public class ClienteLocalizacao
{
    public int Id { get; set; }
    public string Nome { get; set; }
    public string Endereco { get; set; }
    
    // Armazena coordenadas geográficas
    public Point Localizacao { get; set; }
}
```

## 6. Criando a Migration
Gere e aplique a migration para criar a tabela no PostgreSQL:

```sh
dotnet ef migrations add AddClienteLocalizacao
dotnet ef database update
```

Isso criará a seguinte estrutura no banco:

```sql
CREATE TABLE "ClienteLocalizacoes" (
    "Id" SERIAL PRIMARY KEY,
    "Nome" TEXT NOT NULL,
    "Endereco" TEXT NOT NULL,
    "Localizacao" GEOMETRY(Point, 4326) NOT NULL
);
```

## 7. Inserindo um cliente com coordenadas
Para armazenar um cliente no banco com coordenadas:

```csharp
using NetTopologySuite.Geometries;

var geometryFactory = NtsGeometryServices.Instance.CreateGeometryFactory(srid: 4326);
var novaLocalizacao = new ClienteLocalizacao
{
    Nome = "João Silva",
    Endereco = "Rua ABC, 123",
    Localizacao = geometryFactory.CreatePoint(new Coordinate(-46.6333, -23.5505)) // (longitude, latitude)
};

using (var context = new MeuDbContext())
{
    context.ClienteLocalizacoes.Add(novaLocalizacao);
    context.SaveChanges();
}
```

## 8. Buscando clientes próximos (Raio de 2,5 km)
Para encontrar clientes dentro de um raio de **2,5 km**, usamos `ST_DWithin`:

```csharp
using NetTopologySuite.Geometries;

var latitude = -23.5505;
var longitude = -46.6333;
var raioKm = 2.5;

// Criar ponto de referência
var geometryFactory = NtsGeometryServices.Instance.CreateGeometryFactory(srid: 4326);
var pontoReferencia = geometryFactory.CreatePoint(new Coordinate(longitude, latitude));

using (var context = new MeuDbContext())
{
    var clientesProximos = context.ClienteLocalizacoes
        .Where(c => c.Localizacao.IsWithinDistance(pontoReferencia, raioKm * 1000)) // Convertendo km para metros
        .ToList();

    foreach (var cliente in clientesProximos)
    {
        Console.WriteLine($"Cliente próximo: {cliente.Nome}");
    }
}
```

## 9. Conclusão
Este guia mostrou como configurar o **.NET 9** com **PostGIS** para armazenar e consultar coordenadas geográficas. Agora, sua aplicação pode lidar com localização geoespacial de maneira eficiente! 🚀
