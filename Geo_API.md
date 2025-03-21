Para atender aos seus requisitos de geolocalização utilizando soluções open source, aqui estão algumas recomendações:

Geohashing: Uma técnica que converte coordenadas de latitude e longitude em strings alfanuméricas, permitindo agrupar locais próximos com prefixos semelhantes. Isso facilita a busca por usuários em áreas próximas sem a necessidade de cálculos complexos. ​
Reddit

OpenStreetMap (OSM): Um projeto colaborativo que fornece dados geográficos gratuitos e abertos. Você pode utilizar APIs como a Nominatim para geocodificação (converter endereços em coordenadas) e serviços de roteamento para calcular distâncias entre pontos.​

PostGIS: Uma extensão do PostgreSQL que adiciona suporte a objetos geográficos, permitindo armazenar e consultar dados espaciais de forma eficiente. Com PostGIS, é possível realizar consultas para encontrar usuários dentro de um raio específico a partir de uma localização.​

Leaflet: Uma biblioteca JavaScript open source para mapas interativos, que pode ser integrada ao seu aplicativo para visualizar dados geoespaciais e destacar usuários próximos em um mapa.​

Pelias: Um mecanismo de geocodificação open source que utiliza dados do OpenStreetMap e outras fontes, permitindo converter endereços em coordenadas e vice-versa.​

Ao combinar essas ferramentas, você pode desenvolver uma solução completa de geolocalização open source que atenda às suas necessidades de identificar a localização de usuários e encontrar outros usuários próximos dentro de um raio especificado.

-----------------------------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------------------------------

🔹 1️⃣ Configurando PostgreSQL com PostGIS
📌 PostGIS é essencial para trabalhar com geolocalização.

1.1 Instalar o PostgreSQL com PostGIS
Se ainda não tiver instalado, faça isso no Docker:

yaml
Copy code
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
Ou instale manualmente no Ubuntu:

sh
Copy code
sudo apt update
sudo apt install -y postgresql postgis postgresql-contrib
1.2 Criar a Tabela no PostgreSQL
Agora, crie a tabela que vai armazenar os clientes com coordenadas geográficas:

sql
Copy code
CREATE TABLE clientes (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(100),
    email VARCHAR(150) UNIQUE,
    endereco TEXT NOT NULL,
    latitude DOUBLE PRECISION,
    longitude DOUBLE PRECISION,
    localizacao GEOGRAPHY(Point, 4326)
);
📌 Explicação:
✔️ latitude e longitude armazenam as coordenadas.
✔️ localizacao usa o tipo GEOGRAPHY(Point, 4326) para cálculos geoespaciais.

🔹 2️⃣ Convertendo Endereços em Coordenadas com OpenCage
O OpenCage Geocoder permite até 2.500 requisições/dia grátis. Se precisar de 800k/mês, um plano pago será necessário.

📌 Crie uma conta gratuita e obtenha uma chave de API:
🔗 https://opencagedata.com/api

2.1 Criar o Serviço GeolocationService.cs no Backend
csharp
Copy code
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
        public class Result
        {
            public Geometry geometry { get; set; }
        }
        public class Geometry
        {
            public double lat { get; set; }
            public double lng { get; set; }
        }
    }
}
📌 Explicação:
✔️ Consulta o OpenCage e retorna latitude e longitude.
✔️ Parâmetro q → Endereço formatado corretamente.
✔️ Uso da chave de API para autenticação.

🔹 3️⃣ Criar Endpoint para Cadastrar Clientes
Agora, integramos o serviço na API para converter endereço automaticamente e salvar no PostgreSQL.

csharp
Copy code
[ApiController]
[Route("api/clientes")]
public class ClientesController : ControllerBase
{
    private readonly MeuDbContext _context;
    private readonly GeolocationService _geoService;

    public ClientesController(MeuDbContext context, GeolocationService geoService)
    {
        _context = context;
        _geoService = geoService;
    }

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
}
📌 Explicação:
✔️ Chama o OpenCage para converter endereço para coordenadas.
✔️ Salva no PostgreSQL já com GEOGRAPHY(Point, 4326).

🔹 4️⃣ Buscar Pessoas Próximas em um Raio de 2,5 km
Agora criamos um endpoint para buscar clientes próximos.

csharp
Copy code
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
📌 Explicação:
✔️ Busca usuários em um raio de 2,5 km.
✔️ Otimizado para grande volume de dados com PostGIS.

🔹 5️⃣ Criar Serviço no Angular
No clientes.service.ts:

typescript
Copy code
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
