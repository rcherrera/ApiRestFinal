# Cliente.md — Blazor WebAssembly para consumir el API REST de *TareaFinal*

> Objetivo: Crear un **cliente Blazor WebAssembly** minimalista para **ingresar ventas** (maestro–detalle) y **listar productos**, consumiendo el API en `https://localhost:7050/`.

---
## 0) Requisitos
- .NET 8 SDK (o 7+)
- Certificados HTTPS de desarrollo confiables:
  ```bash
  dotnet dev-certs https --trust
  ```
- API corriendo en `https://localhost:7050/` con CORS habilitado para el origen del cliente (ver **Anexo A**).

---
## 1) Crear el proyecto Blazor WASM
```bash
dotnet new blazorwasm -n VentasBlazor
cd VentasBlazor
```
> El proyecto expone una SPA estándar con `wwwroot/index.html` y `App.razor`.

---
## 2) Configurar `Program.cs` (cliente)
Registra un `HttpClient` con la **base URL del API**.

**`Program.cs`**
```csharp
using Microsoft.AspNetCore.Components.Web;
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

// 👉 URL de tu API
builder.Services.AddScoped(sp => new HttpClient {
    BaseAddress = new Uri("https://localhost:7050/")
});

await builder.Build().RunAsync();
```

> Si prefieres fijar los puertos del cliente, ajusta `Properties/launchSettings.json` (perfil `https`).

---
## 3) Modelos (DTOs) del cliente
Crea una carpeta `Models` y agrega:

**`Models/VentasDtos.cs`**
```csharp
namespace VentasBlazor.Models
{
    public class VentaLineaDto
    {
        public int IdVentaDeta { get; set; }           // lo devuelve el GET
        public string CodigoProducto { get; set; } = string.Empty;
        public decimal Cantidad { get; set; }
        public decimal PrecioUnitario { get; set; }
        public decimal LineTotal { get; set; }          // calculado por DB en GET
    }

    public class VentaCreateDto
    {
        public DateTime? Fecha { get; set; } = DateTime.UtcNow;
        public int IdVendedor { get; set; }
        public string Cardcode { get; set; } = string.Empty;
        public List<VentaLineaDto> Detalles { get; set; } = new();
    }

    public class CreateVentaResponse { public int idfact { get; set; } }
}
```

**`Models/Producto.cs`**
```csharp
namespace VentasBlazor.Models
{
    public class Producto
    {
        public int IdProducto { get; set; }
        public string CodigoProducto { get; set; } = string.Empty;
        public string NombreProducto { get; set; } = string.Empty;
        public decimal PrecioUnitario { get; set; }
        public int Stock { get; set; }
    }
}
```

---
## 4) Servicios HTTP
### 4.1 Ventas
**`Services/IVentasApi.cs`**
```csharp
using VentasBlazor.Models;

namespace VentasBlazor.Services
{
    public interface IVentasApi
    {
        Task<int> CrearVentaAsync(VentaCreateDto venta);
    }
}
```

**`Services/VentasApi.cs`**
```csharp
using System.Net.Http.Json;
using VentasBlazor.Models;

namespace VentasBlazor.Services
{
    public class VentasApi : IVentasApi
    {
        private readonly HttpClient _http;
        public VentasApi(HttpClient http) => _http = http;

        public async Task<int> CrearVentaAsync(VentaCreateDto venta)
        {
            var res = await _http.PostAsJsonAsync("api/ventas", venta);
            res.EnsureSuccessStatusCode();
            var payload = await res.Content.ReadFromJsonAsync<CreateVentaResponse>();
            return payload?.idfact ?? 0;
        }
    }
}
```

### 4.2 Productos
**`Services/IProductosApi.cs`**
```csharp
using VentasBlazor.Models;

namespace VentasBlazor.Services
{
    public interface IProductosApi
    {
        Task<List<Producto>> GetAllAsync();
        Task<bool> DeleteAsync(string codigoProducto);
    }
}
```

