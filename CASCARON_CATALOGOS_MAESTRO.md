# ARQUITECTURA MAESTRA Y CASCARÓN ESTÁNDAR — 2026_sistemabase

> **DOCUMENTO DE REFERENCIA OFICIAL Y GUÍA DE REPLICACIÓN**  
> Este documento define el **Cascarón Maestro (Golden Template)** para el desarrollo de catálogos, entidades y pantallas en `2026_sistemabase`.  
> **Filosofía del proyecto (Principio de Eduardo):** *"No se trata de avanzar rápido ni hacer programas a medias; se trata de construir un cascarón perfecto, modular y 100% probado que sirva como molde para replicar todas las tablas y catálogos de los sistemas que se entregan en enero."*

---

## 1. PRINCIPIOS FUNDAMENTALES DE LA ARQUITECTURA

1. **Replicabilidad Sistémica:** Toda entidad debe estructurarse exactamente con las mismas capas y convenciones de nombres para que cualquier desarrollador o IA pueda reproducirla en minutos sin errores.
2. **Cero Mapeos Manuales y Cero `SetValues`:** Queda erradicado el mapeo manual propiedad por propiedad (`ActualizarEntidad`) y el peligroso `CurrentValues.SetValues()`. Se utiliza **AutoMapper** para copiar automáticamente propiedades coincidentes y aislar campos protegidos (claves primarias, auditoría).
3. **Gestión de Memoria y Resiliencia (`IDbContextFactory`):** Cada operación en el DAL crea y desecha su propio `DbContext` mediante `await using var context = await _contextFactory.CreateDbContextAsync(ct)`. No se conservan contextos de larga duración ni entidades atadas en el `ChangeTracker`.
4. **Paginación y Filtrado 100% en Base de Datos:** `DxGrid` se conecta a endpoints `GridData` mediante `GridDevExtremeDataSource<TDto>`, delegando a PostgreSQL el `Skip`, `Take`, ordenación y filtrado. Jamás se hace `.ToList()` de toda la tabla en el servidor.
5. **Configuración Centralizada Estricta:** Ninguna URL, IP o puerto puede estar hardcodeado en C#. Todo proviene de `C:\Sistemas\Configuraciones\Conamat\` (`ConfigDesarrolloConamat.txt` / `ConfigDesarrolloConamat.enc`).
6. **Ecosistema de Bases de Datos:**
   - **PostgreSQL:** Base de datos activa, transaccional y operativa del sistema. Toda alta, baja, modificación y consulta corre aquí.
   - **SQL Server Remoto (`104.192.6.237`):** Solo repositorio de referencia histórica y diagramas. **PROHIBIDO MODIFICAR O ESCRIBIR EN ÉL.**
7. **Resiliencia de Circuito Blazor:** Uso de `IDisposable`, propagación estricta de `CancellationTokenSource _cts` y amortiguación de cancelaciones en `DxGrid` para evitar caídas de circuito o hilos zombis.

---

## 2. ESTRUCTURA COMPLETA DE UN CATÁLOGO (FULL STACK)

Cada nuevo catálogo o tabla consta de **8 componentes clave** distribuidos en sus proyectos correspondientes:

```
[PostgreSQL] (Tablas: tcentidad, tcmodulogrupofuncionopciones, tcusuarioperfilesfunciones)
   │
   ├── [SistemaBase.Client.Shared]
   │     ├── DTOs (EntidadCreateDto, EntidadUpdateDto, EntidadDto)
   │     └── Reglas_de_Validacion (EntidadCreateValidator, EntidadUpdateValidator)

   │
   ├── [FuncionesGeneralesServer.Shared]
   │     └── AutoMapper (EntidadProfile : Profile)
   │
   ├── [SistemaBaseDAL]
   │     └── Servicios/{Modulo}/ServicioTc{Entidad}.cs (Usa IDbContextFactory y AutoMapper)
   │
   ├── [ApiSistemaBase]
   │     └── MapApis/{Modulo}/MapApi{Entidad}.cs (Minimal API con .RequireFuncion("CODIGO"))
   │
   └── [SistemaBase.Client]
         ├── ServiciosCliente/ServiceTc{Entidad}.cs (HttpClient con rutas relativas)
         ├── Pages/{Modulo}/{CODIGO}.razor y .cs (Grilla maestra DxGrid, Toolbar, ModalCuestion)
         └── Pages/{Modulo}/{CODIGO}ABC.razor y .cs (Captura Patrón 2: Encabezado + Detalle tabs)
```

