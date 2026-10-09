# REGLAS DE ARQUITECTURA Y DESARROLLO - 2026_sistemabase

Este documento define las directrices y estándares obligatorios para la creación y mantenimiento de páginas, catálogos y endpoints en el proyecto `2026_sistemabase`. Toda nueva funcionalidad debe seguir estos principios.

---

## 1. ARQUITECTURA DE CATÁLOGOS Y GRILLAS (`DxGrid`)

### A. Paginación y Carga de Datos en Servidor (Obligatorio)
- **Prohibido:** No utilizar consultas que carguen colecciones completas a memoria con `.ToList()` en el servidor para luego paginarlas en el cliente (evita `OutOfMemoryException`).
- **Estándar:** Usar `GridDevExtremeDataSource<TDto>` en el cliente conectado a un endpoint `/api/{Entidad}/GridData`.
- **Backend:** El endpoint debe procesar `DataSourceLoadOptions` mediante `DataSourceLoader.LoadAsync(query, loadOptions, ct)` sobre un `IQueryable<T>` con `AsNoTracking()`. PostgreSQL debe ejecutar el `Skip`, `Take`, ordenación y filtrado. Debe capturar `OperationCanceledException` y retornar `Results.Empty` para evitar pausas del depurador cuando el cliente cancela peticiones en vuelo.
- **Cliente:** En `GetGridDataStreamAsync`, capturar `OperationCanceledException` y retornar `new MemoryStream("{\"data\":[],\"totalCount\":0}"u8.ToArray())` para amortiguar cancelaciones automáticas generadas por `DxGrid` al refrescar o filtrar.

### B. Configuración de `DxGrid`
```razor
<DxGrid @ref="Grid"
        Data="@GridDataSource"
        KeyFieldName="Id"
        PageSize="20"
        PageSizeSelectorVisible="true"
        PageSizeSelectorItems="@(new int[] { 5, 10, 20, 50, 100 })"
        PagerPosition="GridPagerPosition.Bottom"
        ShowFilterRow="@ShowFilterRow"
        ShowSearchBox="true"
        ShowGroupPanel="@ShowGroupPanel"
        AllowSelectRowByClick="true"
        @bind-SelectedDataItem="pRegistroSeleccionado"
        RowDoubleClick="OnRowDoubleClick"
        ColumnResizeMode="GridColumnResizeMode.NextColumn"
        TextWrapEnabled="false"
        EditMode="GridEditMode.PopupEditForm"
        EditFormButtonsVisible="false"
        PopupEditFormHeaderText="@TituloForm"
        PopupEditFormCssClass="modal-dialog-centered"
        EditModelSaving="OnEditModelSaving"
        CustomizeEditModel="Grid_CustomizeEditModel">
    <ToolbarTemplate>
        <ToolbarOpciones parFuncion="@parFuncion" ModuloChanged="OnClickModulo" />
    </ToolbarTemplate>
    ...
```
- **Selección:** Usar siempre `AllowSelectRowByClick="true"` junto con `@bind-SelectedDataItem="pRegistroSeleccionado"`. **NO mezclar con `FocusedRowEnabled="true"`** para evitar desincronizaciones entre el foco y la selección real.
- **Doble Clic:** Suscribir `RowDoubleClick="OnRowDoubleClick"` para abrir inmediatamente la edición del registro pulsado.
- **Edición Modal:** Utilizar siempre `EditMode="GridEditMode.PopupEditForm"` con **`EditFormButtonsVisible="false"`** para suprimir los botones nativos de DevExpress y gobernar los botones exclusivamente desde el `<EditFormTemplate>` ("Aceptar" y "Salir").
- **Prohibido:** No utilizar componentes `DxWindow` o `DxPopup` desacoplados para la edición de registros en catálogos (evita fugas de memoria y caídas de circuito Blazor durante desmontajes).

---

## 2. FORMULARIO DE EDICIÓN Y BOTONES (`EditFormTemplate`)

