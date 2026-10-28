# 03 — Impresión térmica Bluetooth

> **Propósito**: mapa del dominio «impresión térmica Bluetooth 58 mm» del repo `Ejemplos_Maui_Devices`: los tres proyectos MAUI de `Ejemplos_Devices/Printer/` (ESC/POS directo con ESCPOS_NET, motor DSL con `MotorDsl.*`, y motor DSL con overlay MVVM y catálogo de errores) más la carpeta de documentación `Ejemplo_Docs_Printer`.
> **Fuente primaria**: `Ejemplos_Devices/Printer/**` en el commit `285f6fb`. Contexto: `Ejemplos_Devices/Ejemplos_Devices.slnx` y `.github/workflows/cd-ios-printer.Ejemplo_MotorDSL.yml`.
> **Fuera de alcance**: la copia del flujo de impresión dentro de la app híbrida (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/MotorDSL/`) y sus tests → ver [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).
> Las rutas son relativas a la raíz del repo de código; `Printer/` abrevia `Ejemplos_Devices/Printer/`.

---

## §1 Mapa de proyectos del dominio

| Proyecto | Técnica | Archivos clave | En `.slnx` | Workflow CI |
|---|---|---|---|---|
| `Ejemplo_ThermalPrinter` | ESC/POS «a mano» con **ESCPOS_NET 2.2.1** (emisor `EPSON`) sobre socket RFCOMM SPP propio (Android) | `Services/IThermalPrinterService.cs`, `Services/ThermalPrinterService.cs`, `Pages/MainPage.xaml(.cs)` | Sí (`Ejemplos_Devices.slnx:96`) | No |
| `Ejemplo_MotorDSL` | Motor **MotorDsl.\* 1.0.12**: JSON integrado → raster ESC/POS; controles de la librería (`PrinterPickerView`, `PrinterStatusBadge`); code-behind | `MauiProgram.cs`, `Pages/MainPage.xaml(.cs)`, `Samples/MultaIntegratedDsl.cs` | Sí (`Ejemplos_Devices.slnx:90`) | Sí: `cd-ios-printer.Ejemplo_MotorDSL.yml` (sólo `workflow_call`, ver §9) |
| `Ejemplo_MotorDSL_Dialog` | Motor **MotorDsl.\* 1.0.13** + **overlay MVVM** (CommunityToolkit) + resultados tipados + catálogo de errores `PRN-*` | `Services/PrinterService.cs`, `ViewModels/PrinterOverlayViewModel.cs`, `Models/PrinterErrorCatalog.cs`, `Controls/StatusOverlayView.xaml` | Sí (`Ejemplos_Devices.slnx:93`) | No |
| `Ejemplo_Docs_Printer` | Documentación (prompts, manuales generados por IA, pruebas) — no es proyecto compilable | `Readme.md`, `Manuales/*.md`, `Prompts_*/*`, `Pruebas_realizadas/Readme.md` | Como `<File>` en carpetas de solución (`Ejemplos_Devices.slnx:100-146`) | — |

Progresión didáctica: `Ejemplo_ThermalPrinter` (protocolo crudo) → `Ejemplo_MotorDSL` (motor + controles listos) → `Ejemplo_MotorDSL_Dialog` (flujo robusto de UX/errores, el que se replica en la híbrida).

---

## §2 Stack y paquetes NuGet (leídos de los `.csproj`)

| Dato | `Ejemplo_ThermalPrinter` | `Ejemplo_MotorDSL` | `Ejemplo_MotorDSL_Dialog` |
|---|---|---|---|
| TFM | `net10.0-android`; `net10.0-ios` sólo si el host es macOS (`csproj:4-5`) | ídem (`csproj:4-5`) | ídem (`csproj:4-5`) |
| Min OS | Android 25 / iOS 15.0 (`csproj:24-25`) | ídem | ídem |
| `ApplicationId` | `com.ejemplos.devices.ThermalPrinter` (`csproj:17`) | `com.ejemplos.devices.MotorDSL` (`csproj:17`) | `com.ejemplos.devices.MotorDSL.Dialog` (`csproj:17`) |
| `Microsoft.Maui.Controls` | `$(MauiVersion)` (`csproj:103`) | `$(MauiVersion)` (`csproj:79`) | **`10.0.60` fijo** (`csproj:79`) |
| `Microsoft.Extensions.Logging.Debug` | 10.0.3 | 10.0.3 | 10.0.3 |
| Impresión | `ESCPOS_NET` **2.2.1** (`csproj:105`) | `MotorDsl.Core/Parser/Rendering/Extensions/Printing.Abstractions/Bluetooth/Maui` **1.0.12** (`csproj:85-91`) | los mismos siete `MotorDsl.*` en **1.0.13** (`csproj:90-96`) |
| Otros | `BarcodeScanner.Mobile.Maui` 9.0.1 (`csproj:102`, sin uso en el código, ver §10) | — | `CommunityToolkit.Maui` 14.2.0, `CommunityToolkit.Maui.Core` 14.2.0, `CommunityToolkit.Mvvm` 8.4.2 (`csproj:84-86`) |
| Fuentes | OpenSans | OpenSans + `MaterialIconsOutlined-Regular.otf` (`MauiProgram.cs:24`) | ídem (`MauiProgram.cs:34`) |

- No hay `Directory.Build.props`, `Directory.Packages.props` ni `global.json` versionados: `$(MauiVersion)` lo resuelve el workload MAUI instalado (no verificado en el repo).
- La híbrida consume `MotorDsl.*` 1.0.13 (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Ejemplo_Maui_Hibrida.csproj:116-122`): `Ejemplo_MotorDSL` es el único en 1.0.12.
- El comentario de `Printer/Ejemplo_MotorDSL/Ejemplo_MotorDSL.csproj:83` explica la elección de `PackageReference` en vez de `ProjectReference`: «validates published packages».

