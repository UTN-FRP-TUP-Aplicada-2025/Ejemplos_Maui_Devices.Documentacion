# 03 — Impresión térmica Bluetooth

> **Propósito:** cómo imprimen los tres ejemplos de `Ejemplos_Devices/Printer/` en una térmica Bluetooth de 58 mm (modelo de referencia «58HB6»): conexión SPP, comandos ESC/POS directos, motor DSL (JSON → raster ESC/POS), permisos Bluetooth, catálogo de errores y overlay de estado. Indexado sobre el HEAD `cc80ce1` del código.
> **Fuentes primarias:** `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/` · `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/` · `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/` · `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/` (prosa y material generado por IA).
> La versión integrada en la app híbrida (`LibApp/Devices/MotorDSL/`) va en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md); los workflows de iOS (`cd-ios-printer.Ejemplo_MotorDSL.yml`) en [09_CI-CD-y-Build](09_CI-CD-y-Build.md).

## 1. Los tres proyectos de un vistazo

| | `Ejemplo_ThermalPrinter` | `Ejemplo_MotorDSL` | `Ejemplo_MotorDSL_Dialog` |
|---|---|---|---|
| Enfoque | ESC/POS **imperativo** con `ESCPOS_NET` (emisor `EPSON`) | **Motor DSL**: JSON integrado → `byte[]` ESC/POS bitmap | Motor DSL + **servicio tipado → resultado → overlay** (MVVM) |
| TFM | `net10.0-android` (+ `net10.0-ios` sólo en macOS) `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Ejemplo_ThermalPrinter.csproj:4-5` | ídem `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Ejemplo_MotorDSL.csproj:4-5` | ídem `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Ejemplo_MotorDSL_Dialog.csproj:4-5` |
| ApplicationId | `com.ejemplos.devices.ThermalPrinter` (`…/Ejemplo_ThermalPrinter.csproj:17`) | `com.ejemplos.devices.MotorDSL` (`…/Ejemplo_MotorDSL.csproj:17`) | `com.ejemplos.devices.MotorDSL.Dialog` (`…/Ejemplo_MotorDSL_Dialog.csproj:17`) |
| Min OS | Android 25 / iOS 15 (`:24-25` de cada csproj) | ídem | ídem |
| NuGet | `Microsoft.Maui.Controls` `$(MauiVersion)`, `Microsoft.Extensions.Logging.Debug` 10.0.3, `ESCPOS_NET` **2.2.1**, `BarcodeScanner.Mobile.Maui` 9.0.1 (`…/Ejemplo_ThermalPrinter.csproj:102-105`) | `Maui.Controls` `$(MauiVersion)`, `Logging.Debug` 10.0.3, `MotorDsl.Core/Parser/Rendering/Extensions/Printing.Abstractions/Bluetooth/Maui` **1.0.12** (`…/Ejemplo_MotorDSL.csproj:79-91`) | `Maui.Controls` **10.0.60** (fijo), `Logging.Debug` 10.0.3, `CommunityToolkit.Maui(.Core)` 14.2.0, `CommunityToolkit.Mvvm` 8.4.2, `MotorDsl.*` **1.0.13** (`…/Ejemplo_MotorDSL_Dialog.csproj:79-96`) |
| En `Ejemplos_Devices.slnx` | sí — `Ejemplos_Devices/Ejemplos_Devices.slnx:96` | sí — `Ejemplos_Devices/Ejemplos_Devices.slnx:90` | sí — `Ejemplos_Devices/Ejemplos_Devices.slnx:93` |
| UI | Página con botones + `DisplayAlertAsync` | Controles de la librería (`PrinterStatusBadge`, `PrinterPickerView`) + `Label` | Botón + `StatusOverlayView` (overlay de 3 estados) |
| iOS | Servicio devuelve vacío/`false` fuera de Android | Controles y botón ocultos en iOS | `PrinterService.IsSupported == false` → overlay «Impresión no disponible» |

Rutas abreviadas `…/X.csproj` = `Ejemplos_Devices/Printer/<proyecto>/X.csproj`.