### A. Estructura y Etiquetas
- Seguir el diseño limpio y compacto de `CAT1100`:
  - Etiquetas concisas: `ID :`, `Nombre :`, `Descripción :`, etc. No incluir textos explicativos o técnicos dentro del caption.
  - El campo clave / ID debe ser de solo lectura (`ReadOnly="true"`). En Altas, el ID se inicializa en 0 o vacío y es asignado por la secuencia de la base de datos al guardar.
- Los inputs deben responder a `ReadOnly="@consultaItem"`.

### B. Control de Botones
- En **Alta** y **Editar**: Mostrar botón verde "Aceptar" (`SubmitFormOnClick="true"`) y botón "Salir" (`Click="OnCancelButtonClick"` llamando a `Grid.CancelEditAsync()`).
- En **Consultar**: Ocultar el botón "Aceptar" (`consultaBotCk = false`) y habilitar únicamente el botón "Salir".

### C. Mapeo del Modelo en `Grid_CustomizeEditModel`
- Obtener siempre los datos desde `e.DataItem`:
  ```csharp
  private void Grid_CustomizeEditModel(GridCustomizeEditModelEventArgs e)
  {
      if (e.IsNew)
      {
          ContextPrincipal = new TDto();
      }
      else if (e.DataItem is TDto dto)
      {
          ContextPrincipal = new TDto { /* mapeo de propiedades */ };
      }
      else if (pRegistroSeleccionado is TDto reg)
      {
          ContextPrincipal = new TDto { /* fallback seguro */ };
      }
  }
  ```

---

## 3. TOOLBAR Y COMANDOS (`OnClickModulo` Y `ToolbarOpciones`)

### A. Estándar Visual y Tooltips (Obligatorio)
- **Tooltips:** Toda barra de herramientas (`ToolbarOpciones.razor`) debe renderizar obligatoriamente `Tooltip="@(!string.IsNullOrWhiteSpace(o.Tooltip) ? o.Tooltip : o.MenuName)"` en cada `DxToolbarItem`. Las pestañas subordinadas de detalle deben replicar exactamente la misma experiencia con tooltips interactivos que la grilla principal.
- **Iconos Contextuales:** No usar iconos genéricos ni estáticos. Se debe vincular `IconCssClass` a clases de FontAwesome según la acción (`ALTA`: `fa-plus text-success`, `EDITAR`: `fa-pencil-alt text-primary`, `BAJA`: `fa-trash-alt text-danger`, `CONSULTAR`: `fa-eye text-info`, `FILTRO`: `fa-filter`, `ACTUALIZAR`: `fa-sync-alt`, etc.).
- **Fallback Tolerante a Fallos:** Si una función secundaria o subordinada (ej. `CFG1111`) aún no ha sido registrada en `tcmodulogrupofuncionopciones` de PostgreSQL, `ToolbarOpciones` debe proveer automáticamente las 6 opciones estándar (Alta, Editar, Baja, Consultar, Filtrar, Actualizar) con sus tooltips correspondientes, evitando barras vacías o excepciones de deserialización.

### B. Sinónimos de Comandos en `OnClickModulo`
El método receptor de eventos del toolbar debe contemplar sinónimos para asegurar compatibilidad con diferentes configuraciones de menú en la base de datos:
- **Alta:** `"ALTA"`, `"NUEVO"`, `"NUEVA"`, `"NEW"` -> Llama a `Grid.StartEditNewRowAsync()` o activa formulario en modo ALTA. Invocar `StateHasChanged()`.
- **Editar:** `"EDITAR"`, `"MODIFICAR"`, `"EDIT"` -> Llama a `Grid.StartEditDataItemAsync(item)` o conmuta a formulario en modo EDITAR. Invocar `StateHasChanged()`.
- **Consultar:** `"CONSULTAR"`, `"CONSULTA"`, `"VER"` -> Abre en solo lectura (`consultaItem = true`, `consultaBotCk = false`). Invocar `StateHasChanged()`.
- **Eliminar:** `"BAJA"`, `"ELIMINAR"`, `"BORRAR"`, `"DELETE"` -> Despliega el modal de confirmación `ModalCuestion`.
- **Filtro:** `"FILTRO"`, `"FILTRAR"` -> Alterna `ShowFilterRow`. Invocar `StateHasChanged()`.
- **Grupo:** `"GRUPO"`, `"AGRUPAR"` -> Alterna `ShowGroupPanel`. Invocar `StateHasChanged()`.
- **Expandir / Colapsar:** `"EXPANDIR"` / `"COLAPSAR"` -> `Grid.ExpandAllGroupRows()` / `Grid.CollapseAllGroupRows()`.
- **Actualizar:** `"ACTUALIZAR"`, `"REFRESCAR"`, `"RELOAD"` -> Recarga la fuente de datos. Invocar `StateHasChanged()`.