**`Services/ProductosApi.cs`**
```csharp
using System.Net.Http.Json;
using VentasBlazor.Models;

namespace VentasBlazor.Services
{
    public class ProductosApi : IProductosApi
    {
        private readonly HttpClient _http;
        public ProductosApi(HttpClient http) => _http = http;

        public async Task<List<Producto>> GetAllAsync()
        {
            return await _http.GetFromJsonAsync<List<Producto>>("api/productos")
                   ?? new List<Producto>();
        }

        public async Task<bool> DeleteAsync(string codigoProducto)
        {
            var res = await _http.DeleteAsync($"api/productos/{codigoProducto}");
            return res.IsSuccessStatusCode;
        }
    }
}
```

Registra los servicios en **`Program.cs`**:
```csharp
builder.Services.AddScoped<VentasBlazor.Services.IVentasApi, VentasBlazor.Services.VentasApi>();
builder.Services.AddScoped<VentasBlazor.Services.IProductosApi, VentasBlazor.Services.ProductosApi>();
```

---
## 5) Página **Nueva Venta** (maestro–detalle)
**`Pages/NuevaVenta.razor`**
```razor
@page "/nueva-venta"
@using System.Linq
@using VentasBlazor.Models
@inject VentasBlazor.Services.IVentasApi VentasApi

<h3>Nueva Venta</h3>

<EditForm Model="@venta" OnValidSubmit="@Guardar">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <fieldset>
        <legend>Encabezado</legend>
        <div class="mb-2">
            <label>Fecha (UTC):</label>
            <InputDate @bind-Value="venta.Fecha" class="form-control" />
        </div>
        <div class="mb-2">
            <label>Id Vendedor:</label>
            <InputNumber @bind-Value="venta.IdVendedor" class="form-control" />
        </div>
        <div class="mb-2">
            <label>Cardcode (cliente):</label>
            <InputText @bind-Value="venta.Cardcode" class="form-control" />
        </div>
    </fieldset>

    <fieldset class="mt-3">
        <legend>Detalles</legend>
        <table class="table table-sm">
            <thead>
                <tr>
                    <th>Código Producto</th>
                    <th class="text-end">Cantidad</th>
                    <th class="text-end">Precio Unitario</th>
                    <th class="text-end">Total Línea</th>
                    <th></th>
                </tr>
            </thead>
            <tbody>
                @if (venta.Detalles?.Count > 0)
                {
                    @for (int i = 0; i < venta.Detalles.Count; i++)
                    {
                        var linea = venta.Detalles[i];
                        <tr @key="linea">
                            <td><InputText @bind-Value="linea.CodigoProducto" class="form-control" /></td>
                            <td><InputNumber @bind-Value="linea.Cantidad" class="form-control" step="0.01" /></td>
                            <td><InputNumber @bind-Value="linea.PrecioUnitario" class="form-control" step="0.01" /></td>
                            <td class="text-end">@((linea.Cantidad * linea.PrecioUnitario).ToString("0.00"))</td>
                            <td><button type="button" class="btn btn-outline-danger btn-sm" @onclick="(() => QuitarLinea(i))">X</button></td>
                        </tr>
                    }
                }
                else
                {
                    <tr>
                        <td colspan="5"><em>No hay líneas.</em> <button type="button" class="btn btn-primary btn-sm" @onclick="AgregarLinea">Agregar primera línea</button></td>
                    </tr>
                }
            </tbody>
            <tfoot>
                <tr>
                    <td colspan="5"><button type="button" class="btn btn-outline-primary btn-sm" @onclick="AgregarLinea">+ Agregar línea</button></td>
                </tr>
                <tr>
                    <th colspan="3" class="text-end">Total:</th>
                    <th class="text-end">@TotalPreview().ToString("0.00")</th>
                    <th></th>
                </tr>
            </tfoot>
        </table>
    </fieldset>

    <div class="mt-3">
        <button class="btn btn-success" type="submit" disabled="@guardando">
            @(guardando ? "Guardando..." : "Guardar Venta")
        </button>
        @if (!string.IsNullOrWhiteSpace(msg))
        {
            <span class="ms-3">@msg</span>
        }
    </div>
</EditForm>

@code {
    private VentaCreateDto venta = new()
    {
        Fecha = DateTime.UtcNow,
        IdVendedor = 1,
        Cardcode = string.Empty,
        Detalles = new() { new() { CodigoProducto = string.Empty, Cantidad = 1, PrecioUnitario = 0 } }
    };

    private bool guardando;
    private string? msg;

    protected override void OnInitialized()
    {
        venta ??= new();
        venta.Detalles ??= new();
        if (venta.Detalles.Count == 0)
            venta.Detalles.Add(new() { CodigoProducto = string.Empty, Cantidad = 1, PrecioUnitario = 0 });
    }

    private void AgregarLinea() { venta.Detalles.Add(new() { Cantidad = 1, PrecioUnitario = 0 }); StateHasChanged(); }
    private void QuitarLinea(int idx) { if (idx >= 0 && idx < venta.Detalles.Count) { venta.Detalles.RemoveAt(idx); StateHasChanged(); } }
    private decimal TotalPreview() => venta.Detalles.Sum(x => x.Cantidad * x.PrecioUnitario);

    private async Task Guardar()
    {
        msg = null;
        if (string.IsNullOrWhiteSpace(venta.Cardcode)) { msg = "El Cardcode es requerido."; return; }
        if (venta.IdVendedor <= 0) { msg = "IdVendedor debe ser mayor a 0."; return; }
        if (venta.Detalles.Count == 0 || venta.Detalles.Any(d => string.IsNullOrWhiteSpace(d.CodigoProducto) || d.Cantidad <= 0 || d.PrecioUnitario < 0))
        { msg = "Revisa las líneas: código, cantidad (>0) y precio (>=0)."; return; }

        try
        {
            guardando = true;
            var idfact = await VentasApi.CrearVentaAsync(venta);
            msg = idfact > 0 ? $"✅ Venta creada. ID: {idfact}" : "✅ Venta creada.";
        }
        catch (Exception ex) { msg = $"❌ {ex.Message}"; }
        finally { guardando = false; StateHasChanged(); }
    }
}
```