---

## §3 Arquitectura por variante

### §3.1 `Ejemplo_ThermalPrinter` — ESC/POS directo

```
MainPage (code-behind, eventos Clicked)
   │  IThermalPrinterService (interfaz propia del proyecto)
   ▼
ThermalPrinterService  (#if ANDROID)
   ├─ BluetoothAdapter.DefaultAdapter      → BondedDevices (no hay escaneo real)
   ├─ CreateRfcommSocketToServiceRecord(UUID SPP 00001101-…)
   ├─ ESCPOS_NET.Emitters.EPSON            → byte[] por comando
   └─ ByteSplicer.Combine → OutputStream.WriteAsync + FlushAsync
```

- DI: `AddSingleton<IThermalPrinterService, ThermalPrinterService>()` y `AddTransient<MainPage>()` (`Printer/Ejemplo_ThermalPrinter/MauiProgram.cs:33-35`).
- Todo el código de dispositivo está bajo `#if ANDROID`; en otras plataformas los métodos devuelven lista vacía / `false` / `Task.CompletedTask` (`Services/ThermalPrinterService.cs:69-72`, `112-115`).
- La UI habilita y deshabilita botones según el estado de conexión (`Pages/MainPage.xaml.cs:302-340`) y escribe un log de una línea con hora en `MessageLabel` (`:342-345`).

### §3.2 `Ejemplo_MotorDSL` — motor DSL con controles de la librería

```
MainPage(IDocumentEngine, IThermalPrinterService)      ← ambos de MotorDsl.*
   ├─ PrinterStatusBadge / PrinterPickerView  (MotorDsl.Maui.Controls, sólo Android)
   ├─ OnImprimirClicked:
   │     1. DeviceProfile("58HB6", 32, "escpos-bitmap") + capabilities
   │     2. _engine.Render(MultaIntegratedDsl.Document, profile) → RenderResult
   │     3. si Output is byte[] y _printer.IsConnected → SendBytesAsync(bytes)
   └─ errores: volcado de tipo/mensaje/inner/stack en MessageLabel + Console
```

- Registro (`Printer/Ejemplo_MotorDSL/MauiProgram.cs:29-41`): `AddMotorDslEngine()` → `.AddProfiles(p => p.Add(new DeviceProfile("thermal_58mm", 32, "escpos-bitmap")))` → `.AddMotorDslMaui()`; luego `AddBluetoothPrinterTransport()` («Android Classic SPP», comentario `:40`). El registro de template `AddTemplates(...)` está **comentado** (`:30-33`).
- El orden «render siempre primero, después impresora» es deliberado: permite diagnosticar el DSL sin hardware (`Pages/MainPage.xaml.cs:127`, `150-154`).

### §3.3 `Ejemplo_MotorDSL_Dialog` — servicio tipado + overlay MVVM

```
MainPage(MainViewModel)  ── Button → ImprimirEjemploCommand
   │                         └─ StatusOverlayView  BindingContext = PrinterOverlayViewModel
MainViewModel.ImprimirEjemplo
   ├─ _engine.Render(MultaIntegratedDsl.Document, profile "58HB6")
   └─ PrinterOverlayViewModel.ImprimirAsync(render)
          │  (máquina None / Busy / Error de StatusOverlayViewModel)
          ▼
PrinterService (sin referencias a ViewModels)
   ├─ EnsurePermissionsAsync → BluetoothPermissionResult
   ├─ DiscoverAsync          → DiscoverResult (Found / Empty / BluetoothOff / PermissionRevoked / NotSupported)
   ├─ ConnectAsync           → bool (+ guarda predeterminada en Preferences)
   └─ SendAsync              → PrintResult (Success / Failure(PrintFailure))
          ▼
MotorDsl.Printing.IThermalPrinterService (librería)
```

- DI (`Printer/Ejemplo_MotorDSL_Dialog/MauiProgram.cs:60-77`): `PrinterService`, `PrinterOverlayViewModel`, `MainViewModel` y `MainPage` como **singletons**. Activa `UseMauiCommunityToolkit()` y `UseMauiCommunityToolkitCore()` (`:24-25`).
- Mismo registro del motor que §3.2 (`MauiProgram.cs:41-49`), pero acá no está ni siquiera el bloque comentado de `AddTemplates`; el comentario «El template registrado es un JSON integrado» (`:40`) quedó igual.
- Los comentarios presentan cada pieza como «espejo» de su par en otros dominios (`CallService/GpsService`, `CallOverlayViewModel`, `CallResult`): `Services/PrinterService.cs:9-12`, `ViewModels/PrinterOverlayViewModel.cs:13`, `Models/DiscoverResult.cs:5`.
- `README.md` del proyecto sólo contiene el fragmento de `MauiProgram` que hay que copiar (`Printer/Ejemplo_MotorDSL_Dialog/README.md:3-39`).
- `Pages/OverlayBlueToothThermalPrintPage.xaml` es una página vacía, sin ruta ni registro en DI (`:1-8`).

---

## §4 APIs y servicios clave

### §4.1 `IThermalPrinterService` propio (`Ejemplo_ThermalPrinter`)

| Miembro | Implementación real (`Services/ThermalPrinterService.cs`) |
|---|---|
| `ScanDevicesAsync()` | Devuelve `BondedDevices` (emparejados); lanza si no hay adaptador o está apagado (`:44-73`) |
| `ConnectAsync(address)` | `GetRemoteDevice` → socket RFCOMM con UUID SPP (`:93-97`) → envía `Initialize()` (`:102`) |
| `DisconnectAsync()` | Flush + Dispose del stream y del socket (`:118-146`) |
| `PrintTextAsync(text, fontSize, bold, centered)` | `CodePage(PC858_EURO)`, alineación, `FontB` si `fontSize > 1`, `Bold` (`:148-193`) |
| `PrintReceiptAsync(...)` | Encabezado, fecha, ítems a 32 columnas, subtotal/impuesto/total, pie, `FeedLines(3)` + `FullCutAfterFeed(5)` (`:195-279`) |
| `PrintBarcodeAsync` / `PrintQRCodeAsync` | **Imprimen el dato como texto** (`:281-299`, `:301-323`) |
| `PrintImageAsync` | `NotImplementedException` envuelta (`:325-344`) |
| `FeedLinesAsync` / `CutPaperAsync` | `FeedLines((byte)lines)` / `FullCutAfterFeed(5)` (`:346-366`) |

