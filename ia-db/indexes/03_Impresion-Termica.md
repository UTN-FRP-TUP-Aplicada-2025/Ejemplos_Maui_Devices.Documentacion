# 03 — Impresión térmica Bluetooth

> **Propósito:** los tres ejemplos de impresión sobre la térmica 58 mm, la evolución de ESC/POS a mano → motor DSL → overlay con catálogo de errores, y la documentación del hardware.
> **Fuente primaria:** `Ejemplos_Devices/Printer/` (3 proyectos + `Ejemplo_Docs_Printer/`, ~3.100 líneas de manuales).
> **Índices relacionados:** [00_MASTER-INDEX](00_MASTER-INDEX.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (la híbrida hereda este overlay) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md).

---

## 1. Los tres proyectos: una evolución en tres saltos

| # | Proyecto | `ApplicationId` | Cómo genera los bytes | Cómo maneja los errores |
|---|----------|-----------------|-----------------------|-------------------------|
| 1 | `Ejemplo_ThermalPrinter` | `com.ejemplos.devices.ThermalPrinter` | **ESC/POS a mano** con `ESCPOS_NET` 2.2.1 sobre socket RFCOMM propio | Excepciones y mensajes sueltos |
| 2 | `Ejemplo_MotorDSL` | `com.ejemplos.devices.MotorDSL` | **Motor DSL** (`MotorDsl.*` 1.0.12): JSON → render ESC/POS bitmap | Un `Label` de estado; `try/catch` que vuelca tipo/mensaje/inner/stack |
| 3 | `Ejemplo_MotorDSL_Dialog` | `com.ejemplos.devices.MotorDSL.Dialog` | Motor DSL (`MotorDsl.*` **1.0.13**) | **Overlay** con catálogo de errores codificado y botonera por causa |

El salto 1→2 cambia *quién produce los bytes*; el salto 2→3 cambia *qué ve el usuario cuando algo falla*. El tercero es el que se portó a la app híbrida.

Hardware de referencia: impresora térmica portátil Bluetooth 58 mm **58HB6** (papel 58 mm, 384 puntos de ancho).

## 2. Proyecto 1 — `Ejemplo_ThermalPrinter`: ESC/POS a mano

Paquetes: `ESCPOS_NET` 2.2.1 + `BarcodeScanner.Mobile.Maui` 9.0.1 (el proyecto además lee QR).

- `Services/IThermalPrinterService.cs` (117 líneas) — contrato de alto nivel orientado al caso de uso: `ScanDevicesAsync`, `ConnectAsync(mac)`, `PrintTextAsync(text, fontSize, bold, centered)`, `PrintReceiptAsync(storeName, items, subtotal, tax, total, footer)`, `PrintBarcodeAsync`, `PrintQRCodeAsync`, `PrintImageAsync`, `FeedLinesAsync`, `CutPaperAsync`.
- `Services/ThermalPrinterService.cs` (406 líneas) — implementación **enteramente bajo `#if ANDROID`**; en iOS cada método degrada a `Task.CompletedTask`.

Lo que enseña de bajo nivel:

| Detalle | Dónde |
|---------|-------|
| El socket es **RFCOMM/SPP** con el UUID estándar `00001101-0000-1000-8000-00805F9B34FB` | `:93-97` |
| Conexión: `GetRemoteDevice(mac)` → `CreateRfcommSocketToServiceRecord(uuid)` → `Connect()` en `Task.Run` | `:85-99` |
| Un texto con estilo es una **secuencia de comandos** (`CodePage(PC858_EURO)`, alineación, `SetStyles(FontB/Bold)`, `Print`, reset) unida con `ByteSplicer.Combine` | `:148-207` |
| Ancho de papel como constante de puntos: `PAPER_WIDTH_58MM = 384` | `:20` |

## 3. Proyectos 2 y 3 — el motor DSL

`MotorDsl.*` es un conjunto de **7 paquetes NuGet** (no hay `ProjectReference`: se consumen publicados, con el comentario explícito «*NuGet packages (instead of ProjectReference) — validates published packages*»):

`MotorDsl.Core` · `.Parser` · `.Rendering` · `.Extensions` · `.Printing.Abstractions` · `.Bluetooth` · `.Maui`

| Proyecto | Versión de `MotorDsl.*` |
|----------|------------------------|
| `Ejemplo_MotorDSL` | **1.0.12** |
| `Ejemplo_MotorDSL_Dialog` | **1.0.13** (igual que la app híbrida) |

Arranque idéntico en ambos (`MauiProgram.cs`):