---
## 6) Página **Productos** (listado + filtro + borrar)
**`Pages/Productos.razor`**
```razor
@page "/productos"
@using VentasBlazor.Models
@inject VentasBlazor.Services.IProductosApi Api

<h3>Productos</h3>

<div class="mb-2 d-flex gap-2">
    <input class="form-control" placeholder="Filtrar por código o nombre..."
           @bind="filtro" @bind:event="oninput" style="max-width:360px" />
    <button class="btn btn-outline-primary" @onclick="Cargar">Actualizar</button>
</div>

@if (cargando)
{
    <p>Cargando...</p>
}
else if (!string.IsNullOrWhiteSpace(error))
{
    <div class="alert alert-danger">@error</div>
}
else
{
    <table class="table table-striped table-sm">
        <thead>
            <tr>
                <th>Código</th>
                <th>Nombre</th>
                <th class="text-end">Precio</th>
                <th class="text-end">Stock</th>
                <th></th>
            </tr>
        </thead>
        <tbody>
            @if (filtrados.Count == 0)
            {
                <tr><td colspan="5"><em>Sin resultados.</em></td></tr>
            }
            else
            {
                @foreach (var p in filtrados)
                {
                    <tr @key="p.CodigoProducto">
                        <td>@p.CodigoProducto</td>
                        <td>@p.NombreProducto</td>
                        <td class="text-end">@p.PrecioUnitario.ToString("0.00")</td>
                        <td class="text-end">@p.Stock</td>
                        <td class="text-end">
                            <button class="btn btn-sm btn-outline-danger" @onclick="() => Borrar(p.CodigoProducto)">Borrar</button>
                        </td>
                    </tr>
                }
            }
        </tbody>
    </table>
}

@code {
    private List<Producto> productos = new();
    private List<Producto> filtrados = new();
    private string? filtro;
    private bool cargando;
    private string? error;

    protected override async Task OnInitializedAsync() => await Cargar();

    private async Task Cargar()
    {
        try
        {
            cargando = true; error = null;
            productos = await Api.GetAllAsync();
            AplicarFiltro();
        }
        catch (Exception ex) { error = ex.Message; }
        finally { cargando = false; }
    }

    private void AplicarFiltro()
    {
        if (string.IsNullOrWhiteSpace(filtro))
            filtrados = productos.OrderBy(p => p.NombreProducto).ToList();
        else
        {
            var f = filtro.Trim().ToLowerInvariant();
            filtrados = productos
                .Where(p => (p.CodigoProducto?.ToLowerInvariant().Contains(f) ?? false) ||
                            (p.NombreProducto?.ToLowerInvariant().Contains(f) ?? false))
                .OrderBy(p => p.NombreProducto)
                .ToList();
        }
        StateHasChanged();
    }

    private async Task Borrar(string codigo)
    {
        if (string.IsNullOrWhiteSpace(codigo)) return;
        if (!await Api.DeleteAsync(codigo)) { error = $"No se pudo borrar {codigo}"; return; }
        productos = productos.Where(p => p.CodigoProducto != codigo).ToList();
        AplicarFiltro();
    }
}
```