---

## 3. ESPECIFICACIÓN TÉCNICA DETALLADA (EL CASCARÓN)

Tomando como referencia dorada el módulo **`CFG1100`** (`tccompanias`):

### PIEZA 1: DTOs Desacoplados (`SistemaBase.Client.Shared/DTOs`)
Nunca se expone la entidad cruda de Entity Framework hacia el cliente. Se diferencian los propósitos:
- **`{Entidad}CreateDto`**: Campos requeridos para el alta (sin ID de base de datos).
- **`{Entidad}UpdateDto`**: Campos modificables en edición (incluye ID primario obligatorio).
- **`{Entidad}Dto`**: Proyección completa para consulta, grillas y respuestas al usuario.

```csharp
// Ejemplo: TccompaniaUpdateDto.cs
public class TccompaniaUpdateDto
{
    public int Idnumcia { get; set; }
    public string Idclaveexterna { get; set; } = string.Empty;
    public string Nombre { get; set; } = string.Empty;
    public string? Nombrecorto { get; set; }
    public string? Razonsocial { get; set; }
    public string? Rfc { get; set; }
    public string? Representantelegal { get; set; }
    public int? Regimenfiscal { get; set; }
    public bool? Activo { get; set; }
}
```

---

### PIEZA 2: Validación con FluentValidation (`SistemaBase.Client.Shared/Reglas_de_Validacion`)
Se definen validadores independientes para creación y actualización:
```csharp
public class TccompaniaUpdateValidator : AbstractValidator<TccompaniaUpdateDto>
{
    public TccompaniaUpdateValidator()
    {
        RuleFor(x => x.Idnumcia)
            .GreaterThan(0).WithMessage("El identificador de la compañía es obligatorio.");

        RuleFor(x => x.Nombre)
            .NotEmpty().WithMessage("El nombre de la compañía es obligatorio.")
            .MaximumLength(150).WithMessage("El nombre no puede exceder 150 caracteres.");

        RuleFor(x => x.Rfc)
            .MaximumLength(13).WithMessage("El RFC no puede exceder 13 caracteres.");
    }
}
```

---

### PIEZA 3: Perfil de AutoMapper (`FuncionesGeneralesServer.Shared/AutoMapper`)
Mapea automáticamente propiedades coincidentes, protegiendo claves en altas y auditoría:
```csharp
public class TccompaniaProfile : Profile
{
    public TccompaniaProfile()
    {
        // 1. CREATE: Mapea DTO hacia entidad nueva, ignorando la clave primaria (la base o secuencia la asigna)
        CreateMap<TccompaniaCreateDto, Tccompania>()
            .ForMember(x => x.Idnumcia, opt => opt.Ignore());

        // 2. UPDATE: Mapea DTO sobre entidad existente en el DAL, ignorando clave primaria y auditoría
        CreateMap<TccompaniaUpdateDto, Tccompania>()
            .ForMember(x => x.Idnumcia, opt => opt.Ignore())
            .ForMember(x => x.Fechaactualizacion, opt => opt.Ignore());

        // 3. CONSULTA / RESPUESTA: Mapea entidad a DTO y DTO a entidad (NUNCA ignorar clave primaria en DTO -> Entidad)
        CreateMap<Tccompania, Tccompaniadto>();
        CreateMap<Tccompaniadto, Tccompania>()
            .ForMember(x => x.Fechaactualizacion, opt => opt.Ignore());

        // 4. DAL UPDATE INTERNO: Copia entre entidades rastreadas ignorando clave primaria y auditoría
        CreateMap<Tccompania, Tccompania>()
            .ForMember(x => x.Idnumcia, opt => opt.Ignore())
            .ForMember(x => x.Fechaactualizacion, opt => opt.Ignore());
    }
}
```

---

### PIEZA 4: Servicio DAL con `IDbContextFactory` (`SistemaBaseDAL/Servicios`)
- Crea y libera el `DbContext` por operación (`await using var context = await _contextFactory...`).
- Utiliza `.AsNoTracking()` en consultas de solo lectura.
- Ejecuta `_mapper.Map(dto, entidadExistente)` para que EF Core actualice solo los campos modificados.
- Respeta `CancellationToken` y captura concurrencia (`DbUpdateConcurrencyException`).

