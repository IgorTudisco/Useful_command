# 📌 Guia Completo: Implementação do Hangfire no .NET 9 com PostgreSQL

## ✅ O que é o Hangfire?
O **Hangfire** é uma biblioteca para **agendamento de tarefas em background**, permitindo que você execute **tarefas recorrentes, agendadas ou em fila** sem bloquear a aplicação.

No nosso caso, vamos usá-lo para **atualizar periodicamente a tabela ClienteLocalizacoes** com as coordenadas de latitude e longitude dos clientes.

---

# 🛠️ Passo a passo para configurar o Hangfire

## 🏗️ 1. Instalar o Hangfire e o Provider do PostgreSQL
Execute no terminal:

```sh
dotnet add package Hangfire
dotnet add package Hangfire.PostgreSql
```

Isso adiciona o Hangfire e a integração com PostgreSQL ao projeto.

---

## 📄 2. Configurar a conexão no `appsettings.json`
Adicione a string de conexão:

```json
{
  "ConnectionStrings": {
    "PostgresDb": "Host=localhost;Port=5432;Database=MeuBanco;Username=meuuser;Password=minhasenha"
  },
  "Hangfire": {
    "Schema": "hangfire"
  }
}
```

---

## ⚙️ 3. Configurar o Hangfire no `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);
var connectionString = builder.Configuration.GetConnectionString("PostgresDb");

builder.Services.AddHangfire(config => config.UsePostgreSqlStorage(connectionString));
builder.Services.AddHangfireServer();

var app = builder.Build();

app.UseHangfireDashboard();

RecurringJob.AddOrUpdate(
    "atualizar-coordenadas-clientes",
    () => AtualizarLocalizacoesClientes(),
    "0 0 1 */4 *" // A cada 4 meses
);

app.Run();

void AtualizarLocalizacoesClientes()
{
    Console.WriteLine($"[{DateTime.Now}] Atualizando coordenadas dos clientes...");
}
```

---

## 🗃️ 4. Como o Hangfire armazena as tarefas?
O Hangfire cria automaticamente tabelas no PostgreSQL:

| 🗃 Tabela | 🔍 O que armazena? |
|----------|----------------|
| `hangfire.job` | Lista de jobs agendados |
| `hangfire.jobqueue` | Jobs na fila |
| `hangfire.server` | Servidores ativos |
| `hangfire.state` | Estado do job |

Se quiser criar o schema manualmente:

```sql
CREATE SCHEMA hangfire;
```

---

## 🔄 5. Criar o serviço para atualizar a tabela ClienteLocalizacoes

```csharp
public class ClienteService
{
    private readonly MeuDbContext _context;

    public ClienteService(MeuDbContext context)
    {
        _context = context;
    }

    public void AtualizarCoordenadas()
    {
        var clientesSemCoordenadas = _context.Clientes
            .Where(c => !_context.ClienteLocalizacoes.Any(cl => cl.ClienteId == c.Id))
            .ToList();

        foreach (var cliente in clientesSemCoordenadas)
        {
            var coordenadas = ObterCoordenadas(cliente.EnderecoCompleto);
            if (coordenadas != null)
            {
                _context.ClienteLocalizacoes.Add(new ClienteLocalizacao
                {
                    ClienteId = cliente.Id,
                    Latitude = coordenadas.Latitude,
                    Longitude = coordenadas.Longitude
                });
            }
        }

        _context.SaveChanges();
    }

    private Coordenadas? ObterCoordenadas(string endereco)
    {
        return new Coordenadas( /* latitude */, /* longitude */ );
    }
}
```

---

## 🔗 6. Chamando o serviço no Hangfire

```csharp
using (var scope = app.Services.CreateScope())
{
    var clienteService = scope.ServiceProvider.GetRequiredService<ClienteService>();
    RecurringJob.AddOrUpdate(
        "atualizar-coordenadas-clientes",
        () => clienteService.AtualizarCoordenadas(),
        "0 0 1 */4 *"
    );
}
```

Agora o Hangfire chamará `AtualizarCoordenadas()` a cada **4 meses**.

---

## 📊 7. Acessando o Painel do Hangfire

1️⃣ **Suba a aplicação:**  
```sh
dotnet run
```
2️⃣ **Acesse no navegador:**  
```
http://localhost:5150/hangfire
```
3️⃣ **Gerencie os jobs pela interface web.**

---

## 🎯 Recapitulando
✅ Instalamos o Hangfire e configuramos o banco PostgreSQL.  
✅ Criamos a conexão no **Program.cs**.  
✅ Criamos um **serviço para atualizar as coordenadas dos clientes**.  
✅ Agendamos um job **recorrente a cada 4 meses**.  
✅ Testamos tudo acessando o **Painel do Hangfire**.  

---

## 🚀 Conclusão
Agora, o Hangfire manterá **automaticamente** a tabela `ClienteLocalizacoes` atualizada.

Para testar manualmente:

```csharp
BackgroundJob.Enqueue(() => clienteService.AtualizarCoordenadas());
```

---

### 📝 Próximos passos
🔹 Testar a API Pelias ou Nominatim para obter coordenadas.  
🔹 Configurar logs para monitoramento.  
🔹 Implementar notificações em caso de falha.

Agora você tem uma **solução completa e automatizada** para atualizar a localização dos clientes! 🚀🔥