⚠️ **Versiones divergentes de `MotorDsl.*`**: 1.0.12 en `Ejemplo_MotorDSL` vs 1.0.13 en `Ejemplo_MotorDSL_Dialog` (citas arriba). La app híbrida también referencia `MotorDsl` (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Ejemplo_Maui_Hibrida.csproj`, ver [08](08_App-Hibrida-Integrada.md)).
⚠️ `BarcodeScanner.Mobile.Maui` está referenciado en `Ejemplo_ThermalPrinter` (`Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Ejemplo_ThermalPrinter.csproj:102`) pero ningún `.cs` del proyecto lo usa (`grep -i barcodescanner` sin resultados en `.cs`).

```
ThermalPrinter            MotorDSL                         MotorDSL_Dialog
──────────────            ────────                         ───────────────
MainPage (code-behind)    MainPage (code-behind)           MainPage ─BindingContext→ MainViewModel
   │                         │  Render(json, profile)          │  Render(json, profile)
   ▼                         ▼                                 ▼
IThermalPrinterService    MotorDsl IDocumentEngine         PrinterOverlayViewModel (: StatusOverlayViewModel)
(propio, ESCPOS_NET)         │ byte[]                          │ permisos / discover / connect / send
   │ RFCOMM SPP              ▼                                 ▼
   ▼                      MotorDsl IThermalPrinterService  PrinterService → MotorDsl IThermalPrinterService
BluetoothSocket           (transport BT de la librería)      → resultados tipados + PrinterErrorCatalog
```

⚠️ **Dos `IThermalPrinterService` distintos**: el propio de `Ejemplo_ThermalPrinter` (`Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/IThermalPrinterService.cs:14`) y el de la librería (`using MotorDsl.Printing;` en `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:4` y `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:4`). Mismo nombre, contratos distintos.

## 2. Conexión Bluetooth

### 2.1 Directa (`Ejemplo_ThermalPrinter`)

| Paso | Qué hace | Cita |
|---|---|---|
| Adaptador | `BluetoothAdapter.DefaultAdapter` en el constructor | `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/ThermalPrinterService.cs:34` |
| «Escaneo» | No descubre: lista `BondedDevices` (sólo emparejados); lanza si no hay adaptador o está apagado | `…/Services/ThermalPrinterService.cs:44-68` |
| Conexión | `GetRemoteDevice(MAC)` → `CreateRfcommSocketToServiceRecord(UUID SPP 00001101-0000-1000-8000-00805F9B34FB)` → `Connect()` en `Task.Run` | `…/Services/ThermalPrinterService.cs:85-97` |
| Inicialización | Envía `EPSON.Initialize()` al conectar | `…/Services/ThermalPrinterService.cs:102` |
| Escritura | `WriteAsync` + `FlushAsync` sobre `socket.OutputStream`; envuelve errores en `Exception` genérica | `…/Services/ThermalPrinterService.cs:379-395` |
| Guarda | `EnsureConnected()` lanza si `IsConnected == false` | `…/Services/ThermalPrinterService.cs:370-376` |
| No-Android | Todo bajo `#if ANDROID`; en otras plataformas retorna lista vacía / `false` | `…/Services/ThermalPrinterService.cs:69-71`, `:112-115` |
| DI | `AddSingleton<IThermalPrinterService, ThermalPrinterService>` | `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/MauiProgram.cs:33` |

(`…/Services/…` = `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/…`.)

### 2.2 Vía librería (`MotorDSL` y `MotorDSL_Dialog`)

- Registro idéntico en ambos: `AddMotorDslEngine().AddProfiles(thermal_58mm).AddMotorDslMaui()` + `AddBluetoothPrinterTransport()` («Android Classic SPP» según el comentario) — `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/MauiProgram.cs:29-41`, `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/MauiProgram.cs:41-49`.
- MotorDSL: el `PrinterPickerView` de la librería escanea y conecta; la página sólo engancha `DeviceSelected`/`ScanError` y llama `ScanAsync()` — `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:36-51`, controles en `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml:11-13`.
- Dialog: `PrinterService` compone sobre el `IThermalPrinterService` de la librería — `DiscoverDevicesAsync(kind: "bluetooth")`, `ConnectAsync(PrinterDevice)`, `SendBytesAsync(bytes)` (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:98`, `:131`, `:141`).
- Dialog chequea el adaptador vía `BluetoothManager` (no `DefaultAdapter`, deprecado en API 31) **antes** de descubrir, porque la librería se traga las excepciones del transport y devolvería lista vacía — `…/Services/PrinterService.cs:57-94`.
- **Impresora predeterminada** (Dialog): se guarda en `Preferences` al conectar con éxito (`default_printer_id`/`_name`), se reusa sólo si sigue en `BondedDevices`, se olvida con `ClearDefault()`; alias por dispositivo `printer_alias_{id}` — `…/Services/PrinterService.cs:15-16`, `:121-134`, `:155-177`.
- «Activar Bluetooth» abre `Settings.ActionBluetoothSettings`; no enciende el adaptador (`Enable()` deprecado desde API 33) — `…/Services/PrinterService.cs:181-198`.

(`…/Services/PrinterService.cs` = `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs`.)

## 3. Comandos ESC/POS (`Ejemplo_ThermalPrinter`)

Se arman con el emisor `ESCPOS_NET.Emitters.EPSON` y se concatenan con `ByteSplicer.Combine` (`Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/ThermalPrinterService.cs:1-2`, `:189`). Los bytes de cada comando los genera la librería; la columna «ESC/POS» remite a la tabla del manual generado (no verificada contra el código de `ESCPOS_NET`).

| Método del servicio | Llamadas `EPSON` | ESC/POS (según manual) | Cita código |
|---|---|---|---|
| `ConnectAsync` | `Initialize()` | `ESC @` (`1B 40`) | `…/ThermalPrinterService.cs:102` |
| `PrintTextAsync` | `CodePage(PC858_EURO)`, `Center/LeftAlign`, `SetStyles(FontB/Bold/None)`, `Print`, `PrintLine("")`, `ResetLineSpacing` | `ESC a n`, `ESC M n`, `ESC E n` | `…/ThermalPrinterService.cs:148-193` |
| `PrintReceiptAsync` | encabezado centrado en negrita, ítems a **32 columnas** (`TruncateString(…,32)`, relleno de espacios), totales a la derecha, `FeedLines(3)`, `FullCutAfterFeed(5)` | `ESC d n`, `GS V` | `…/ThermalPrinterService.cs:195-279` |
| `PrintBarcodeAsync` | **imprime el dato como texto** (no emite `GS k`) | — | `…/ThermalPrinterService.cs:281-299` |
| `PrintQRCodeAsync` | **imprime el dato como texto** (no emite `GS ( k`); el `size` se acota a 1–16 y no se usa | — | `…/ThermalPrinterService.cs:301-323` |
| `PrintImageAsync` | lanza `NotImplementedException` (TODO raster) | — | `…/ThermalPrinterService.cs:325-344` |
| `FeedLinesAsync` / `CutPaperAsync` | `FeedLines(n)` / `FullCutAfterFeed(5)` | `ESC d n` / `GS V` | `…/ThermalPrinterService.cs:351`, `:362` |

(`…/ThermalPrinterService.cs` = `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/ThermalPrinterService.cs`. Tabla de comandos del manual: `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/Manuales/02_MANUAL_TECNICO_ESC_POS.md:150-275`.)

⚠️ `fontSize > 1` selecciona `PrintStyle.FontB` (`…/ThermalPrinterService.cs:169-172`), pero según el propio manual la Fuente B es **más chica** (42 caracteres/línea contra 32 de la A: `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/Manuales/02_MANUAL_TECNICO_ESC_POS.md:289`, `:300`). El «tamaño 2» del botón de prueba (`Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Pages/MainPage.xaml.cs:182`) no agranda la letra.
⚠️ La interfaz promete barcode y QR con `BarcodeType` (`Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/IThermalPrinterService.cs:64-71`, `:106-117`), pero la implementación los degrada a texto (comentarios en `…/ThermalPrinterService.cs:289-290`, `:313-314`). `PAPER_WIDTH_58MM = 384` está declarado y no se usa (`…/ThermalPrinterService.cs:20`).

## 4. Motor DSL (`Ejemplo_MotorDSL` y `Ejemplo_MotorDSL_Dialog`)

### 4.1 Pipeline

```
MultaIntegratedDsl.Document (JSON "integrated")
        │  IDocumentEngine.Render(json, DeviceProfile)
        ▼
RenderResult { IsSuccessful, Output: byte[] (ESC/POS bitmap), Errors, Warnings }
        │  si falla → se informa y NO se toca la impresora
        ▼
IThermalPrinterService.SendBytesAsync(bytes)   ← sólo si IsConnected / tras conectar
```

- «Render SIEMPRE primero, hardware después»: `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:127-157`; `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/MainViewModel.cs:32-41`.
- Perfil usado en el render (en línea): `new DeviceProfile("58HB6", 32, "escpos-bitmap")` + capacidades `supports_bitmap=true`, `bitmap_max_width_px=320`, `bitmap_binarization_threshold=128` — `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:129-132`, `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/MainViewModel.cs:33-36`.
- ⚠️ El perfil **registrado** en DI es otro (`thermal_58mm`, `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/MauiProgram.cs:36`, `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/MauiProgram.cs:44`) y el `Render` no lo usa.
- El registro de templates por nombre está comentado: `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/MauiProgram.cs:30-33`; se pasa el JSON directo.

### 4.2 Gramática observada del documento

Sólo se lista lo que aparece en el sample (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Samples/MultaIntegratedDsl.cs:25-130`); la gramática completa de la librería no está en este repo (no verificado).

| Elemento | Propiedades usadas | Valores en el sample | Cita (`MultaIntegratedDsl.cs`) |
|---|---|---|---|
| Documento | `id`, `version`, `format`, `root` | `"format": "integrated"` | `:27-30` |
| `container` | `layout`, `children` | `"vertical"` | `:31-33`, `:82-89` |
| `text` | `value`, `style.align`, `style.bold` | `align: "center"`, `bold: true`; separadores `"===="` de 32 chars | `:42-45`, `:51` |
| `image` (bitmap) | `source`, `imageType`, `width`, `style.align` | `data:image/bmp;base64,…`, `"bitmap"`, 200 / 150 | `:34-40`, `:119-125` |
| `image` (QR) | `source`, `imageType` | URL, `"qrcode"` | `:112-116` |

- Formato «integrated» = valores ya resueltos: sin `{{placeholders}}`, `loop` ni `conditional`, sin etapa *Evaluate* (comentarios `:4-9`, `:20-24`). Esas construcciones se mencionan pero **no se ejercitan** en el repo.
- El JSON es un raw string interpolado `$$"""…"""` que inyecta `LogoBase64` y `FirmaBase64` (`:13-18`, `:25`, `:36`, `:121`).
- ⚠️ Los comentarios dicen «PNG 1x1 blanco» / «PNG pequeño» (`:12`, `:16`) pero los datos son BMP (base64 empieza con `Qk0` = «BM», y el `source` declara `image/bmp`).
- ⚠️ Mismo archivo en las dos apps con distinto namespace: `Ejemplo_MotorDSL.Samples` (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Samples/MultaIntegratedDsl.cs:2`) vs `Ejemplo_MotorDSL.Templates` en Dialog (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Samples/MultaIntegratedDsl.cs:2`, importado en `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/MainViewModel.cs:7`); la carpeta es `Samples/` en ambas.

### 4.3 API de la librería que se toca

| Tipo / miembro | Dónde se usa |
|---|---|
| `IDocumentEngine.Render(string, DeviceProfile)` → `RenderResult` | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:133` |
| `RenderResult.IsSuccessful / Output / Errors / Warnings` | `…/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:135-142` |
| `IThermalPrinterService.IsConnected / SendBytesAsync / DiscoverDevicesAsync / ConnectAsync` | `…/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:150-157`; `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:98`, `:131`, `:141` |
| `PrinterDevice { Id, Name }` | `…/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:125`, `:157-158` |
| `PrintError.FromException` / `PrintErrorType` | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/PrinterErrorCatalog.cs:109-127` |
| Controles `PrinterStatusBadge`, `PrinterPickerView` (`FilterKind="bluetooth"`) | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml:4`, `:11-13` |

## 5. Permisos Bluetooth por plataforma

### 5.1 Android — manifiestos

| Permiso | ThermalPrinter | MotorDSL / Dialog |
|---|---|---|
| `BLUETOOTH`, `BLUETOOTH_ADMIN` (legacy) | `:14-15` | `:14-15` |
| `BLUETOOTH_SCAN` | `:16` **sin** `neverForLocation` | `:17-18` con `usesPermissionFlags="neverForLocation"` |
| `BLUETOOTH_CONNECT` | `:17` | `:19` |
| `ACCESS_FINE/COARSE_LOCATION` | `:20-21` | `:22-23` |
| `INTERNET`, `ACCESS_NETWORK_STATE` | `:10-11` | `:10-11` |
| `uses-sdk` en el manifest | `minSdk 25 / targetSdk 36` (`:30`) | no |

Archivos: `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Platforms/Android/AndroidManifest.xml`, `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Platforms/Android/AndroidManifest.xml`, `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Platforms/Android/AndroidManifest.xml`.

### 5.2 Android — pedido en runtime (tres generaciones)

| Proyecto | Mecánica | ¿Espera el resultado? | Cita |
|---|---|---|---|
| ThermalPrinter | Pide `LocationWhenInUse` en **todas** las versiones; en API ≥ 31 dispara `ActivityCompat.RequestPermissions(SCAN, CONNECT)` y retorna `true` | ⚠️ no (carrera) | `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Pages/MainPage.xaml.cs:36-76` |
| MotorDSL | API ≥ 31: `RequestPermissions` + `Task.Delay(3000)` fijo y re-chequeo; API < 31: `LocationWhenInUse` | ⚠️ por espera fija | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:57-107` (delay en `:75`) |
| Dialog | Permiso MAUI propio `BluetoothPermissions : BasePlatformPermission` (API ≥ 31 → SCAN+CONNECT; si no → FINE_LOCATION) + `CheckStatusAsync/RequestAsync/ShouldShowRationale` → `BluetoothPermissionResult` | sí (awaitable) | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/BluetoothPermissions.cs:9-21`; `…/Services/PrinterService.cs:37-54`; enum `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/BluetoothPermissionResult.cs:5-11` |

Dialog además re-chequea `BLUETOOTH_CONNECT` en cada descubrimiento (revocación con la app viva) → `DiscoverResult.PermissionRevoked` (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:87-93`).

### 5.3 iOS

- Ningún `Info.plist` de Printer declara `NSBluetoothAlwaysUsageDescription` (grep `bluetooth` sin resultados en los tres `Platforms/iOS/Info.plist`). Coherente con que la impresión SPP sea sólo Android: Dialog `IsSupported => false` fuera de Android (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:26-31`); MotorDSL oculta controles con `OnPlatform iOS=False` (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml:11`, `:13`, `:18`).
- ⚠️ El `Info.plist` de Dialog trae claves de cámara/fotos copiadas de otro dominio (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Platforms/iOS/Info.plist:58-64`).

## 6. Catálogo de errores y decisión del mensaje (Dialog)

### 6.1 Tipos

| Tipo | Variantes | Cita |
|---|---|---|
| `PrintFailure(Code, Title, UserMessage, TechnicalMessage, Exception?)` | `DisplayMessage = UserMessage + "\n\n" + Code` (lo único visible) | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/PrintFailure.cs:13-25` |
| `DiscoverResult` | `Found(devices)`, `Empty`, `BluetoothOff(err)`, `PermissionRevoked(err)`, `NotSupported` | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/DiscoverResult.cs:13-20` |
| `PrintResult` | `Success`, `Failure(PrintFailure)` | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/PrintResult.cs:6-10` |

### 6.2 Códigos (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/PrinterErrorCatalog.cs`)

Regla de diseño: **un código por acción distinta del usuario**, no por causa técnica; los códigos no se renombran (`:9-15`).

| Código | Fábrica / origen | Título | Botón primario en el overlay | Cita catálogo |
|---|---|---|---|---|
| `PRN-DOC-NET` | `DocumentoSinConexion` | Sin conexión | — (sin uso en este PoC) | `:20`, `:41-45` |
| `PRN-DOC-CONTRACT` | `DocumentoInvalido` | Comprobante inválido | — (sin uso en este PoC) | `:21`, `:47-51` |
| `PRN-DOC-RENDER` | `RenderFallido` (render no exitoso) | No se pudo generar el documento | Cerrar | `:22`, `:53-57` |
| `PRN-BT-OFF` | `BluetoothApagado` | Bluetooth desactivado | Activar Bluetooth | `:25`, `:59-63` |
| `PRN-BT-PERM` | `PermisoRevocado` | Permiso Bluetooth revocado | Abrir configuración | `:26`, `:65-69` |
| `PRN-DEV-NONE` | `SinImpresoras` (`Empty`) | No se encontraron impresoras | Reintentar (+ Emparejar impresora) | `:29`, `:71-75` |
| `PRN-DEV-ABSENT` | `PredeterminadaAusente` (falla conectar a la predeterminada) | La impresora no responde | Reintentar (+ Elegir otra si hay >1, Olvidar y emparejar otra) | `:30`, `:83-88` |
| `PRN-CONN-FAIL` | `NoSePudoConectar` (falla con una elegida) | No se pudo conectar | Reintentar (+ Elegir otra) | `:31`, `:91-95` |
| `PRN-HW-PAPER` | `Describe` → `Hardware`, mensaje contiene «paper» | Sin papel | «Ya cargué papel — Reintentar» | `:34`, `:139-142` |
| `PRN-HW-COVER` | ídem, contiene «cover» | Tapa abierta | «Ya la cerré — Reintentar» | `:35`, `:144-147` |
| `PRN-HW-OTHER` | ídem, resto | Problema en la impresora | Reintentar | `:36`, `:149-151` |
| `PRN-LINK-LOST` | `PrintErrorType.Connection` | Se perdió la conexión | Reintentar (+ Elegir otra si hay >1) | `:37`, `:115-117` |
| `PRN-TIMEOUT` | `PrintErrorType.Timeout` | La impresora no responde | ídem | `:38`, `:119-121` |
| `PRN-UNKNOWN` | `Protocol` y `Unknown` comparten | Falló la impresión | Reintentar | `:39`, `:123-126` |

### 6.3 Cómo se decide el mensaje

```
excepción de SendBytesAsync ──► PrinterService.SendAsync toma ex.InnerException ?? ex
                                 (la librería envuelve: "Print failed after N attempt(s)")
        ──► PrinterErrorCatalog.Describe(original, ex.Message)
              └─ PrintError.FromException(original,1,1).Type  (clasificación de la librería)
                   Hardware → Contains("paper"/"cover") sobre el mensaje original
        ──► PrintResult.Failure(PrintFailure) ──► PrinterOverlayViewModel.MostrarFalloImpresion
              switch(fallo.Code) arma la botonera; ShowError(icono, Title, DisplayMessage, acciones)
```

Citas: `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:137-153`; `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/PrinterErrorCatalog.cs:107-152`; `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/PrinterOverlayViewModel.cs:201-247`.
⚠️ El único punto frágil está declarado en el código: depende de los literales «paper»/«cover» de la librería; si cambian, degrada a `PRN-HW-OTHER` (`…/Models/PrinterErrorCatalog.cs:135-138`).
El mensaje técnico nunca se muestra: queda en `TechnicalMessage`/`Exception` y en `Debug.WriteLine("[PRINT] …")` (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/PrinterService.cs:200-205`).

En los otros dos ejemplos no hay catálogo: ThermalPrinter muestra `ex.Message` en `DisplayAlertAsync` (p. ej. `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Pages/MainPage.xaml.cs:193-196`); MotorDSL vuelca tipo, mensaje, inner y stack al `Label` (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml.cs:160-170`).

## 7. Flujo del overlay de impresión (Dialog)

```
ImprimirAsync(render)
 ├─ render falló ───────────────────────────► Error PRN-DOC-RENDER [Cerrar]
 ├─ !IsSupported ───────────────────────────► Error «Impresión no disponible» [Cerrar]
 ├─ Busy «Preparando impresión» → EnsurePermissionsAsync
 │    ├─ DeniedCanRetry ► [Pedir permiso] · Denied ► [Abrir configuración] · Restricted ► [Cerrar]
 └─ BuscarYImprimirAsync(forzarSelector=false)
      Busy «Buscando impresoras» → DiscoverAsync
      ├─ Found: predeterminada presente ► conectar (esPredeterminada) · 1 sola ► conectar · varias ► selector
      ├─ Empty / BluetoothOff / PermissionRevoked / NotSupported ► Error con botonera propia
      └─ ConectarEImprimirAsync → Busy «Conectando» → falla ► DEV-ABSENT | CONN-FAIL
                                 → EnviarAsync → Busy «Imprimiendo» → Success ► Hide | Failure ► §6.3
```

Citas: `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/PrinterOverlayViewModel.cs:44-71` (entrada), `:79-125` (descubrimiento), `:127-145` (selector; sufijo de MAC sólo si hay nombres repetidos, `:132-138`, `:287-291`), `:153-197` (conexión), `:249-273` (permisos), `:303-336` (comandos `[RelayCommand]`; `ElegirOtra` fuerza el selector para romper el bucle de la predeterminada, `:314`).

Base reutilizable `StatusOverlayViewModel` (CommunityToolkit `ObservableObject`): `OverlayMode { None, Busy, Error }`, `OverlayAction(Text, Command, Style)`, `ShowBusy`, `ShowError(icon, title, message, params actions)`, `Hide` — `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/StatusOverlayViewModel.cs:11-77`. Vista `StatusOverlayView` (ícono como ligadura de `MaterialIconsOutlined`; `Secondary` por `DataTrigger`) — `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Controls/StatusOverlayView.xaml:34`, `:46-51`; montada sobre la página en `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Pages/MainPage.xaml:18`. Fundamento del patrón: ver [Prosa relacionada](#10-prosa-relacionada).

⚠️ `Pages/OverlayBlueToothThermalPrintPage` de Dialog es una página vacía sin referencias (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Pages/OverlayBlueToothThermalPrintPage.xaml:1-8`); su gemela vive en la híbrida (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/MotorDSL/Pages/OverlayBlueToothThermalPrintPage.xaml:4`).
⚠️ El `README.md` de Dialog sólo contiene un fragmento de `MauiProgram` (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/README.md:3-40`).

## 8. `Ejemplo_Docs_Printer`: qué es cada cosa

24 archivos: **prosa y material de origen**, no un proyecto compilable (no tiene `.csproj`). Sólo `Readme.md`, `Ejemplo_MotorDSL/flowchat.*`, `Manuales/`, `Prompts_*/` y `Pruebas_realizadas/` figuran como `<File>` en `Ejemplos_Devices/Ejemplos_Devices.slnx:100-147`.

| Ruta (bajo `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/`) | Contenido |
|---|---|
| `Readme.md` | Índice de la carpeta + 2 videos de referencia (`:5-20`) |
| `Manuales/01_MANUAL_USUARIO.md` | Manual de usuario de la 58HB6 (carga, emparejamiento, LED, problemas) |
| `Manuales/02_MANUAL_TECNICO_ESC_POS.md` | Tabla de comandos ESC/POS, fuentes, code pages, estados (`:150-275`) |
| `Manuales/03_GUIA_INTEGRACION_ESCPOS_NET.md` | Guía `ESCPOS_NET` 2.2.1 para MAUI (USB y Bluetooth) |
| `Manuales/04_Borrador_Manual.md`, `Manuales/Readme.md` | Borrador unificado y resumen de los tres manuales |
| `Prompts_Generacion_Apk_ejemplo/Readme.md` + `ficheros_claude.ia/*` (`.cs`, `.XAML`, `.csproj`, `.md`) | Prompt a claude.ai y **código de partida** que originó `Ejemplo_ThermalPrinter` (no compila en la solución; es archivo histórico). `Androidmanifest.XML` está **vacío** (0 bytes) |
| `Prompts_generacion_documentacion/*` | Prompts a ChatGPT/Copilot que generaron los manuales (`Readme.md:5-20`) |
| `Borradores/Readme.md` | Borrador de README del ejemplo directo |
| `Pruebas_realizadas/Readme.md` | Corrida de `Ejemplo_ThermalPrinter` en Android 13 + capturas (`:5-12`) |

⚠️ Los manuales fueron **generados por IA** (ver `Prompts_generacion_documentacion/Readme.md:5-20`) y no están contrastados con el hardware: p. ej. `02_MANUAL_TECNICO_ESC_POS.md:56-60` describe «Bluetooth 5.0 (BLE)» con «SPP emulado», mientras el código usa RFCOMM/SPP clásico (`Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/ThermalPrinterService.cs:93-95`).
⚠️ El `.slnx` referencia `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md` (`Ejemplos_Devices/Ejemplos_Devices.slnx:111`), ruta inexistente; el archivo real está en `Ejemplo_Docs_Printer/Borradores/`.

## 9. Diferencias clave entre los tres ejemplos

| Aspecto | ThermalPrinter | MotorDSL | MotorDSL_Dialog |
|---|---|---|---|
| Qué enseña | El protocolo: bytes ESC/POS a mano | Layout como datos (JSON) rasterizado | Orquestación robusta y UX de errores |
| Layout | Código C# a 32 columnas | JSON integrado | JSON integrado (mismo sample) |
| Imágenes / QR | No (texto) | Sí (bitmap BMP y `qrcode`) | Sí |
| Selección de impresora | `Picker` de emparejados + Conectar | `PrinterPickerView` de la librería | Predeterminada → única → selector, con alias y desambiguación por MAC |
| Permisos | Sin await (carrera) | `Task.Delay(3000)` | Awaitable, 4 resultados |
| Errores | `ex.Message` en alerta | Volcado técnico en `Label` | Catálogo `PRN-*` + botonera por causa |
| Patrón | Code-behind | Code-behind | MVVM (CommunityToolkit) + overlay |

Citas por fila en §§2–7. Decisión de arquitectura detrás de esta progresión: [ADR-0005](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0005-impresion-motor-dsl.md).
⚠️ ADR-0005 dice que el enfoque directo es «sin dependencia externa» y que las «librerías tipo ESCPOS_NET» no se adoptaron; el código de `Ejemplo_ThermalPrinter` **sí** depende de `ESCPOS_NET` 2.2.1 (`Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Ejemplo_ThermalPrinter.csproj:105`, `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/ThermalPrinterService.cs:1-2`).

## 10. Prosa relacionada

| Enlace | Qué aporta |
|---|---|
| [pieces/printer/README.md](../../Ejemplos_Maui_Devices-docs/docs/pieces/printer/README.md) | Ficha de la pieza: comparativa ESC/POS vs DSL, snippets con procedencia, preguntas guía |
| [ADR-0005](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0005-impresion-motor-dsl.md) | Por qué el motor DSL sobre ESC/POS directo (decisión reconstruida) |
| [ADR-0002](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0002-servicio-tipado-overlay-mvvm.md) | Patrón servicio tipado → overlay MVVM que sigue Dialog |
| [01-architecture/07-overlays-dispositivos.md](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/07-overlays-dispositivos.md) | Fundamento de la máquina de tres estados y la botonera dinámica |
| [Analisis/Plan-Armonizacion-Overlays.md](../../Analisis/Plan-Armonizacion-Overlays.md) | Armonización de los overlays (incluida Impresión) en la híbrida |
| [GAP-REPORT.md](../../Ejemplos_Maui_Devices-docs/GAP-REPORT.md) | Deriva de versiones `MotorDsl.*` y bug de permisos con carrera (B-04) |

## 11. Fuentes

| Ruta | Contenido |
|---|---|
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/ThermalPrinterService.cs` | Conexión SPP y emisión ESC/POS con `ESCPOS_NET` |
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Services/IThermalPrinterService.cs` | Contrato propio, `BluetoothDevice`, `BarcodeType` |
| `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/Pages/MainPage.xaml(.cs)` | UI y permisos del ejemplo directo |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/MauiProgram.cs` | Cableado DI del motor y transport BT |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Pages/MainPage.xaml(.cs)` | Render-primero + controles de la librería |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/Samples/MultaIntegratedDsl.cs` | Documento DSL de ejemplo (acta de infracción) |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Services/{PrinterService,BluetoothPermissions}.cs` | Servicio tipado y permiso awaitable |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/*.cs` | `PrintFailure`, `PrinterErrorCatalog`, `DiscoverResult`, `PrintResult`, `BluetoothPermissionResult` |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/*.cs` | `MainViewModel`, `PrinterOverlayViewModel`, `StatusOverlayViewModel` |
| `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Controls/StatusOverlayView.xaml` | Vista genérica del overlay |
| `Ejemplos_Devices/Printer/*/Platforms/Android/AndroidManifest.xml`, `*/Platforms/iOS/Info.plist` | Permisos por plataforma |
| `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/` | Manuales, prompts y pruebas (§8) |
| `Ejemplos_Devices/Ejemplos_Devices.slnx:89-147` | Pertenencia a la solución y archivos de docs |