```csharp
public async Task<ServiceResult<Tccompania>> UpdateAsync(
    TccompaniaUpdateDto dto, 
    CancellationToken cancellationToken = default)
{
    var validacion = await _validatorUpdate.ValidateAsync(dto, cancellationToken);
    if (!validacion.IsValid)
    {
        return new ServiceResult<Tccompania>
        {
            Successful = false,
            Message = string.Join("; ", validacion.Errors.Select(x => x.ErrorMessage))
        };
    }

    try
    {
        await using var context = await _contextFactory.CreateDbContextAsync(cancellationToken);

        var entidad = await context.Tccompanias
            .FirstOrDefaultAsync(x => x.Idnumcia == dto.Idnumcia, cancellationToken);

        if (entidad is null)
        {
            return new ServiceResult<Tccompania>
            {
                Successful = false,
                Message = "Registro no encontrado."
            };
        }

        // Mapeo automático limpio sin métodos manuales propiedad por propiedad
        _mapper.Map(dto, entidad);

        entidad.Fechaactualizacion = DateTime.UtcNow;

        await context.SaveChangesAsync(cancellationToken);

        return new ServiceResult<Tccompania>
        {
            Successful = true,
            Message = "Registro actualizado correctamente.",
            Value = entidad
        };
    }
    catch (DbUpdateConcurrencyException ex)
    {
        _logger.LogWarning(ex, "Conflicto de concurrencia al actualizar {Id}", dto.Idnumcia);
        return new ServiceResult<Tccompania>
        {
            Successful = false,
            Message = "El registro fue modificado por otro proceso. Recargue e intente nuevamente."
        };
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error actualizando compañía {Id}", dto.Idnumcia);
        throw;
    }
}
```

---

### PIEZA 5: Endpoints Minimal API (`ApiSistemaBase/MapApis`)
- Protegidos obligatoriamente con `.RequireFuncion("CODIGO_FUNCION")`.
- `GridData` ejecuta `DataSourceLoader.LoadAsync(query, loadOptions, ct)` sobre un `IQueryable<T>` con `AsNoTracking()`.

```csharp
public static void MapApiCompanias(this IEndpointRouteBuilder routes)
{
    var group = routes.MapGroup("/api/Tccompania");

    // GridData para DxGrid (Server-side Skip/Take)
    group.MapGet("/GridData", async (
        DataSourceLoadOptions loadOptions,
        ServicioTccompanias servicio,
        CancellationToken ct) =>
    {
        try
        {
            var query = servicio.Consultar(); // IQueryable<Tccompania> AsNoTracking
            var loadResult = await DataSourceLoader.LoadAsync(query, loadOptions, ct);
            return Results.Ok(loadResult);
        }
        catch (OperationCanceledException)
        {
            return Results.Empty;
        }
    })
    .WithTags("SistemaBase")
    .RequireFuncion("CFG1100");

    // CRUD estándar
    group.MapGet("/Get/{id:int}", ...).RequireFuncion("CFG1100");
    group.MapPost("/Insert", ...).RequireFuncion("CFG1100");
    group.MapPut("/Update", ...).RequireFuncion("CFG1100");
    group.MapDelete("/Delete/{id:int}", ...).RequireFuncion("CFG1100");
}
```

---

### PIEZA 6: Servicio Cliente HTTP (`SistemaBase.Client/ServiciosCliente`)
- Rutas relativas (`api/Tccompania/Get/{id}`, `api/Tccompania/Insert`).
- `ObtenerUriGridData()` deriva estrictamente de `_http.BaseAddress`. Cero fallbacks hardcodeados.
- **DOBLE REGISTRO EN DI (OBLIGATORIO):** Cada servicio HTTP cliente debe registrarse en:
  1. `SistemaBase/Program.cs` (Host Servidor): `builder.Services.AddHttpClient<ServiceTc{Entidad}>(client => client.BaseAddress = new Uri(apiSettings.UrlBase));`
  2. `SistemaBase.Client/Program.cs` (Host WebAssembly): `builder.Services.AddScoped<ServiceTc{Entidad}>();`
  *(Omitir el registro en el servidor provocará `InvalidOperationException` y desconectará el circuito de Blazor al abrir componentes subordinados).*