---

## 4. CONFIRMACIÓN DE BAJAS / ELIMINACIONES

- **Estándar:** Usar el componente Blazor nativo `ModalCuestion.razor`:
  ```razor
  <ModalCuestion Text="¿Deseas Borrar el Registro Seleccionado?"
                 @bind-ShowModal="@ShowDelete"
                 OnClose="OnConfirmDelete" />
  ```
- **Ventaja:** Al ser un componente Blazor puro con Bootstrap y `DxButton`, no depende de llamadas síncronas a JavaScript, eliminando cualquier riesgo de `JSDisconnectedException`.

---

## 5. VALIDACIÓN EN DOS FASES (FLUENTVALIDATION)

1. **Fase 1 (Cliente):** Al enviar el formulario (`HandleValidSubmitCon`), validar contra la clase `AbstractValidator<TDto>` (ubicada en `SistemaBase.Client.Shared/Reglas_de_Validacion`). Si es inválido, asignar `LeyendaError` y cancelar el guardado sin tocar la red.
2. **Fase 2 (Servidor):** En el método `PostAsync` / `PutAsync` del servicio en `SistemaBaseDAL`, revalidar con FluentValidation antes de persistir con EF Core.

---

## 6. CICLO DE VIDA Y RESILIENCIA CONTRA CAÍDAS (`IDisposable`)

- Toda página debe implementar `IDisposable`.
- Declarar un `CancellationTokenSource _cts = new();`.
- Pasar siempre `_cts.Token` a todas las peticiones asíncronas hacia servicios o APIs (`InsertAsync`, `UpdateAsync`, `DeleteAsync`).
- En el método `Dispose()`:
  ```csharp
  public void Dispose()
  {
      try
      {
          _cts.Cancel();
          _cts.Dispose();
      }
      catch { }
  }
  ```
- Esto cancela peticiones en vuelo cuando el usuario cambia de ruta o cierra la pestaña, evitando fallos de circuito zombi.

---

## 7. SEGURIDAD Y PERMISOS

1. **Cliente:**
   - La página debe heredar de `PageFuncionBase`.
   - En `OnInitializedAsync`, validar si `dataSystem.LogiState.Usuario` está presente. Si está vacío, resetear estado y redirigir inmediatamente a `"Login"`.
2. **API:**
   - Todos los endpoints mapeados en `MapApi*.cs` deben exigir autenticación y aplicar autorización granular por función:
     ```csharp
     group.MapPost("...", ...).WithTags("SistemaBase").RequireFuncion("CODIGO_FUNCION");
     ```

---

## 8. PATRÓN 2 — DOCUMENTOS TRANSACCIONALES / MAESTRO-DETALLE (`{CODIGO}ABC`)

### A. Concepto y Navegación
- Para documentos complejos (Órdenes de Compra, Facturas, Requisiciones, Usuarios con Perfiles), **no se utiliza un popup flotante para la captura general**.
- La página principal (`{CODIGO}.razor`) gestiona un flag `bool Editar`.
  - Si `!Editar`: muestra la grilla paginada en servidor de los registros maestros (`DxGrid` con `GridDevExtremeDataSource`).
  - Si `Editar`: conmuta la vista al componente hijo `{CODIGO}ABC.razor` (`@bind-Editar="@Editar"`), dejando la grilla en reposo ("en muerto") sin recargar la URL ni reiniciar el circuito Blazor.