```csharp
builder.Services.AddMotorDslEngine()
    .AddProfiles(p => p.Add(new DeviceProfile("thermal_58mm", 32, "escpos-bitmap")))
    .AddMotorDslMaui();
builder.Services.AddBluetoothPrinterTransport();   // Android Classic SPP
```

### 3.1 El documento: `MultaIntegratedDsl`

`Samples/MultaIntegratedDsl.cs` (132 líneas, **presente en ambos proyectos**) es un acta de infracción municipal en **formato integrado** (`"format": "integrated"`): el JSON viene con todos los valores ya resueltos — sin `{{placeholders}}`, sin `loop` ni `conditional`, con el `loop` de infracciones ya expandido en dos `container` concretos y las imágenes en base64. Por eso el motor lo procesa con `Render(json, profile)` directamente, **sin etapa de `Evaluate`**.

Estructura del árbol: `root` es un `container` vertical con nodos `image` (logo BMP base64, firma), `text` con `style` (`align`, `bold`), y contenedores anidados.

⚠️ En `Ejemplo_MotorDSL_Dialog` el archivo declara `namespace Ejemplo_MotorDSL.**Templates**` (no `Ejemplo_MotorDSL_Dialog.Samples`), a pesar de vivir en `Samples/`. Por eso `MainViewModel.cs:7` importa `Ejemplo_MotorDSL.Templates`.

### 3.2 El perfil de dispositivo

Ambos proyectos construyen el perfil **en el punto de render**, distinto del que registran en el contenedor:

```csharp
var profile = new DeviceProfile("58HB6", 32, "escpos-bitmap");
profile.SetCapability("supports_bitmap", true);
profile.SetCapability("bitmap_max_width_px", 320);
profile.SetCapability("bitmap_binarization_threshold", 128);
var render = _engine.Render(MultaIntegratedDsl.Document, profile);
```

(`Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:129-133` · `Ejemplo_MotorDSL_Dialog/ViewModels/MainViewModel.cs:33-38`.) El registrado en DI es `("thermal_58mm", 32, "escpos-bitmap")`.

### 3.3 Invariante compartida: **renderizar siempre primero**

Los dos proyectos comentan la misma regla: *«Render SIEMPRE primero, antes de tocar la impresora»* — así un fallo de documento se diagnostica sin depender del hardware. En el proyecto 2 se ve como un `if (!_printer.IsConnected)` **después** del render (`:150-155`); en el 3, como el caso `RenderFallido` del catálogo, previo a pedir permisos.

### 3.4 Diferencia de UI entre 2 y 3

| | `Ejemplo_MotorDSL` | `Ejemplo_MotorDSL_Dialog` |
|---|---|---|
| Patrón | Code-behind con eventos | MVVM (`CommunityToolkit.Mvvm`, `[ObservableProperty]`, `[RelayCommand]`) |
| Selección de impresora | Controles de la librería: `muic:PrinterStatusBadge` y `muic:PrinterPickerView` (`FilterKind="bluetooth"`), ambos `IsVisible="{OnPlatform Android=True, iOS=False}"` | Overlay propio con botonera dinámica |
| Permisos BT | `ActivityCompat.RequestPermissions` + **`await Task.Delay(3000)`** y re-chequeo (`:74-77`); en Android < 12, `LocationWhenInUse` | `BluetoothPermissions : Permissions.BasePlatformPermission`, awaitable |
| Feedback | `MessageLabel.Text` con hora | Capa de espera / capa de error del overlay |
| DI de la página | `AddTransient<MainPage>()` | `AddSingleton<MainPage>` + `AddSingleton<MainViewModel>` + `AddSingleton<PrinterOverlayViewModel>` + `AddSingleton<PrinterService>` |

## 4. El overlay de estado (proyecto 3)

Tres piezas reutilizables, pensadas como base común para cualquier dispositivo:

| Pieza | Archivo | Rol |
|-------|---------|-----|
| `StatusOverlayViewModel` (abstracto) | `ViewModels/StatusOverlayViewModel.cs` (78 l.) | Máquina de tres estados `None / Busy / Error`; `ShowBusy`, `ShowError`, `Hide` |
| `StatusOverlayView` | `Controls/StatusOverlayView.xaml` (62 l.) | Capa de espera (imagen animada + título + subtítulo) y capa de error (ícono + textos + botonera) |
| `OverlayAction` | `record (string Text, ICommand Command, OverlayActionStyle Style = Primary)` | Un botón de la botonera |