```csharp
public Uri ObtenerUriGridData()
{
    if (_http.BaseAddress == null)
    {
        throw new InvalidOperationException(
            "La URL base no está configurada en HttpClient. Verifique C:\\Sistemas\\Configuraciones\\Conamat.");
    }
    var baseStr = _http.BaseAddress.ToString().TrimEnd('/') + "/";
    return new Uri(new Uri(baseStr), "api/Tccompania/GridData");
}
```

---

### PIEZA 7: Vista Principal de Grilla (`{CODIGO}.razor` y `.cs`)
- Hereda de `PageFuncionBase`. Valida `dataSystem.LogiState.Usuario` en `OnInitializedAsync`, redirigiendo a `Login` si no hay sesión.
- Conecta `DxGrid` a `GridDevExtremeDataSource<TDto>`.
- **Toolbar Estándar (`ToolbarOpciones`):** 
  - Renderiza tooltips obligatorios (`Tooltip="@(!string.IsNullOrWhiteSpace(o.Tooltip) ? o.Tooltip : o.MenuName)"`).
  - Utiliza iconos contextuales FontAwesome (`fa-plus`, `fa-pencil-alt`, `fa-trash-alt`, `fa-eye`, etc.).
  - Incluye fallback automático a las 6 opciones básicas si la función aún no está configurada en la BD.
  - Maneja sinónimos en `OnClickModulo` (`ALTA`, `EDITAR`, `CONSULTAR`, `BAJA`, `FILTRO`, `GRUPO`, `ACTUALIZAR`) llamando a `StateHasChanged()`.
- **Doble Clic:** `RowDoubleClick="OnRowDoubleClick"` conmuta inmediatamente a modo EDITAR con el registro pulsado (con fallback a `pRegistroSeleccionado`).
- **Selección:** `AllowSelectRowByClick="true"` con `@bind-SelectedDataItem="pRegistroSeleccionado"`. **NUNCA mezclar con `FocusedRowEnabled="true"`**.
- **Confirmación de Borrado:** `ModalCuestion.razor` (Blazor nativo con Bootstrap y `DxButton`, cero riesgo de `JSDisconnectedException`).
- **Resiliencia:** `CancellationTokenSource _cts` e `IDisposable`. Cancela peticiones en vuelo al desmontar.

```razor
@page "/CFG1100"
@page "/CFG1100/{parFuncion}"
@inherits SistemaBase.Client.Servicios.PageFuncionBase
@implements IDisposable

@if (!Editar)
{
    <DxGrid @ref="Grid"
            Data="@GridDataSource"
            KeyFieldName="Idnumcia"
            PageSize="20"
            ShowFilterRow="@ShowFilterRow"
            ShowSearchBox="true"
            AllowSelectRowByClick="true"
            @bind-SelectedDataItem="pRegistroSeleccionado"
            RowDoubleClick="OnRowDoubleClick" ...>
        <ToolbarTemplate>
            <ToolbarOpciones parFuncion="@parFuncion" ModuloChanged="OnClickModulo" />
        </ToolbarTemplate>
        <Columns>
            ...
        </Columns>
    </DxGrid>
    <ModalCuestion Text="¿Deseas Borrar el Registro Seleccionado?"
                   @bind-ShowModal="@ShowDelete"
                   OnClose="OnConfirmDelete" />
}
else
{
    <CFG1100ABC @bind-Editar="@Editar"
                parFuncion="@parFuncion"
                Accion="@Accion"
                CompaniaEdit="@CompaniaSeleccionada"
                OnGuardadoExitoso="@ActualizarGrid" />
}
```

---

### PIEZA 8: Formulario de Captura Maestro-Detalle (`{CODIGO}ABC.razor` y `.cs`)
- **Encabezado (Arriba):** `DxFormLayout` con inputs concisos (`ReadOnly="@consultaItem"`), botón verde "Aceptar" (`SubmitFormOnClick="true"`) y botón rojo "Salir" (`Click="@OnSalirClick"` que apaga `Editar`).
- **Detalle Subordinado (Abajo):** `DxFormLayoutTabPages` con pestañas para entidades dependientes (Direcciones, Contactos, APIs, Repositorios).
- **Componentes Subordinados de Pestaña:** Cada pestaña aloja un componente independiente (ej. `CFG1111.razor` para Direcciones) que:
  - Recibe la clave padre (`parIdNumCia`) y el modo de consulta (`consultaItem`).
  - Tiene su propio `ToolbarOpciones` con tooltips e iconografía FontAwesome.
  - Tiene su propia grilla `DxGrid` con edición `PopupEditForm` (subordinado CRUD completo).