Tipos auxiliares: `BluetoothDevice { Name, Address, IsPaired }` y `enum BarcodeType` de 9 valores (`Services/IThermalPrinterService.cs:91-117`).

### §4.2 API de `MotorDsl.*` usada (observada en el código; la librería no está en este repo)

| Símbolo | Dónde se usa |
|---|---|
| `IDocumentEngine.Render(string json, DeviceProfile)` → `RenderResult` (`IsSuccessful`, `Output`, `Errors`, `Warnings`) | `Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:133-143`; `Printer/Ejemplo_MotorDSL_Dialog/ViewModels/MainViewModel.cs:38` |
| `DeviceProfile(id, columnas, renderer)` + `SetCapability("supports_bitmap" \| "bitmap_max_width_px" \| "bitmap_binarization_threshold", …)` | `Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:129-132` |
| `MotorDsl.Printing.IThermalPrinterService`: `IsConnected`, `SendBytesAsync`, `DiscoverDevicesAsync(kind:)`, `ConnectAsync(PrinterDevice)` | `Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:150-157`; `Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:98`, `131`, `141` |
| `PrinterDevice { Id, Name }` | `Ejemplo_MotorDSL_Dialog/ViewModels/PrinterOverlayViewModel.cs:276-291` |
| `PrintError.FromException(ex, 1, 1)` → `PrintErrorType` (`Hardware`, `Connection`, `Timeout`, resto) | `Ejemplo_MotorDSL_Dialog/Models/PrinterErrorCatalog.cs:107-128` |
| Controles `PrinterPickerView` (`Service`, `FilterKind`, `ScanAsync`, eventos `DeviceSelected`/`ScanError`) y `PrinterStatusBadge` (`Service`) | `Ejemplo_MotorDSL/Pages/MainPage.xaml:11-13`, `MainPage.xaml.cs:36-39` |

Según los comentarios, el `ThermalPrinterService` de la librería se traga las excepciones de los transports y devuelve lista vacía, y envuelve los fallos de envío en `Exception("Print failed after N attempt(s): …")` con la causa en `InnerException` (`Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:58-66`, `146-149`). Como la librería no está en el repo, eso queda **(no verificado)**.

### §4.3 Documento DSL de ejemplo (`Samples/MultaIntegratedDsl.cs`)

- Acta de infracción en formato `"integrated"`: todos los valores ya están resueltos, sin `{{placeholders}}`, `loop` ni `conditional`, así que no hay etapa de Evaluate (`:5-9`, `:21-25`).
- Nodos usados: `container` (`layout: vertical`), `text` (`style.align`, `style.bold`), `image` con `imageType: "bitmap"` (data URI base64, `width`) y `imageType: "qrcode"` (`source` = URL) (`:31-127`).
- El mismo archivo existe en los dos proyectos del motor; sólo cambia el namespace (ver §10).

---

## §5 Flujo del overlay y diálogo con el usuario (`Ejemplo_MotorDSL_Dialog`)

`ViewModels/PrinterOverlayViewModel.cs`:

```
ImprimirAsync(render)                                   :44
 ├─ render falló / Output no es byte[] → Error PRN-DOC-RENDER [Cerrar]          :48-55
 ├─ !IsSupported (no Android) → «Impresión no disponible» [Cerrar]              :58-64
 ├─ Busy «Verificando permisos…» → EnsurePermissionsAsync ≠ Granted → MostrarPermiso  :66-68
 └─ BuscarYImprimirAsync(forzarSelector=false)          :79
      ├─ Found: predeterminada presente → conectar (esPredeterminada:true)      :89-90
      │         una sola → conectar directo                                     :91
      │         varias o forzarSelector → MostrarSelector                       :94
      ├─ Empty → PRN-DEV-NONE [Reintentar · Emparejar impresora · Cerrar]       :97-103
      ├─ BluetoothOff → PRN-BT-OFF [Activar Bluetooth · Reintentar · Cerrar]    :105-110
      ├─ PermissionRevoked → PRN-BT-PERM [Abrir configuración · Reintentar · Cerrar] :112-117
      └─ NotSupported → «Impresión no disponible» [Cerrar]                      :119-123
ConectarEImprimirAsync(device, esPredeterminada)        :153
 ├─ falla y era predeterminada → PRN-DEV-ABSENT [Reintentar · (Elegir otra) · Olvidar y emparejar otra · Cerrar] :174-188
 ├─ falla y la eligió el usuario → PRN-CONN-FAIL [Reintentar · Elegir otra · Cerrar] :190-197
 └─ ok → EnviarAsync → Success: Hide() | Failure: MostrarFalloImpresion   :201-211
```