La botonera se genera con `BindableLayout.ItemsSource="{Binding Actions}"`; el estilo `Secondary` se aplica por `DataTrigger` sobre `Style`. Convención documentada en el código (`PrinterOverlayViewModel.cs:293-301`): **cuando «Cerrar» es la única acción, va como `Primary`** — un único botón en gris comunica «esto no importa» siendo lo único que se puede hacer.

Íconos: fuente `MaterialIconsOutlined` con glifos `error`, `print`, `print_disabled`, `bluetooth_disabled`, `block`. Espera: `timer.gif`.

## 5. `PrinterService` — la capa de servicio (proyecto 3)

`Services/PrinterService.cs` (206 líneas). Compone permisos + descubrimiento + conexión + envío sobre `IThermalPrinterService` de la librería y **devuelve resultados tipados**; no referencia ningún ViewModel.

| Método | Devuelve | Nota |
|--------|----------|------|
| `IsSupported` | `bool` | `true` solo en Android (BT Classic SPP) |
| `EnsurePermissionsAsync()` | `BluetoothPermissionResult` | `CheckStatusAsync` → `RequestAsync` → `ShouldShowRationale` |
| `DiscoverAsync()` | `DiscoverResult` | Ver abajo |
| `GetDefaultIfPresent(devices)` | `PrinterDevice?` | Predeterminada guardada **si está en la lista descubierta** |
| `ConnectAsync(device)` | `bool` | Al conectar con éxito **guarda la predeterminada** |
| `SendAsync(bytes)` | `PrintResult` | Clasifica el fallo con el catálogo |
| `SaveDefault` / `ClearDefault` / `GetAlias` / `SetAlias` | — | Persistencia en `Preferences` (`default_printer_id`, `default_printer_name`, `printer_alias_{id}`) |
| `OpenAppSettings()` / `OpenBluetoothSettings()` | — | Dos destinos distintos: ajustes de la app vs. panel de Bluetooth del SO |

**Por qué `DiscoverAsync` chequea el adaptador antes de descubrir** (comentario en `:56-66`): el transport sí distingue «Bluetooth apagado» de «permiso revocado» y lanza, pero `ThermalPrinterService` (de la librería) **captura esas excepciones y devuelve lista vacía** por diseño — un transport caído no debe abortar el barrido de los demás. Sin el chequeo previo, ambas causas llegarían a la UI como «no se encontraron impresoras», mandando al usuario a revisar la impresora cuando el problema está en el teléfono. Por eso usa `BluetoothManager` (no `BluetoothAdapter.DefaultAdapter`, deprecado en API 31+) y `CheckSelfPermission(BLUETOOTH_CONNECT)`.

**Por qué no se puede encender el Bluetooth desde la app** (`:181-190`): `BluetoothAdapter.Enable()` está deprecado desde Android 13 (API 33) y falla en silencio, y `ActionRequestEnable` exige una `Activity` para el resultado, que no encaja con el patrón overlay. Se abre el panel y decide el usuario.

Modelos tipados (`Models/`): `DiscoverResult` (`Found` / `Empty` / `BluetoothOff` / `PermissionRevoked` / `NotSupported`), `PrintResult` (`Success` / `Failure`), `BluetoothPermissionResult` (`Granted` / `DeniedCanRetry` / `Denied` / `Restricted`), `PrintFailure`.

> El comentario de `DiscoverResult.cs:34-38` deja una regla de diseño: **cada variante debe ser alcanzable**. `BluetoothOff` existió todo el PoC sin que nadie la construyera nunca (siempre llegaba `Empty`); lo que la hizo alcanzable fue el chequeo previo del adaptador.

## 6. Catálogo de errores: `PrinterErrorCatalog`

`Models/PrinterErrorCatalog.cs` (153 líneas). Traduce cada causa a un `PrintFailure(Code, Title, UserMessage, TechnicalMessage, Exception?)`.

Dos reglas declaradas en el propio archivo (`:8-15`):
1. **Los códigos son contrato con soporte**: una vez que un usuario reportó un `PRN-HW-PAPER`, renombrarlo rompe el historial. Se agregan códigos; no se renombran.
2. **Un código por acción distinta del usuario**, no por causa técnica distinta: dos causas que se resuelven con el mismo gesto comparten código (por eso `Protocol` y `Unknown` comparten `PRN-UNKNOWN`).