- **Regla de Bloqueo en Altas:** En `Accion == "ALTA"`, las pestañas inferiores permanecen bloqueadas con alerta informativa hasta que se guarde el encabezado y PostgreSQL asigne la clave primaria. Al guardar con éxito, conmuta a `EDITAR` y desbloquea el detalle.

---

## 4. GUÍA DE REPLICACIÓN PASO A PASO (CHECKLIST PARA NUEVAS TABLAS)

Cuando se requiera dar de alta un nuevo catálogo (ej. Sucursales `CFG1200`, Bancos `CFG1300`, Monedas, Proveedores):

| Paso | Acción | Archivo / Ubicación |
| :--- | :--- | :--- |
| **1** | Registrar función y permisos | PostgreSQL: `tcmodulogrupofuncionopciones` y `tcusuarioperfilesfunciones`. |
| **2** | Crear DTOs desacoplados | `SistemaBase.Client.Shared/DTOs/{Entidad}CreateDto.cs`, `{Entidad}UpdateDto.cs`, `{Entidad}Dto.cs`. |
| **3** | Crear Validadores FluentValidation | `SistemaBase.Client.Shared/Reglas_de_Validacion/{Entidad}Validators.cs`. |
| **4** | Crear Perfil AutoMapper | `FuncionesGeneralesServer.Shared/AutoMapper/{Entidad}Profile.cs` (no ignorar ID en DTO->Entidad). |
| **5** | Implementar Servicio DAL | `SistemaBaseDAL/Servicios/{Modulo}/ServicioTc{Entidad}.cs` (usar `IDbContextFactory`, `_mapper.Map` y `AsNoTracking`). |
| **6** | Crear Endpoints API | `ApiSistemaBase/MapApis/{Modulo}/MapApi{Entidad}.cs` con `.RequireFuncion("CODIGO")`. |
| **7** | Crear Servicio Cliente HTTP y Registrar en DI | `SistemaBase.Client/ServiciosCliente/ServiceTc{Entidad}.cs`. **Doble registro:** en `SistemaBase/Program.cs` (`AddHttpClient`) y `SistemaBase.Client/Program.cs` (`AddScoped`). |
| **8** | Crear Página Blazor Principal | `SistemaBase.Client/Pages/{Modulo}/{CODIGO}.razor` y `.cs` (Grilla `DxGrid`, `ToolbarOpciones` con tooltips). |
| **9** | Crear Componente ABC y Pestañas Detalle | `SistemaBase.Client/Pages/{Modulo}/{CODIGO}ABC.razor` y componentes subordinados `{CODIGO_SUB}.razor` con toolbar con tooltips. |
| **10**| Compilar y Verificar | Ejecutar compilación completa: **0 Errores** y **0 URLs hardcodeadas**. |

---

## 5. PROHIBICIONES Y REGLAS ESTRICTAS DE CÓDIGO

1. ❌ **PROHIBIDO:** Usar `CurrentValues.SetValues(dto)` en EF Core.
2. ❌ **PROHIBIDO:** Crear métodos manuales propiedad por propiedad tipo `ActualizarEntidad(actual, nuevo)`.
3. ❌ **PROHIBIDO:** Mantener instancias de `DbContext` como campos singleton o de larga duración sin liberar.
4. ❌ **PROHIBIDO:** Colocar URLs fijas (`https://localhost:7217`, `https://localhost:7050`, IPs, etc.) en código C#.
5. ❌ **PROHIBIDO:** Cargar colecciones completas a memoria con `.ToList()` en el servidor para paginarlas en Blazor.
6. ❌ **PROHIBIDO:** Usar `FocusedRowEnabled="true"` en `DxGrid` al mismo tiempo que `AllowSelectRowByClick="true"`.
7. ❌ **PROHIBIDO:** Usar `DxPopup` o `DxWindow` desacoplados para la edición de registros en catálogos.
8. ❌ **PROHIBIDO:** Realizar llamadas asíncronas sin enlazar el `CancellationToken` (`_cts.Token`).