---
## 7) Agregar la navegación
**`Shared/NavMenu.razor`**
```razor
<li class="nav-item px-3">
    <NavLink class="nav-link" href="nueva-venta">
        <span class="oi oi-plus"></span> Nueva Venta
    </NavLink>
</li>
<li class="nav-item px-3">
    <NavLink class="nav-link" href="productos">
        <span class="oi oi-list"></span> Productos
    </NavLink>
</li>
```

---
## 8) Ejecutar
- Levanta el **API** en `https://localhost:7050/`.
- Levanta el **cliente**:
  ```bash
  dotnet run
  ```
- Abre `https://localhost:<puertoWASM>/nueva-venta` y prueba guardar una venta.

---
## 9) Solución de problemas rápidos
- **TypeError: Failed to fetch** → casi siempre **CORS/HTTPS**.
  - Confía el cert: `dotnet dev-certs https --trust`.
  - API: CORS debe permitir el origen del cliente (ver **Anexo A**).
  - Evita *mixed content*: cliente y API en **https**.
- **401 Unauthorized** → el endpoint del API tiene `[Authorize]` y no envías token.
- **500** → revisa `Console` del API; el fetch sí llegó.

---
## Anexo A — CORS (lado API)
En el **API**, habilita CORS para el origen del cliente (ej., `https://localhost:5231`).

```csharp
// Program.cs (API)
var allowedOrigins = new[] { "https://localhost:5231", "http://localhost:5230" };
builder.Services.AddCors(options =>
{
    options.AddPolicy("Blazor", p => p
        .WithOrigins(allowedOrigins)
        .AllowAnyHeader()
        .AllowAnyMethod());
});

var app = builder.Build();
app.UseHttpsRedirection();
app.UseCors("Blazor");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
app.Run();
```

---
## Anexo B — Ejemplo de cuerpo para `POST /api/ventas`
```json
{
  "fecha": "2025-08-30T20:00:00Z",
  "idVendedor": 1,
  "cardcode": "C0001",
  "detalles": [
    { "codigoProducto": "P-001", "cantidad": 2, "precioUnitario": 10.50 },
    { "codigoProducto": "P-002", "cantidad": 1, "precioUnitario": 5.00 }
  ]
}
```

---
### Listo
Con esto tienes un cliente Blazor WASM limpio y funcional para tu API. Siguiente parada: autocompletar de clientes/productos y guards con JWT 😉.

