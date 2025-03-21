# 🚀 Integração de Geolocalização com PostgreSQL + PostGIS + OpenCage + .NET 9 + Angular 19

Este guia detalhado explica como integrar **geolocalização** no seu sistema usando **PostgreSQL + PostGIS**, **OpenCage Geocoder**, **.NET 9** e **Angular 19**. 

---

## **🔹 1️⃣ Configurando PostgreSQL com PostGIS**  

### **1.1 Criar um Banco PostgreSQL com PostGIS no Docker**

```yaml
version: '3.1'
services:
  postgres:
    image: postgis/postgis:latest
    container_name: postgres-geoloc
    restart: always
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin
      POSTGRES_DB: geolocalizacao
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

---

### **1.2 Criar a Tabela para Clientes**  

```sql
CREATE TABLE clientes (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(100),
    email VARCHAR(150) UNIQUE,
    endereco TEXT NOT NULL,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    localizacao GEOGRAPHY(Point, 4326)
);
```

---

## **🔹 2️⃣ Convertendo Endereços em Coordenadas com OpenCage**  

📌 **Criar uma conta gratuita no OpenCage e obter uma chave de API:**  
🔗 [https://opencagedata.com/api](https://opencagedata.com/api)  

```csharp
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;

public class GeolocationService
{
    private readonly HttpClient _httpClient;
    private readonly string _apiKey = "SUA_CHAVE_OPEN_CAGE";

    public GeolocationService(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<(double latitude, double longitude)?> GetCoordinatesFromAddress(string endereco)
    {
        string url = $"https://api.opencagedata.com/geocode/v1/json?q={Uri.EscapeDataString(endereco)}&key={_apiKey}";

        var response = await _httpClient.GetStringAsync(url);
        var result = JsonSerializer.Deserialize<OpenCageResponse>(response);

        if (result?.results?.Count > 0)
        {
            return (result.results[0].geometry.lat, result.results[0].geometry.lng);
        }

        return null;
    }

    private class OpenCageResponse
    {
        public List<Result> results { get; set; }
        public class Result { public Geometry geometry { get; set; } }
        public class Geometry { public double lat { get; set; } public double lng { get; set; } }
    }
}
```

---

## **🔹 3️⃣ Criar Endpoint para Cadastrar Clientes**  

```csharp
[HttpPost]
public async Task<IActionResult> CadastrarCliente([FromBody] ClienteDTO clienteDto)
{
    var coordenadas = await _geoService.GetCoordinatesFromAddress(clienteDto.Endereco);
    if (coordenadas == null)
    {
        return BadRequest("Endereço inválido.");
    }

    var cliente = new Cliente
    {
        Nome = clienteDto.Nome,
        Email = clienteDto.Email,
        Endereco = clienteDto.Endereco,
        Latitude = coordenadas.Value.latitude,
        Longitude = coordenadas.Value.longitude,
        Localizacao = $"SRID=4326;POINT({coordenadas.Value.longitude} {coordenadas.Value.latitude})"
    };

    _context.Clientes.Add(cliente);
    await _context.SaveChangesAsync();

    return Ok(cliente);
}
```

---

## **🔹 4️⃣ Buscar Pessoas Próximas em um Raio de 2,5 km**  

```csharp
[HttpGet("proximos")]
public async Task<IActionResult> GetClientesProximos([FromQuery] double latitude, [FromQuery] double longitude)
{
    var clientes = await _context.Clientes
        .FromSqlRaw(@"SELECT * FROM clientes 
                      WHERE ST_DWithin(localizacao, 
                      ST_GeogFromText('SRID=4326;POINT({0} {1})'), 2500)", 
                      longitude, latitude)
        .ToListAsync();

    return Ok(clientes);
}
```

---

## **🔹 5️⃣ Criar Serviço no Angular**  

No `clientes.service.ts`:  

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class ClientesService {
  private apiUrl = 'http://localhost:5150/api/clientes';

  constructor(private http: HttpClient) {}

  cadastrarCliente(cliente: any): Observable<any> {
    return this.http.post(`${this.apiUrl}`, cliente);
  }

  getClientesProximos(latitude: number, longitude: number): Observable<any> {
    return this.http.get(`${this.apiUrl}/proximos?latitude=${latitude}&longitude=${longitude}`);
  }
}
```

---

# **📌 Conclusão**  

✅ **Banco PostgreSQL configurado com PostGIS** para cálculos geoespaciais.  
✅ **Conversão de endereços para coordenadas com OpenCage Geocoder**.  
✅ **API .NET 9 com endpoints para cadastro e busca de usuários próximos**.  
✅ **Frontend Angular 19 totalmente integrado**.  

🚀 **Agora seu sistema suporta conversão massiva e consultas eficientes de geolocalização!** 🚀
