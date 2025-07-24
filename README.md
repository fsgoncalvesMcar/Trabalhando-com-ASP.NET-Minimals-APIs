using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

// Banco de dados com EF Core em memória
builder.Services.AddDbContext<VeiculoDbContext>(options =>
    options.UseInMemoryDatabase("VeiculosDb"));

// Autenticação JWT simulada para testes
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new()
        {
            ValidateIssuer = false,
            ValidateAudience = false,
            ValidateLifetime = false,
            ValidateIssuerSigningKey = false
        };
    });

// Configuração do Swagger com suporte a JWT
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(c =>
{
    c.SwaggerDoc("v1", new OpenApiInfo { Title = "VeiculoAPI", Version = "v1" });
    c.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT no formato: Bearer {token}",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey
    });
    c.AddSecurityRequirement(new OpenApiSecurityRequirement {
        {
            new OpenApiSecurityScheme {
                Reference = new OpenApiReference {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            new List<string>()
        }
    });
});

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();
app.UseSwagger();
app.UseSwaggerUI();

// Endpoints
app.MapGet("/", () => "API de veículos em operação");

app.MapGet("/veiculos/teste", () => "Endpoint público");

app.MapPost("/admin/veiculos", [Microsoft.AspNetCore.Authorization.Authorize(Roles = "admin")] 
    (Veiculo veiculo, VeiculoDbContext db) =>
{
    db.Veiculos.Add(veiculo);
    db.SaveChanges();
    return Results.Ok($"{veiculo.Modelo} cadastrado com sucesso.");
});

app.Run();

// Modelo
public record Veiculo(int Id, string Modelo, string Placa, int Ano);

// DbContext
public class VeiculoDbContext : DbContext
{
    public VeiculoDbContext(DbContextOptions options) : base(options) {}
    public DbSet<Veiculo> Veiculos => Set<Veiculo>();
}
using System.Net;
using System.Net.Http.Headers;
using System.Text;
using System.Threading.Tasks;
using Xunit;
using Microsoft.AspNetCore.Mvc.Testing;

public class VeiculoApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public VeiculoApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task DeveRegistrarVeiculoComTokenFalso()
    {
        var veiculoJson = "{\"id\":1,\"modelo\":\"Fusca\",\"placa\":\"AAA1234\",\"ano\":1985}";

        _client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", "faketoken");

        var response = await _client.PostAsync("/admin/veiculos", new StringContent(veiculoJson, Encoding.UTF8, "application/json"));

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }

    [Fact]
    public async Task EndpointPublico_DeveResponder()
    {
        var response = await _client.GetAsync("/veiculos/teste");
        var content = await response.Content.ReadAsStringAsync();

        Assert.Contains("público", content);
    }
}
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:7.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "VeiculoApi.dll"]

name: Build and Test API

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '7.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release

      - name: Test
        run: dotnet test --no-build --verbosity normal