| Decisión de UX | Fuente |
|---|---|
| Primero se reusa la predeterminada (guardada en `Preferences` al conectar con éxito) | `Services/PrinterService.cs:121-134` |
| «Elegir otra» fuerza el selector para no volver a caer en la predeterminada | `PrinterOverlayViewModel.cs:73-78`, `314` |
| Homónimas (dos «58HB6»): se agregan los 2 últimos octetos de la MAC sólo si el nombre se repite; el alias del usuario tiene prioridad | `PrinterOverlayViewModel.cs:129-135`, `276-291`; alias en `PrinterService.cs:172-177` |
| Botón primario que confirma un gesto físico: «Ya cargué papel — Reintentar», «Ya la cerré — Reintentar» | `PrinterOverlayViewModel.cs:213-247` |
| «Cerrar» como única acción se muestra **primario**, no gris | `PrinterOverlayViewModel.cs:293-301` |
| «Reintentar» conserva si el dispositivo era la predeterminada | `PrinterOverlayViewModel.cs:29-32`, `318` |
| «Olvidar y emparejar otra»: `ClearDefault()` + abre los ajustes de Bluetooth del SO | `PrinterOverlayViewModel.cs:326-336`; `PrinterService.cs:166-170` |
| «Pedir permiso» reusa el último `RenderResult`, sin volver a renderizar | `PrinterOverlayViewModel.cs:309-310` |

Base visual reutilizable: `StatusOverlayViewModel` (`OverlayMode { None, Busy, Error }`, `OverlayAction(Text, Command, Style)`, `ShowBusy`, `ShowError`, `Hide`) en `ViewModels/StatusOverlayViewModel.cs:11-77`; la vista `Controls/StatusOverlayView.xaml` tiene una capa Busy (GIF `timer.gif` animado) y una capa Error (glifo Material, título, mensaje y botonera generada con `BindableLayout` + `DataTrigger` para `Secondary`) (`:20-58`).

---

## §6 Catálogo de errores y mensajes al usuario

`Printer/Ejemplo_MotorDSL_Dialog/Models/PrinterErrorCatalog.cs`. Reglas declaradas: los códigos son contrato con soporte, se agregan y nunca se renombran; hay un código por **acción distinta del usuario**, no por causa técnica (`:6-15`).

| Código | Fábrica / origen | Título | Mensaje al usuario (resumen) | Línea |
|---|---|---|---|---|
| `PRN-DOC-NET` | `DocumentoSinConexion` | Sin conexión | Revisá tu conexión y reintentá | `:41-45` |
| `PRN-DOC-CONTRACT` | `DocumentoInvalido` | Comprobante inválido | Avisá a soporte con este código | `:47-51` |
| `PRN-DOC-RENDER` | `RenderFallido` | No se pudo generar el documento | Avisá a soporte con este código | `:53-57` |
| `PRN-BT-OFF` | `BluetoothApagado` | Bluetooth desactivado | Activá el Bluetooth | `:59-63` |
| `PRN-BT-PERM` | `PermisoRevocado` | Permiso Bluetooth revocado | Habilitalo desde los ajustes | `:65-69` |
| `PRN-DEV-NONE` | `SinImpresoras` | No se encontraron impresoras | Encendida y emparejada | `:71-75` |
| `PRN-DEV-ABSENT` | `PredeterminadaAusente(etiqueta)` | La impresora no responde | Apagada/fuera de alcance **o** no es la emparejada | `:83-88` |
| `PRN-CONN-FAIL` | `NoSePudoConectar(etiqueta)` | No se pudo conectar | Encendida y cerca | `:91-95` |
| `PRN-HW-PAPER` | `Describe` → `Hardware`, mensaje contiene «paper» | Sin papel | Cargá un rollo nuevo | `:139-142` |
| `PRN-HW-COVER` | `Describe` → `Hardware`, mensaje contiene «cover» | Tapa abierta | Cerrala y reintentá | `:144-147` |
| `PRN-HW-OTHER` | `Describe` → `Hardware`, otro texto | Problema en la impresora | Revisala y reintentá | `:149-151` |
| `PRN-LINK-LOST` | `Describe`, `PrintErrorType.Connection` | Se perdió la conexión | Encendida y cerca | `:115-117` |
| `PRN-TIMEOUT` | `Describe`, `PrintErrorType.Timeout` | La impresora no responde | Reintentá | `:119-121` |
| `PRN-UNKNOWN` | `Describe`, Protocol y Unknown | Falló la impresión | Reintentá | `:123-126` |