| Código | Título | Origen |
|--------|--------|--------|
| `PRN-DOC-NET` | Sin conexión | No se pudo obtener el comprobante (caso de la híbrida, no de este PoC) |
| `PRN-DOC-CONTRACT` | Comprobante inválido | Formato inesperado |
| `PRN-DOC-RENDER` | No se pudo generar el documento | `render.IsSuccessful == false` |
| `PRN-BT-OFF` | Bluetooth desactivado | `adapter.IsEnabled == false` |
| `PRN-BT-PERM` | Permiso Bluetooth revocado | `CheckSelfPermission(BLUETOOTH_CONNECT) != Granted` |
| `PRN-DEV-NONE` | No se encontraron impresoras | `DiscoverDevicesAsync` devolvió 0 |
| `PRN-DEV-ABSENT` | La impresora no responde | Falló conectar con la **predeterminada** |
| `PRN-CONN-FAIL` | No se pudo conectar | Falló conectar con una elegida por el usuario |
| `PRN-HW-PAPER` | Sin papel | `Hardware` + el mensaje contiene «paper» |
| `PRN-HW-COVER` | Tapa abierta | `Hardware` + el mensaje contiene «cover» |
| `PRN-HW-OTHER` | Problema en la impresora | `Hardware` sin causa reconocida |
| `PRN-LINK-LOST` | Se perdió la conexión | `PrintErrorType.Connection` |
| `PRN-TIMEOUT` | La impresora no responde | `PrintErrorType.Timeout` |
| `PRN-UNKNOWN` | Falló la impresión | `Protocol` / desconocido |

`PrintFailure.DisplayMessage` = `UserMessage` + línea en blanco + `Code`: el código va **separado y al final** para que el usuario pueda dictárselo a soporte sin que compita con la acción que tiene que ejecutar (`PrintFailure.cs:20-25`).

`Describe(original, technical)` reusa `PrintError.FromException` **de la librería** en vez de reimplementar la clasificación. Dos advertencias declaradas:
- `ThermalPrinterService` envuelve el fallo real en un `Exception` genérico («Print failed after N attempt(s): …») y conserva la causa en `InnerException`; es esa causa la que hay que pasar (`PrinterService.cs:146-151`).
- `Hardware()` es el **único punto que depende de literales de la librería** («paper» / «cover», de `BluetoothPrinterTransport.Detection.cs`). Si esos textos cambian degrada a `PRN-HW-OTHER`: se pierde precisión, no corrección.

## 7. El flujo de impresión y sus salidas por causa

`PrinterOverlayViewModel` (337 líneas) orquesta:

```
ImprimirAsync(render)
  ├─ render fallido        → PRN-DOC-RENDER + [Cerrar]
  ├─ !IsSupported          → "Impresión no disponible" + [Cerrar]
  ├─ permisos              → MostrarPermiso(...)
  └─ BuscarYImprimirAsync()
       ├─ Found  → predeterminada presente? → ConectarEImprimir(esPredeterminada: true)
       │         → una sola?                → ConectarEImprimir(esPredeterminada: false)
       │         → varias                   → MostrarSelector
       ├─ Empty            → [Reintentar] [Emparejar impresora] [Cerrar]
       ├─ BluetoothOff     → [Activar Bluetooth] [Reintentar] [Cerrar]
       ├─ PermissionRevoked→ [Abrir configuración] [Reintentar] [Cerrar]
       └─ NotSupported     → [Cerrar]
             ↓
       ConectarEImprimirAsync → falla → esPredeterminada ? PredeterminadaAusente : NoSePudoConectar
             ↓ ok
       EnviarAsync → Success → Hide()   |   Failure → MostrarFalloImpresion(fallo)
```

Decisiones de UX documentadas en el código, todas con su razón:

| Decisión | Razón (del propio comentario) |
|----------|------------------------------|
| `_lastEraPredeterminada` se guarda para el reintento | Sin eso, el segundo fallo se diagnosticaría como conexión común y el usuario perdería la salida «Olvidar y emparejar otra», que es justo la que necesita si cambió de impresora |
| `ElegirOtra` llama a `BuscarYImprimirAsync(forzarSelector: true)` | Sin forzar, se volvía a descubrir, se volvía a encontrar la predeterminada en `BondedDevices` y se volvía a conectar con la misma: **un bucle sin salida por UI** |
| «Elegir otra impresora» solo aparece si `_lastDevices.Count > 1` | No tiene sentido ofrecer cambiar si no hay otra emparejada |
| El botón primario refleja el **gesto físico**: «Ya cargué papel — Reintentar», «Ya la cerré — Reintentar» | Convierte el botón en la confirmación de un gesto en lugar de una repetición ciega que va a fallar idéntico |
| Sufijo de MAC (últimos 2 octetos) solo ante nombres repetidos | Los nombres BT de las térmicas genéricas son el **modelo**, no una identidad: dos 58HB6 dan dos botones idénticos |
| `PedirPermiso` reusa `_lastRender` | Reintenta el permiso sin rehacer el render |
| «predeterminada presente» = **emparejada**, no encendida | El descubrimiento solo lista `BondedDevices`; el fallo recién aparece al conectar |