### B. Encabezado y Detalle (`{CODIGO}ABC.razor`)
1. **Encabezado (Arriba):**
   - Captura de los datos generales de la entidad maestra en un `DxFormLayout`.
   - Botón verde "Aceptar" (`SubmitFormOnClick="true"`) y botón rojo "Salir" (`Click="OnSalirClick"` que emite `EditarChanged.InvokeAsync(false)`).
2. **Detalle Subordinado (Abajo):**
   - Ubicado en pestañas (`DxFormLayoutTabPages`).
   - Contiene una grilla (`DxGrid`) con su propia barra de herramientas para altas, bajas y consultas de renglones subordinados.
3. **Flujo de Bloqueo / Desbloqueo:**
   - En **Altas**: El detalle permanece inactivo o bloqueado con mensaje informativo hasta que el usuario llena el encabezado y pulsa "Aceptar".
   - Al guardar exitosamente el encabezado en PostgreSQL, el backend retorna la entidad con su clave primaria / secuencia asignada, la acción conmuta a "EDITAR" y el detalle se desbloquea automáticamente.
   - En **Editar**: El encabezado se precarga y la pestaña de detalle está desbloqueada desde el inicio.
   - En **Consultar**: Encabezado y detalle operan en solo lectura (`ReadOnly="true"`).

---

## 9. CONFIGURACIÓN CENTRALIZADA Y PROHIBICIÓN DE URLs HARDCODEADAS

### A. Prohibición Absoluta
- **Prohibido:** Queda estrictamente prohibido colocar URLs, IPs o puertos hardcodeados en código C# (por ejemplo, `"https://localhost:7217/..."` o `"https://localhost:7050/..."`), tanto en llamadas directas como en bloques de fallback.