- `PrintFailure(Code, Title, UserMessage, TechnicalMessage, Exception?)`: al usuario sólo le llega `UserMessage`; lo técnico queda para el log (`Models/PrintFailure.cs:3-19`). `DisplayMessage` = mensaje + `\n\n` + código, para que el usuario se lo pueda dictar a soporte (`:20-25`).
- `PRN-DOC-NET` y `PRN-DOC-CONTRACT` no se producen en este PoC: el documento es local (`PrinterOverlayViewModel.cs:17-20`). Los usa la híbrida → [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).
- Distinguir papel de tapa depende de los literales de la librería («paper out» / «cover open»). Si cambian, se cae a `PRN-HW-OTHER` (`PrinterErrorCatalog.cs:135-138`).
- Log: `Debug.WriteLine("[PRINT] …")` en un único punto (`Services/PrinterService.cs:200-205`).
- `PRN-DEV-ABSENT` y `PRN-TIMEOUT` comparten el título «La impresora no responde»: el código y el mensaje son los que los distinguen (`PrinterErrorCatalog.cs:85`, `119`).
- En `DiscoverAsync`, cualquier excepción de `DiscoverDevicesAsync` se informa como `PRN-BT-OFF`. El comentario la llama «red de seguridad, hoy inalcanzable» (`Services/PrinterService.cs:103-109`). `PrinterService.ConnectAsync` no captura excepciones (`:129-134`): si el `ConnectAsync` de la librería lanzara en vez de devolver `false`, la excepción saldría del comando sin pasar por el catálogo **(no verificado: depende de la librería)**.
- Los tests del catálogo (`PrinterErrorCatalogTests.cs`, `PrinterOverlayTests.cs`) ejercitan la copia de la híbrida, no este proyecto → [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

Los otros dos proyectos no usan catálogo: `Ejemplo_ThermalPrinter` muestra `DisplayAlertAsync("Error", $"…: {ex.Message}")` (`Pages/MainPage.xaml.cs:111-115`, etc.) y `Ejemplo_MotorDSL` vuelca tipo, mensaje, inner y stack al `MessageLabel` (`Pages/MainPage.xaml.cs:160-170`).

---

## §7 Permisos y configuración por plataforma

### §7.1 Android — manifiestos

| Permiso | ThermalPrinter | MotorDSL | MotorDSL_Dialog |
|---|---|---|---|
| `BLUETOOTH`, `BLUETOOTH_ADMIN` | sí (`:14-15`) | sí, «legacy API < 31» (`:13-15`) | sí |
| `BLUETOOTH_SCAN` | sí, sin flags (`:16`) | sí, `usesPermissionFlags="neverForLocation"` (`:17-18`) | ídem MotorDSL |
| `BLUETOOTH_CONNECT` | sí | sí | sí |
| `ACCESS_FINE/COARSE_LOCATION` | sí (`:20-21`) | sí (`:22-23`) | sí |
| `<queries>` `REQUEST_ENABLE` | sí | sí | sí |
| `<uses-sdk>` | `min 25 / target 36` (`:30`) | no declara | no declara |

Los manifiestos de `MotorDSL` y `MotorDSL_Dialog` coinciden salvo espacios en blanco.

### §7.2 Android — solicitud en tiempo de ejecución

| Variante | Mecanismo | Fuente |
|---|---|---|
| ThermalPrinter | En `OnAppearing`: `LocationWhenInUse` **siempre**; en API ≥ 31, además `ActivityCompat.RequestPermissions(SCAN, CONNECT)` sin esperar el resultado | `Pages/MainPage.xaml.cs:25-83` |
| MotorDSL | API ≥ 31: `RequestPermissions` + `Task.Delay(3000)` + vuelve a chequear; API < 31: `LocationWhenInUse`. Si se otorgan, `DevicePicker.ScanAsync()` | `Pages/MainPage.xaml.cs:42-46`, `57-107` |
| MotorDSL_Dialog | Permiso propio `BluetoothPermissions : Permissions.BasePlatformPermission` (SCAN + CONNECT en API ≥ 31; `ACCESS_FINE_LOCATION` antes) con `CheckStatusAsync` / `RequestAsync` / `ShouldShowRationale` → `BluetoothPermissionResult { Granted, DeniedCanRetry, Denied, Restricted }`. `DiscoverAsync` vuelve a verificar adaptador (`BluetoothManager`) y `BLUETOOTH_CONNECT` | `Services/BluetoothPermissions.cs:9-20`; `Services/PrinterService.cs:37-54`, `67-92`; `Models/BluetoothPermissionResult.cs` |

En Dialog, `OpenBluetoothSettings()` lanza `Settings.ActionBluetoothSettings` y no enciende el adaptador. El comentario lo justifica así: `BluetoothAdapter.Enable()` está deprecado desde API 33 y `ActionRequestEnable` exige una Activity (`Services/PrinterService.cs:181-198`).

### §7.3 iOS

- Ningún `Info.plist` del dominio declara `NSBluetoothAlwaysUsageDescription` (`Printer/*/Platforms/iOS/Info.plist`). El de Dialog declara cámara y fotos con el comentario «captura de medios» (`Printer/Ejemplo_MotorDSL_Dialog/Platforms/iOS/Info.plist:58-64`).
- No hay impresión en iOS: ThermalPrinter es todo `#if ANDROID`; MotorDSL oculta badge, picker y botón con `OnPlatform iOS=False` (`Pages/MainPage.xaml:11-18`) y en `OnAppearing` iOS no hace nada (`MainPage.xaml.cs:47-48`); Dialog devuelve `IsSupported = false` (`Services/PrinterService.cs:26-31`).
- `Entitlements.plist` y `Entitlements.Development.plist` de los proyectos del motor tienen el mismo contenido (`aps-environment=development`, `get-task-allow=true`); difieren sólo en BOM y espacios. `Ejemplo_ThermalPrinter` no tiene archivos de entitlements en `Platforms/iOS/`.

---

## §8 Diferencias entre variantes

| Aspecto | ThermalPrinter | MotorDSL | MotorDSL_Dialog |
|---|---|---|---|
| Qué se envía | Texto + comandos ESC/POS por llamada | Raster ESC/POS del documento completo | ídem MotorDSL |
| Descubrimiento | `BondedDevices` propio | `PrinterPickerView` de la librería | `DiscoverDevicesAsync(kind:"bluetooth")` + selector en overlay |
| Conexión | Picker + botón «Conectar» | La hace el picker al seleccionar | Automática (predeterminada / única) o selector |
| Patrón UI | Code-behind | Code-behind | MVVM (CommunityToolkit) + overlay |
| Errores | `ex.Message` crudo en alerta | Volcado técnico en etiqueta | Catálogo `PRN-*` con mensajes accionables |
| Perfil | Ancho fijo 32 columnas en código | `DeviceProfile("58HB6", 32, "escpos-bitmap")` local | ídem MotorDSL |
| Persistencia | — | — | `Preferences`: `default_printer_id/name`, `printer_alias_{id}` |
| Interfaz `IThermalPrinterService` | **propia** (`Ejemplo_ThermalPrinter.Services`) | de la librería (`MotorDsl.Printing`) | de la librería |

---

## §9 Solución y CI

- **`.slnx`**: los tres proyectos están en la carpeta `/Ejemplos/Printer/` con `<Deploy />` (`Ejemplos_Devices/Ejemplos_Devices.slnx:89-98`). La documentación se cuelga como `<File>` de `/Ejemplos/Printer/Docs/...` (`:100-146`) y el workflow como archivo de solución (`:213`).
- **Workflow** `.github/workflows/cd-ios-printer.Ejemplo_MotorDSL.yml` (sólo `Ejemplo_MotorDSL`):

| Aspecto | Valor | Línea |
|---|---|---|
| Disparo | `push` comentado; sólo `workflow_call` | `:3-15` |
| Runner | `macos-15` | `:51` |
| Xcode | 26.0 descargado de Google Drive con `gdown` y expandido con `xip` | `:23-26`, `84-122` |
| .NET | SDK `10.0.300`, workloads `ios maui maui-ios` `--version 10.0.100` | `:29-31`, `200` |
| Build | `dotnet build … -c Release -f net10.0-ios -p:RuntimeIdentifier=iossimulator-arm64`, sin firma, entitlements de desarrollo | `:307-318` |
| Firma | ad hoc (`codesign --sign "-"`) de DLL, binario y bundle | `:337-369` |
| Artefactos | `.app.zip` (nombre: fecha + versión de `Info.plist` + `PACKAGE_NAME`) y GIF de evidencia vía `Utilities/simular.sh` | `:384-433` |

- Ningún archivo de `.github/workflows/` invoca otro con `uses: ./.github/workflows/...`, y ningún workflow tiene `workflow_dispatch`. Tal como está versionado, este workflow no tiene disparador activo. El patrón se repite en los demás `cd-ios-*` menos `cd-ios-Integrada.*`, que tiene `push`.
- No hay workflow para `Ejemplo_ThermalPrinter`, para `Ejemplo_MotorDSL_Dialog` ni para Android.

---

## §10 Trampas de lectura y discrepancias ⚠️

| # | ⚠️ Trampa / discrepancia | Evidencia |
|---|---|---|
| 1 | ⚠️ **«Tamaño de fuente» achica en vez de agrandar.** El picker ofrece «Grande (2x)» y «Muy Grande (3x)», pero `fontSize > 1` sólo aplica `PrintStyle.FontB`. El manual del repo describe la Fuente B como «monoespaciado condensado» de 9×17 puntos y 42 columnas, frente a 12×24 y 32 de la A. Además 2x y 3x dan el mismo resultado | `Printer/Ejemplo_ThermalPrinter/Pages/MainPage.xaml:106-108`; `Services/ThermalPrinterService.cs:168-172`; `Printer/Ejemplo_Docs_Printer/Manuales/02_MANUAL_TECNICO_ESC_POS.md:280-303` |
| 2 | ⚠️ **QR y código de barras se imprimen como texto.** Los botones dicen «Imprimir Código QR / de Barras» y la UI informa «impreso correctamente», pero se envía el dato plano; `size` y `barcodeType` se ignoran. La versión original generada por claude.ai sí usaba `Code128(...)` y `QrCode(...)` | `Services/ThermalPrinterService.cs:281-323`; `Pages/MainPage.xaml.cs:235-272`; `Printer/Ejemplo_Docs_Printer/Prompts_Generacion_Apk_ejemplo/ficheros_claude.ia/Thermalprinterservice.cs:294-345` |
| 3 | ⚠️ **El ticket de ejemplo no suma.** Los ítems dan 2×15,50 + 1×25,00 + 3×8,75 = 82,25, pero `subtotal` está fijo en 67,25 (impuesto 10,76 y total 78,01 se calculan sobre 67,25) | `Printer/Ejemplo_ThermalPrinter/Pages/MainPage.xaml.cs:206-215` |
| 4 | ⚠️ **«Escanear» no escanea**: sólo lista los dispositivos emparejados (`BondedDevices`). La UI lo aclara cuando la lista viene vacía | `Services/ThermalPrinterService.cs:54-68`; `Pages/MainPage.xaml.cs:104-108` |
| 5 | ⚠️ **Dos `IThermalPrinterService` distintos** con el mismo nombre: el propio de `Ejemplo_ThermalPrinter` y el de `MotorDsl.Printing` (con `SendBytesAsync`/`DiscoverDevicesAsync`). No son intercambiables | `Printer/Ejemplo_ThermalPrinter/Services/IThermalPrinterService.cs:14`; `Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:4`, `23` |
| 6 | ⚠️ **Comentario «PNG» sobre datos BMP.** `LogoBase64` dice «PNG 1x1 blanco» y `FirmaBase64` «PNG pequeño», pero ambos empiezan con `Qk0` (= `BM`, cabecera BMP), van como `data:image/bmp` y miden 225×90 y 225×38 según la cabecera | `Printer/Ejemplo_MotorDSL/Samples/MultaIntegratedDsl.cs:12-17`, `36`, `121` |
| 7 | ⚠️ **Namespace que no coincide con carpeta ni proyecto**: en Dialog, `Samples/MultaIntegratedDsl.cs` declara `namespace Ejemplo_MotorDSL.Templates` (ni `Samples` ni `Ejemplo_MotorDSL_Dialog`), y `MainViewModel` lo importa así | `Printer/Ejemplo_MotorDSL_Dialog/Samples/MultaIntegratedDsl.cs:2`; `ViewModels/MainViewModel.cs:7` |
| 8 | ⚠️ **El perfil registrado no es el que se usa.** DI registra `thermal_58mm` con `AddProfiles`, pero las dos apps renderizan con un `DeviceProfile("58HB6", …)` armado en el momento. Además el registro del template está comentado y se pasa el JSON directo | `Printer/Ejemplo_MotorDSL/MauiProgram.cs:30-37`; `Pages/MainPage.xaml.cs:129-133`; `Printer/Ejemplo_MotorDSL_Dialog/ViewModels/MainViewModel.cs:33-38` |
| 9 | ⚠️ **Overlay «semitransparente» que no lo es.** El comentario dice que el fondo «deja ver el contenido cargado por detrás», pero el `ContentView` raíz tiene un `BackgroundColor` opaco (White/OffBlack) debajo de las capas `#CC000000` | `Printer/Ejemplo_MotorDSL_Dialog/Controls/StatusOverlayView.xaml:8`, `15`, `21`, `31` |
| 10 | ⚠️ **Selector con texto en plural** («Se detectaron varias impresoras») que también aparece con una sola, cuando se llega por «Elegir otra». «Elegir otra» se ofrece en `PRN-CONN-FAIL` aunque haya una única emparejada (en los otros casos se chequea `_lastDevices.Count > 1`) | `ViewModels/PrinterOverlayViewModel.cs:94`, `143-144`, `195` vs `181`, `236` |
| 11 | ⚠️ **Suscripción de eventos en `OnAppearing`**: `DeviceSelected` y `ScanError` se agregan cada vez que la página aparece (no hay `-=`) | `Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:36-39` |
| 12 | ⚠️ **Permisos por espera fija**: MotorDSL pide los permisos y espera 3 s antes de volver a chequear (sin callback). ThermalPrinter ni siquiera espera el resultado y además pide ubicación también en API ≥ 31 | `Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:72-78`; `Printer/Ejemplo_ThermalPrinter/Pages/MainPage.xaml.cs:40-74` |
| 13 | ⚠️ **NuGet sin uso**: `BarcodeScanner.Mobile.Maui` 9.0.1 está referenciado en ThermalPrinter y ningún `.cs` del dominio lo usa. `Plugin.BLE`, que piden el Readme de prompts y `Instalacion.md`, no está referenciado | `Printer/Ejemplo_ThermalPrinter/Ejemplo_ThermalPrinter.csproj:102`; `Printer/Ejemplo_Docs_Printer/Prompts_Generacion_Apk_ejemplo/Readme.md:60`; `ficheros_claude.ia/Instalacion.md:16-17` |
| 14 | ⚠️ **Material generado por IA ≠ proyecto real**: `ficheros_claude.ia/` usa net8, `ESCPOS_NET` 4.5.0 y `Plugin.BLE` 3.1.0, y su `Androidmanifest.XML` está vacío (0 bytes). El proyecto real es net10 con `ESCPOS_NET` 2.2.1 tras la corrección con Copilot | `ficheros_claude.ia/Thermalprintermaui.csproj:4`, `50-58`; `Prompts_Generacion_Apk_ejemplo/Readme.md:81-86` |
| 15 | ⚠️ **Los manuales sobreafirman.** `Manuales/Readme.md` dice «Compatible con .NET MAUI 10 (Android/iOS)», «Total ~4,500+ líneas» y «ejemplos compilables», pero el servicio es sólo Android y 01+02+03 suman 2157 líneas. `02_MANUAL_TECNICO` titula «Bluetooth 5.0 (BLE)» mientras el código usa RFCOMM/SPP clásico. `Borradores/Readme.md` lista 9 tipos de código de barras «soportados» | `Manuales/Readme.md:204-214`; `Manuales/02_MANUAL_TECNICO_ESC_POS.md:56-60`; `Borradores/Readme.md:144-167`; `ThermalPrinterService.cs:93-95` |
| 16 | ⚠️ **Ruta rota y archivo duplicado en `.slnx`**: la carpeta de solución `…/Ejemplo_ThermalPrinter/Borradores/` apunta a `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md`, que no existe en ningún lado (en Maps sólo hay `Maps/Ejemplo_Docs_Maps/Readme.md`). El archivo real `Printer/Ejemplo_Docs_Printer/Borradores/Readme.md` queda fuera de la solución. Además `Printer/Ejemplo_Docs_Printer/Readme.md` figura dos veces. Es la única ruta del dominio en el `.slnx` que no existe | `Ejemplos_Devices/Ejemplos_Devices.slnx:101`, `108`, `111` |
| 17 | ⚠️ **ADR-0005 (repo hermano) describe mal la línea base**: dice que `Ejemplo_ThermalPrinter` es «ESC/POS directo… sin dependencia externa» y presenta ESCPOS_NET como alternativa «no adoptada como camino principal», pero ese mismo proyecto depende de `ESCPOS_NET` 2.2.1 (`EPSON`, `ByteSplicer`) | `Ejemplos_Maui_Devices.Documentacion/Ejemplos_Maui_Devices-docs/docs/04-decisions/0005-impresion-motor-dsl.md:31-32`; `Printer/Ejemplo_ThermalPrinter/Ejemplo_ThermalPrinter.csproj:105`; `Services/ThermalPrinterService.cs:1-2` |
| 18 | ⚠️ **Workflow sin disparador activo** (ver §9): existe y figura en la solución, pero no hay `push` ni caller | `.github/workflows/cd-ios-printer.Ejemplo_MotorDSL.yml:3-15` |
| 19 | ⚠️ **`{{LogoBase64}}` no es un placeholder del DSL.** El comentario dice «sin `{{placeholders}}`», pero el JSON contiene `{{LogoBase64}}` y `{{FirmaBase64}}`. Es interpolación de C#: el literal es un raw string `$$"""`, así que `{{…}}` se reemplaza al compilar y el motor recibe el base64 ya resuelto | `Printer/Ejemplo_MotorDSL/Samples/MultaIntegratedDsl.cs:6`, `25`, `36`, `121` |

Rasgos deliberados del repo didáctico (no son defectos): `PropertyGroup` de Android e iOS duplicados bajo `<!--resources-->` en ThermalPrinter (`csproj:30-33` = `65-68`, `51-61` = `70-80`); constante `PAPER_WIDTH_58MM` sin uso (`ThermalPrinterService.cs:20`); `PrintImageAsync` sin implementar; `MultaIntegratedDsl.cs` y `Styles.xaml` duplicados entre MotorDSL y Dialog; el texto «Art. Art.» en el JSON de ejemplo (`MultaIntegratedDsl.cs:85`, `94`); página vacía `OverlayBlueToothThermalPrintPage`; `PropertyGroup` extra para `iossimulator-arm64` (`MtouchExtraArgs` con `DYLD_LIBRARY_PATH`, `NoWarn` MT5209/5212/5213) sólo en ThermalPrinter (`csproj:82-86`), aunque ese proyecto no tiene workflow.

---

## §11 Documentación `Ejemplo_Docs_Printer`

| Carpeta / archivo | Contenido | Líneas |
|---|---|---|
| `Readme.md` | Índice de docs + GIF + 2 videos de referencia | 21 |
| `Prompts_Generacion_Apk_ejemplo/Readme.md` | Prompt a claude.ai («construirme el código para imprimir desde maui net por bluetooth»), respuesta y ajuste final con Copilot («no compila…») | 86 |
| `Prompts_Generacion_Apk_ejemplo/ficheros_claude.ia/*` | Salida original de claude.ia (servicio, UI, csproj, manifest vacío, ejemplos avanzados) | ~1850 |
| `Prompts_generacion_documentacion/Readme.md` + `01..03_*.md` | Metodología: ChatGPT define los manuales → Copilot los genera (01–03) → prompt «sin planeamiento» (04) | 71 + 401 |
| `Manuales/01_MANUAL_USUARIO.md` | Usuario final: LED, botón, papel, emparejamiento, problemas | 428 |
| `Manuales/02_MANUAL_TECNICO_ESC_POS.md` | Comandos ESC/POS en hex, fuentes A/B, code pages, QR, buffer, estados | 571 |
| `Manuales/03_GUIA_INTEGRACION_ESCPOS_NET.md` | Integración ESCPOS_NET 2.2.1 en MAUI (servicio, permisos, ejemplos) | 1158 |
| `Manuales/04_Borrador_Manual.md` | Manual unificado no planificado | 735 |
| `Borradores/Readme.md` | Borrador de README de ESC/POS + ESCPOS_NET | 421 |
| `Pruebas_realizadas/Readme.md` | Prueba de `Ejemplo_ThermalPrinter` en Android 13 (GIF + foto del ticket) | 13 |
| `Ejemplo_MotorDSL/flowchat.bmp` / `.png` | Diagrama de flujo del motor (BMP 2768×2392, sin texto indexable) | — |

Modelo de impresora citado: «58HB6-THERMAL-PRINTER» (publicación «58hb4» de MercadoLibre) (`Prompts_generacion_documentacion/Readme.md:47-66`). El mismo nombre se usa como id de `DeviceProfile` en el código.

---

## §12 Preguntas frecuentes

| Pregunta | Sección |
|---|---|
| ¿Qué proyecto conviene mirar para entender ESC/POS crudo? | §1, §3.1, §4.1 |
| ¿Qué paquetes y versiones de `MotorDsl.*` usa cada ejemplo? | §2 |
| ¿Cómo se registra el motor DSL en `MauiProgram`? | §3.2 |
| ¿Cómo es el JSON del documento de ejemplo? | §4.3 |
| ¿Qué pasa si el Bluetooth está apagado o se revocó el permiso? | §5, §6, §7.2 |
| ¿Qué código de error ve el usuario cuando falta papel? | §6 |
| ¿Por qué la impresora predeterminada «ausente» tiene un mensaje con dos causas? | §5, §6 (`PRN-DEV-ABSENT`) |
| ¿Qué permisos Android hacen falta y cómo se piden en cada variante? | §7.1, §7.2 |
| ¿Se puede imprimir desde iOS? | §7.3 |
| ¿Qué workflow compila estos proyectos y cuándo corre? | §9 |
| ¿Por qué el QR sale como texto? | §10 #2 |
| ¿Por qué «Grande (2x)» imprime más chico? | §10 #1 |
| ¿Qué diferencia hay entre `Ejemplo_MotorDSL` y `Ejemplo_MotorDSL_Dialog`? | §3, §8 |
| ¿Dónde está la versión de este flujo que usa la app híbrida y sus tests? | [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) |
| ¿Son confiables los manuales generados por IA? | §10 #14, #15; §11 |

---

## §13 Fuentes

| Ruta | Rol |
|---|---|
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Ejemplo_ThermalPrinter.csproj` | TFM, NuGet ESCPOS_NET 2.2.1 |
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/MauiProgram.cs` | DI |
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/IThermalPrinterService.cs` | Contrato propio |
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/ThermalPrinterService.cs` | RFCOMM + ESC/POS |
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Pages/MainPage.xaml(.cs)` | UI, permisos, ticket |
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Platforms/Android/AndroidManifest.xml` | Permisos |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Ejemplo_MotorDSL.csproj` | MotorDsl 1.0.12 |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/MauiProgram.cs` | Registro del motor y transport |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml(.cs)` | Render + envío, permisos |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Samples/MultaIntegratedDsl.cs` | Documento DSL integrado |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Platforms/{Android,iOS}/*` | Manifest, Info.plist, Entitlements |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Ejemplo_MotorDSL_Dialog.csproj` | MotorDsl 1.0.13, CommunityToolkit |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/MauiProgram.cs` | DI singletons |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/{PrinterService,BluetoothPermissions}.cs` | Servicio tipado, permiso |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/*.cs` | Resultados tipados, catálogo |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/*.cs` | Overlay, base de estados, VM principal |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Controls/StatusOverlayView.xaml` | Vista del overlay |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/README.md` | Fragmento de integración |
| `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/**` | Prompts, manuales, pruebas |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | Pertenencia a la solución |
| `.github/workflows/cd-ios-printer.Ejemplo_MotorDSL.yml` | CI iOS simulador |
| `Ejemplos_Maui_Devices.Documentacion/.../04-decisions/0005-impresion-motor-dsl.md` (repo hermano) | Decisión motor DSL vs ESC/POS |
