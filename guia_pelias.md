# 🗺️ Guia Completo: Configuração do Pelias, Banco de Dados e API .NET 9  

## 🎯 Objetivo  
1. **Converter endereços em coordenadas (latitude e longitude) usando Pelias**  
2. **Armazenar essas coordenadas no PostgreSQL + PostGIS**  
3. **Criar uma API .NET 9 para buscar clientes próximos dentro de um raio específico**  

---

## 🛠️ 1. Requisitos  

Antes de começar, instale as seguintes ferramentas:  
- ✅ **Docker** e **Docker Compose**  
- ✅ **Node.js** (versão recomendada pelo Pelias)  
- ✅ **Git**  
- ✅ **PostgreSQL** com **PostGIS**  
- ✅ **.NET 9**  

---

## 📥 2. Clonando e Configurando o Pelias  

```bash
git clone https://github.com/pelias/docker.git pelias
cd pelias
```

Agora, configuramos o **Elasticsearch**:  

```bash
mkdir data
docker compose up elasticsearch
```

Verifique se está rodando:  

```bash
curl "http://localhost:9200/_cat/health?v"
```

---

## 🌍 3. Baixando e Processando Dados Geográficos  

```bash
docker compose run whosonfirst
docker compose run openstreetmap
```

Indexando os dados no Elasticsearch:  

```bash
docker compose run placeholder
docker compose run interpolation
docker compose run polylines
docker compose run openaddresses
```

Se precisar reindexar:  

```bash
docker compose run elasticsearch index
```

---

## 🚀 4. Iniciando a API do Pelias  

```bash
docker compose up api
```

Agora a API estará disponível em `http://localhost:4000`.  

**Teste a API:**  

```bash
curl "http://localhost:4000/v1/search?text=Rua+X,Cidade+Y,Numero+Tal"
```

---

## 🗄️ 5. Configuração do Banco de Dados PostgreSQL + PostGIS  

### **5.1 Criando o Banco de Dados**  

```sql
CREATE DATABASE geolocalizacao;
```

Ativando o PostGIS:  

```sql
CREATE EXTENSION postgis;
```

### **5.2 Criando a Tabela de Localizações**  

```sql
CREATE TABLE cliente_localizacao (
    id SERIAL PRIMARY KEY,
    cliente_id INT NOT NULL,
    nome VARCHAR(255) NOT NULL,
    endereco TEXT NOT NULL,
    latitude FLOAT,
    longitude FLOAT,
    geom GEOMETRY(Point, 4326),
    FOREIGN KEY (cliente_id) REFERENCES cliente(id) ON DELETE CASCADE
);
```

---

## 🔄 6. Convertendo Endereços para Coordenadas  

Crie o arquivo `geocode.sh`:  

```bash
#!/bin/bash
DB_HOST="localhost"
DB_USER="seu_usuario"
DB_NAME="geolocalizacao"

psql -h $DB_HOST -U $DB_USER -d $DB_NAME -t -c "
SELECT cliente_id, nome, endereco FROM cliente_localizacao 
WHERE latitude IS NULL OR longitude IS NULL" | while read -r cliente_id nome endereco; do

  response=$(curl -s "http://localhost:4000/v1/search?text=$endereco")
  
  lat=$(echo $response | jq '.features[0].geometry.coordinates[1]')
  lon=$(echo $response | jq '.features[0].geometry.coordinates[0]')

  psql -h $DB_HOST -U $DB_USER -d $DB_NAME -c "
    UPDATE cliente_localizacao 
    SET latitude=$lat, longitude=$lon, geom=ST_SetSRID(ST_MakePoint($lon, $lat), 4326) 
    WHERE cliente_id=$cliente_id;
  "
done
```

Executando o script:  

```bash
chmod +x geocode.sh
./geocode.sh
```

---

## 🌐 7. Criando a API Backend em .NET 9  

### **7.1 Criando o Projeto**  

```bash
dotnet new webapi -o GeoApi
cd GeoApi
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add package NetTopologySuite
```

---

### **7.2 Configurando o Contexto do Banco**  

Arquivo `AppDbContext.cs`:  

```csharp
using Microsoft.EntityFrameworkCore;
using NetTopologySuite.Geometries;

public class AppDbContext : DbContext
{
    public DbSet<ClienteLocalizacao> ClienteLocalizacoes { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseNpgsql("Host=localhost;Database=geolocalizacao;Username=seu_usuario;Password=sua_senha", o => o.UseNetTopologySuite());
    }
}
```

---

### **7.3 Criando o Modelo**  

```csharp
public class ClienteLocalizacao
{
    public int Id { get; set; }
    public int ClienteId { get; set; }
    public string Nome { get; set; }
    public string Endereco { get; set; }
    public double Latitude { get; set; }
    public double Longitude { get; set; }
    public Point Geom { get; set; }
}
```

---

### **7.4 Criando o Controller**  

```csharp
[Route("api/[controller]")]
[ApiController]
public class GeolocalizacaoController : ControllerBase
{
    private readonly AppDbContext _context;

    public GeolocalizacaoController(AppDbContext context)
    {
        _context = context;
    }

    [HttpGet("proximos")]
    public IActionResult GetClientesProximos(double latitude, double longitude, double raio = 2500)
    {
        var clientes = _context.ClienteLocalizacoes
            .Where(c => c.Geom.IsWithinDistance(new Point(longitude, latitude) { SRID = 4326 }, raio))
            .ToList();

        return Ok(clientes);
    }
}
```

---

## ✅ Conclusão  

Agora, sua aplicação consegue:  
✔️ **Converter endereços em coordenadas automaticamente**  
✔️ **Armazenar as coordenadas no PostgreSQL + PostGIS**  
✔️ **Buscar clientes próximos com a API em .NET 9**  

Basta integrar essa API ao **Angular 19** e exibir os resultados no frontend! 🚀