## 8. Documentación del hardware (`Ejemplo_Docs_Printer/`)

| Carpeta / archivo | Contenido |
|-------------------|-----------|
| `Manuales/01_MANUAL_USUARIO.md` (428 l.) | Guía operativa no técnica: LEDs, botón multifunción, carga de papel, emparejamiento, 6 problemas frecuentes, mantenimiento, garantía |
| `Manuales/02_MANUAL_TECNICO_ESC_POS.md` (571 l.) | Referencia ESC/POS del 58HB6 con arquitectura y diagrama |
| `Manuales/03_GUIA_INTEGRACION_ESCPOS_NET.md` (1.158 l.) | Guía de integración con `ESCPOS_NET` — el documento más extenso del repositorio |
| `Manuales/04_Borrador_Manual.md` (735 l.) | Borrador previo |
| `Prompts_Generacion_Apk_ejemplo/` | Los prompts y los archivos generados por IA (`ficheros_claude.ia/`) con los que se construyó el primer APK: `Thermalprinterservice.cs`, `Ithermalprinterservice.cs`, `MainPage.XAML{,.cs}`, `MauiProgram.cs`, `Androidmanifest.XML`, un `.csproj` **net8.0-android34.0** con `ESCPOS_NET` 4.5.0 y `Plugin.BLE` 3.1.0 |
| `Prompts_generacion_documentacion/` | Los prompts (ChatGPT, Copilot) con los que se generaron los manuales |
| `Pruebas_realizadas/` | Evidencia real: `ejemplo_impresion.jpg` y `ejemplo_print_thermal.gif` (este último es la portada del `README.md` raíz) |
| `Ejemplo_MotorDSL/flowchat.{png,bmp}` | Diagrama del flujo del motor DSL |

> Los `ficheros_claude.ia/` son **material histórico del prompt**, no código compilable del repositorio: usan `net8.0`, `ESCPOS_NET` 4.5.0 y `Plugin.BLE`, mientras el proyecto real quedó en net10 con `ESCPOS_NET` 2.2.1 y socket RFCOMM propio.

## 9. Observaciones

- **`Ejemplo_ThermalPrinter` no está en ningún workflow de CI**; sí lo está `Ejemplo_MotorDSL` (`cd-ios-printer.Ejemplo_MotorDSL.yml`) y **no** `Ejemplo_MotorDSL_Dialog` ([índice 09](09_CI-CD-y-Build.md)).
- Los tres proyectos son **solo Android en la práctica**: en iOS los controles del picker se ocultan por `OnPlatform`, `PrinterService.IsSupported` es `false` y `ThermalPrinterService` degrada a no-op.
- `Ejemplo_MotorDSL` deja comentado el registro de templates (`.AddTemplates(t => t.Add("acta-infraccion-integrada", …))`, `MauiProgram.cs:30-33`): el documento integrado se pasa directo al `Render`, sin registrar.
- El PoC del proyecto 3 **no tiene el caso «no se pudo obtener el documento»** (documentado en `PrinterOverlayViewModel.cs:17-20`): renderiza una muestra local, así que el documento siempre está disponible. Los códigos `PRN-DOC-NET` / `PRN-DOC-CONTRACT` existen para la app híbrida, que sí lo trae por red.

## 10. Fuentes

| Ruta | Contenido |
|------|-----------|
| `Printer/Ejemplo_ThermalPrinter/Services/` | ESC/POS a mano, socket RFCOMM/SPP, contrato de alto nivel |
| `Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml{,.cs}` | Motor DSL + controles `PrinterStatusBadge`/`PrinterPickerView` de la librería |
| `Printer/Ejemplo_MotorDSL*/Samples/MultaIntegratedDsl.cs` | El documento JSON integrado del acta de infracción |
| `Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs` | Permisos, descubrimiento, conexión, envío y predeterminada |
| `Printer/Ejemplo_MotorDSL_Dialog/Models/PrinterErrorCatalog.cs` | Los 14 códigos y sus mensajes |
| `Printer/Ejemplo_MotorDSL_Dialog/ViewModels/PrinterOverlayViewModel.cs` | El flujo completo y la botonera por causa |
| `Printer/Ejemplo_MotorDSL_Dialog/{Controls,ViewModels}/StatusOverlay*` | Overlay reutilizable de tres estados |
| `Printer/Ejemplo_Docs_Printer/Manuales/` | Manual de usuario, ESC/POS y guía de integración del 58HB6 |