### B. Fuente Única de Configuración
- Todas las URLs de servicios, APIs y cadenas de conexión deben provenir **exclusivamente** de los archivos de configuración centralizados en:
  `C:\Sistemas\Configuraciones\Conamat\`
  - `ConfigDesarrolloConamat.txt` (Desarrollo: formato JSON con variables como `ApiSistemaBase`, `ApiErpConfiguraciones`, etc.)
  - `ConfigDesarrolloConamat.enc` (Producción: cifrado con DPAPI / LocalMachine)

### C. Consumo en Servidor y Cliente
1. **En Servidor / Backend (`Program.cs` / `Configuraciones.GetSecret`):**
   - Se leen las variables mediante `Configuraciones.GetSecret("ApiSistemaBase")` y se inyectan en `builder.Services.AddHttpClient<TService>(client => client.BaseAddress = new Uri(apiSettings.UrlBase));`.
2. **En Cliente / Servicios (`Service*.cs`):**
   - Los métodos de los servicios cliente deben usar **rutas relativas** (ej. `api/Tccompania/Get/{id}`, `api/Tccompania/Insert`).
   - Para métodos que requieren `Uri` absoluta (como `ObtenerUriGridData()` para `GridDevExtremeDataSource`), se debe construir a partir de `_http.BaseAddress`:
     ```csharp
     if (_http.BaseAddress == null)
     {
         throw new InvalidOperationException("La URL base no está configurada en HttpClient. Verifique la configuración centralizada en C:\\Sistemas\\Configuraciones\\Conamat.");
     }
     var baseStr = _http.BaseAddress.ToString().TrimEnd('/') + "/";
     return new Uri(new Uri(baseStr), "api/{Entidad}/GridData");
     ```
   - Si `_http.BaseAddress` es nulo, se debe lanzar una excepción informativa; **nunca** colocar una URL por defecto como fallback.

---

## 10. AUTOMAPPER ESTRICTO EN EL DAL (PROHIBIDO `ActualizarEntidad` Y `SetValues`)

### A. Justificación y Directriz
- En tablas con 20 a 50 columnas, la asignación manual propiedad por propiedad (`ActualizarEntidad(actual, nuevo)`) es propensa a errores humanos, omisiones y mantenimiento insostenible.
- El uso de `CurrentValues.SetValues(dto)` en Entity Framework Core es peligroso: si el DTO no incluye ciertos campos o trae nulos, puede borrar información crítica o campos de auditoría.
- **Estándar:** Utilizar **AutoMapper** mediante perfiles dedicados (`Profile`).

### B. Mapeo Quirúrgico en Edición
- En el método `UpdateAsync` / `PutAsync`:
  ```csharp
  // 1. Obtener la entidad rastreada de la base de datos
  var entidad = await context.Tccompanias
      .FirstOrDefaultAsync(x => x.Idnumcia == dto.Idnumcia, cancellationToken);

  if (entidad is null)
      return new ServiceResult<Tccompania> { Successful = false, Message = "Registro no encontrado." };

  // 2. AutoMapper copia solo los campos coincidentes configurados
  _mapper.Map(dto, entidad);

  // 3. Asignar auditoría
  entidad.Fechaactualizacion = DateTime.UtcNow;

  // 4. EF Core ChangeTracker detecta exactamente qué propiedades cambiaron y emite un UPDATE SQL quirúrgico
  await context.SaveChangesAsync(cancellationToken);
  ```

### C. Configuración del Perfil (`Profile`)
- **Regla Estricta sobre Claves Primarias:**
  1. En `CreateMap<TCreateDto, TEntidad>()`: Se ignora la clave primaria (`.ForMember(x => x.Id, opt => opt.Ignore())`) porque la base de datos o secuencia la genera.
  2. En `CreateMap<TUpdateDto, TEntidad>()`: Se ignora la clave primaria y fecha de auditoría porque se mapea *sobre* la entidad rastreada existente (`_mapper.Map(dto, entidadExistente)`).
  3. En `CreateMap<TDto, TEntidad>()`: **NUNCA IGNORAR LA CLAVE PRIMARIA**. El DTO completo contiene el ID del registro que el endpoint `Update` y los servicios necesitan para localizar la entidad. Si se ignora el ID, llegará en `0` y la base de datos devolverá "Registro no encontrado". Solo ignorar `Fechaactualizacion`.
  4. En `CreateMap<TEntidad, TEntidad>()`: Configurar mapeo de entidad sobre entidad para el DAL, ignorando clave primaria y `Fechaactualizacion`.

```csharp
public class TccompaniaProfile : Profile
{
    public TccompaniaProfile()
    {
        // 1. Alta: ignora ID (autogenerado)
        CreateMap<TccompaniaCreateDto, Tccompania>()
            .ForMember(x => x.Idnumcia, opt => opt.Ignore());

        // 2. Edición vía UpdateDto: ignora ID y auditoría
        CreateMap<TccompaniaUpdateDto, Tccompania>()
            .ForMember(x => x.Idnumcia, opt => opt.Ignore())
            .ForMember(x => x.Fechaactualizacion, opt => opt.Ignore());

        // 3. Proyección a DTO y viceversa (NO ignorar ID en Dto -> Entidad)
        CreateMap<Tccompania, Tccompaniadto>();
        CreateMap<Tccompaniadto, Tccompania>()
            .ForMember(x => x.Fechaactualizacion, opt => opt.Ignore());

        // 4. Mapeo entidad a entidad para UpdateAsync en el DAL
        CreateMap<Tccompania, Tccompania>()
            .ForMember(x => x.Idnumcia, opt => opt.Ignore())
            .ForMember(x => x.Fechaactualizacion, opt => opt.Ignore());
    }
}
```

---

## 11. GESTIÓN DE ACCESO A DATOS CON `IDbContextFactory` Y CONCURRENCIA

### A. Ciclo de Vida de `DbContext`
- **Prohibido:** No inyectar `dbSistema_BaseContext` como campo singleton o servicio scoped de larga vida en los servicios del DAL. La acumulación de entidades en el `ChangeTracker` causa bloqueos de concurrencia y fugas de memoria (`OutOfMemoryException`).
- **Estándar:** Inyectar `IDbContextFactory<dbSistema_BaseContext>` y crear una instancia puntual por cada método u operación:
  ```csharp
  await using var context = await _contextFactory.CreateDbContextAsync(cancellationToken);
  ```
- En consultas de lectura, aplicar siempre `.AsNoTracking()`.

### B. Manejo de Concurrencia
- Capturar siempre `DbUpdateConcurrencyException` para alertar al usuario si otro proceso modificó el registro simultáneamente:
  ```csharp
  catch (DbUpdateConcurrencyException ex)
  {
      _logger.LogWarning(ex, "Conflicto de concurrencia al actualizar {Id}", dto.Idnumcia);
      return new ServiceResult<Tccompania>
      {
          Successful = false,
          Message = "El registro fue modificado por otro proceso o usuario. Recargue la pantalla e intente nuevamente."
      };
  }
  ```

---

## 12. TRÍADA DE DTOs DESACOPLADOS

- **Prohibido:** Nunca exponer ni enviar directamente los modelos de entidad crudos de Entity Framework hacia el cliente Blazor.
- **Estándar:** Toda tabla o catálogo debe contar con su tríada de DTOs en `SistemaBase.Client.Shared/DTOs`:
  1. **`{Entidad}CreateDto`**: Propiedades necesarias para la inserción. No incluye la clave primaria (es autogenerada o asignada por secuencia).
  2. **`{Entidad}UpdateDto`**: Propiedades modificables en la edición. Incluye la clave primaria obligatoria para identificar el registro a actualizar.
  3. **`{Entidad}Dto`**: Proyección completa utilizada para alimentar el `DxGrid`, visualización, reportes y transferencias entre capas.

---

## 13. ECOSISTEMA DE BASES DE DATOS (POSTGRESQL vs SQL SERVER REMOTO)

1. **Base de Datos Operativa (PostgreSQL):**
   - Es el motor de datos transaccional activo del sistema `2026_sistemabase`.
   - Todas las tablas del sistema residen aquí: `tccompanias`, `tcmodulogrupofuncionopciones`, `tcfuncionesopciones`, `tcusuarioperfilesfunciones`, etc.
   - Toda operación de alta, baja, cambio o consulta (`GridData`) corre contra PostgreSQL.
2. **SQL Server Remoto (`104.192.6.237`):**
   - Es **exclusivamente un repositorio de referencia** para consultar diagramas, estructuras heredadas o comparativas de datos.
   - **PROHIBIDO MODIFICAR:** No ejecutar sentencias `CREATE`, `ALTER`, `DROP`, `INSERT`, `UPDATE` o `DELETE` sobre el servidor SQL Server remoto.

---

## 14. EL CASCARÓN MAESTRO DE REFERENCIA (`CFG1100`) Y CHECKLIST DE REPLICACIÓN

### A. Filosofía del Proyecto (Principio de Eduardo Kerys)
> *"No se trata de avanzar rápido haciendo pantallas a medias; se trata de construir un cascarón perfecto, modular y 100% probado que sirva como molde para replicar todas las tablas y catálogos de los sistemas que se entregan en enero."*

### B. Catálogo Modelo
- El módulo **`CFG1100`** (Catálogo de Compañías / `Tccompania`) es la **referencia dorada** obligatoria que implementa la totalidad de estas directrices.

### C. Checklist de Replicación en 10 Pasos para Nuevos Catálogos
Cuando se requiera dar de alta una nueva pantalla o catálogo (ej. Sucursales `CFG1200`, Bancos `CFG1300`, etc.):

| Paso | Tarea | Ubicación del Archivo |
| :--- | :--- | :--- |
| **1** | Registrar permisos y menús | PostgreSQL: `tcmodulogrupofuncionopciones` y `tcusuarioperfilesfunciones`. |
| **2** | Crear Tríada de DTOs | `SistemaBase.Client.Shared/DTOs/{Entidad}CreateDto.cs`, `{Entidad}UpdateDto.cs`, `{Entidad}Dto.cs`. |
| **3** | Crear Validadores FluentValidation | `SistemaBase.Client.Shared/Reglas_de_Validacion/{Entidad}Validators.cs`. |
| **4** | Configurar Perfil AutoMapper | `FuncionesGeneralesServer.Shared/AutoMapper/{Entidad}Profile.cs` (ignorar IDs y auditoría). |
| **5** | Implementar Servicio DAL | `SistemaBaseDAL/Servicios/{Modulo}/ServicioTc{Entidad}.cs` (con `IDbContextFactory`, `_mapper.Map` y `AsNoTracking`). |
| **6** | Crear Endpoints Minimal API | `ApiSistemaBase/MapApis/{Modulo}/MapApi{Entidad}.cs` con `GridData` y `.RequireFuncion("CODIGO")`. |
| **7** | Implementar Servicio Cliente HTTP y Registrar en DI | `SistemaBase.Client/ServiciosCliente/ServiceTc{Entidad}.cs` (rutas relativas). **OBLIGATORIO:** Registrar en `SistemaBase/Program.cs` (`AddHttpClient` con URL centralizada) y en `SistemaBase.Client/Program.cs` (`AddScoped`). |
| **8** | Crear Vista Principal Blazor | `SistemaBase.Client/Pages/{Modulo}/{CODIGO}.razor` y `.cs` (`DxGrid`, `ToolbarOpciones` con tooltips, `ModalCuestion`, `IDisposable`). |
| **9** | Crear Formulario ABC y Pestañas Subordinadas | `SistemaBase.Client/Pages/{Modulo}/{CODIGO}ABC.razor` y `.cs` (Encabezado + Tabs subordinadas `{CODIGO_SUBORDINADO}.razor` con `ToolbarOpciones`, tooltips y `DxGrid` modal). |
| **10**| Compilación y Verificación | Compilar toda la solución con **0 Errores**, **0 URLs hardcodeadas** y verificar en navegador. |

---

## 15. ESTÁNDARES AVANZADOS DE DAL, SEGURIDAD Y RESILIENCIA (AUDITORÍA 2026)

### A. Erradicación Total de `_context` e Inyección Estricta de `IDbContextFactory`
- **Prohibición Absoluta:** Jamás inyectar `dbSistema_BaseContext` como campo privado (`private readonly dbSistema_BaseContext _context;`) en servicios del DAL. Los contextos scoped o mantenidos como campos acumulan entidades en el `ChangeTracker`, causando fugas de memoria y bloqueos de concurrencia.
- **Constructor Limpio:** El constructor del servicio solo recibe `IDbContextFactory<dbSistema_BaseContext>`, `IMapper`, `ILogger<T>`, `IMemoryCache` y validadores.
- **Ámbito Puntual por Operación:** Todo método (`GetsAsyn`, `GetsAllAsyn`, `GetAsync`, `PostAsync`, `PutAsync`, `DeleteAsync`, `ExistsAsync`, `CountAsync`) debe instanciar y liberar su propio contexto:
  ```csharp
  await using var context = await _contextFactory.CreateDbContextAsync(cancellationToken);
  ```

### B. Contrato Estricto del Endpoint `GridData` (GET Obligatorio para DevExpress)
- **Regla Inquebrantable:** El endpoint de datos para la grilla Blazor `DxGrid` debe responder a **HTTP GET**:
  ```csharp
  group.MapGet("{Entidad}/GridData",
      async (DataSourceLoadOptions loadOptions,
             IDbContextFactory<dbSistema_BaseContext> contextFactory,
             CancellationToken ct) =>
      {
          try
          {
              await using var context = await contextFactory.CreateDbContextAsync(ct);
              var query = context.Tc{Entidad}.AsNoTracking().Select(...);
              var resultado = await DataSourceLoader.LoadAsync(query, loadOptions, ct);
              return Results.Ok(resultado);
          }
          catch (OperationCanceledException)
          {
              return Results.Empty;
          }
      }).WithTags("SistemaBase").RequireFuncion("{CODIGO}");
  ```
- **Razón:** El conector cliente `GridDevExtremeDataSource<TDto>` de DevExpress emite solicitudes HTTP GET con parámetros OData/DevExtreme (`skip`, `take`, `sort`, `filter`) y requiere el formato `{ data: [...], totalCount: N }`.
- **Integraciones Secundarias / GridRequest:** Si clientes móviles o externos envían un objeto de paginación manual (`GridRequest`), este debe exponerse en un endpoint complementario (ej. `group.MapPost("{Entidad}/GridData", ...)` o `{Entidad}/GridDataPaginado`), **NUNCA** sustituyendo el endpoint `GET` de DevExtreme.

### C. Gestión de Memoria en Caché (`IMemoryCache`)
- **Regla de Tamaño en .NET:** Si se configura `options.SizeLimit` en `builder.Services.AddMemoryCache()`, **toda entrada** en toda la aplicación debe asignar `.SetSize(n)`. Omitirlo lanza `InvalidOperationException` en tiempo de ejecución. Por ello, el registro general se mantiene sin `SizeLimit` global estricto a menos que todos los servicios del DAL especifiquen tamaño.
- **Operaciones Síncronas:** Las operaciones sobre `IMemoryCache` son locales en RAM. No envolverlas en tareas asíncronas artificiales (`Task.FromResult` / `Task.CompletedTask`). Usar métodos directos: `GetFromCache<T>`, `SetInCache<T>`, `RemoveFromCache`.

### D. Resiliencia de Conexión en Base de Datos (Npgsql)
- En `Program.cs` de la API, configurar `AddPooledDbContextFactory` con reintentos automáticos para tolerar microcortes de red con PostgreSQL:
  ```csharp
  builder.Services.AddPooledDbContextFactory<dbSistema_BaseContext>(options =>
  {
      options.UseNpgsql(cadenaNpgsql, npgsqlOptions =>
      {
          npgsqlOptions.EnableRetryOnFailure(maxRetryCount: 3, maxRetryDelay: TimeSpan.FromSeconds(5), errorCodesToAdd: null);
          npgsqlOptions.CommandTimeout(60);
      });
  });
  ```

### E. Sanitización y Prevención XSS (`.NoXss()`)
- En las reglas de validación de FluentValidation (`{Entidad}Validators.cs`), utilizar la extensión `.NoXss()` en campos de texto libre para bloquear contenido malicioso con scripts o etiquetas HTML sospechosas.

### F. Logging Estructurado y Seguro
- Registrar identificadores puntuales (`{Idnumcia}`) en vez de la entidad completa (`{@Registro}`) para proteger datos confidenciales y optimizar el almacenamiento de logs.

### G. Generación de Claves Primarias Concurrentes
- Prohibido en entornos productivos depender de `MaxAsync() + 1` en el código para asignar identificadores. La secuencia o `IDENTITY` nativo de PostgreSQL debe gobernar la generación de claves primarias para garantizar atomicidad ante transacciones concurrentes.

### H. Organización y Agrupación en Swagger por Catálogo (`.WithTags("{Catalogo}")`)
- **Prohibido:** Jamás utilizar un tag genérico global como `.WithTags("SistemaBase")` para todos los endpoints de la API. Esto satura la documentación y provoca que Swagger UI agrupe decenas de endpoints en una sola sección monolítica y desordenada.
- **Estándar Obligatorio:** Cada archivo `MapApi{Catalogo}.cs` debe definir un tag temático correspondiente a su catálogo o entidad funcional:
  ```csharp
  var group = app.MapGroup("/api").WithTags("Companias").RequireAuthorization().RequireRateLimiting("default");
  ```
  O en cada endpoint del catálogo:
  ```csharp
  group.MapGet("Tccompania/GridData", ...).WithTags("Companias");
  ```
- **Convención de Nombres de Tags:**
  - `Tccompania` / `Tccompaniasdireccione` -> `.WithTags("Companias")`
  - `Tcusuario` / `Tcusuarioperfile` -> `.WithTags("Usuarios")`
  - `Tcperfile` -> `.WithTags("Perfiles")`
  - `Tcicono` -> `.WithTags("Iconos")`
  - `Tcsistemabases` -> `.WithTags("Sistema Base")`
  - `Tccatalogoreporte` -> `.WithTags("Reportes")`
- **Efecto en Swagger UI:** Swagger renderiza un acordeón independiente, limpio y colapsable por cada catálogo del ERP, manteniendo la API 100% navegable y profesional.



