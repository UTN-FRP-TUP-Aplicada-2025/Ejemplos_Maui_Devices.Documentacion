# 03 — Impresión térmica Bluetooth

> **Propósito**: entender los tres ejemplos de impresión en térmica de 58 mm por Bluetooth Classic (SPP), cómo pasan de ESC/POS armado a mano a un motor de documentos (MotorDsl), cómo piden los permisos Bluetooth en cada plataforma y cómo un fallo de impresión termina en el mensaje que lee el usuario.
> **Fuente primaria**: `Ejemplos_Devices/Printer/**` (repo de código `Ejemplos_Maui_Devices`, commit 285f6fb).
> **Índices relacionados**: [00_MASTER-INDEX](00_MASTER-INDEX.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (copia del flujo dentro de la app híbrida, `PrintCommandHandler`, tests) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) (workflow `cd-ios-printer.Ejemplo_MotorDSL.yml`, `.slnx`) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md) (README raíz, CHANGELOG, documentos externos) · [06_Telefonia](06_Telefonia.md) y [04_GPS](04_GPS.md) (los «espejos» `CallResult`/`GpsResult` que citan los comentarios).

---

## 1. Mapa del dominio

Abreviaturas de ruta que usa este índice (todas relativas a la raíz del repo de código):

| Abrev. | Ruta |
|---|---|
| `TP/` | `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter/` |
| `MD/` | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL/` |
| `DLG/` | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/` |
| `DOCS/` | `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/` |

```
Ejemplos_Devices/Printer/
├── Ejemplo_ThermalPrinter/     ← v1: ESC/POS "a mano" con ESCPOS_NET + socket RFCOMM propio (code-behind)
├── Ejemplo_MotorDSL/           ← v2: JSON del documento → MotorDsl.* (NuGet 1.0.12) → bytes ESC/POS bitmap
├── Ejemplo_MotorDSL_Dialog/    ← v3: igual que v2 (NuGet 1.0.13) + MVVM + overlay + resultados tipados + catálogo de errores
└── Ejemplo_Docs_Printer/       ← prosa: prompts de generación, manuales IA del hardware 58HB6, pruebas, flowchart
```

Evolución didáctica (cada paso conserva el anterior; la duplicación es deliberada):

```
TP  : Page ──try/catch──► IThermalPrinterService (propio) ──ESCPOS_NET.EPSON──► BluetoothSocket ─► impresora
MD  : Page ──► IDocumentEngine.Render(json, DeviceProfile) ─byte[]─► MotorDsl IThermalPrinterService.SendBytesAsync
DLG : MainViewModel ─Render─► PrinterOverlayViewModel ─► PrinterService ─► (DiscoverResult / PrintResult / PrintFailure)
                                       │                                         └─► MotorDsl IThermalPrinterService
                                       └─► StatusOverlayView (Busy / Error + botonera)
```

El flujo v3 se portó tal cual a la app híbrida (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/MotorDSL/`): ver §9 y [08](08_App-Hibrida-Integrada.md).

## 2. Ficha de los tres proyectos

| Aspecto | `Ejemplo_ThermalPrinter` | `Ejemplo_MotorDSL` | `Ejemplo_MotorDSL_Dialog` |
|---|---|---|---|
| TFM | `net10.0-android` (+`net10.0-ios` sólo en macOS) `TP/Ejemplo_ThermalPrinter.csproj:4-5` | ídem `MD/Ejemplo_MotorDSL.csproj:4-5` | ídem `DLG/Ejemplo_MotorDSL_Dialog.csproj:4-5` |
| `ApplicationId` | `com.ejemplos.devices.ThermalPrinter` (`:17`) | `com.ejemplos.devices.MotorDSL` (`:17`) | `com.ejemplos.devices.MotorDSL.Dialog` (`:17`) |
| Min OS | Android 25 / iOS 15 (`:24-25`) | ídem | ídem |
| Paquetes de impresión | `ESCPOS_NET` 2.2.1 (`:105`) | 7 × `MotorDsl.*` **1.0.12** (`:85-91`) | 7 × `MotorDsl.*` **1.0.13** (`:90-96`) |
| Otros paquetes | `BarcodeScanner.Mobile.Maui` 9.0.1 (`:102`, sin uso en el código) | — | `CommunityToolkit.Maui(.Core)` 14.2.0, `CommunityToolkit.Mvvm` 8.4.2 (`:84-86`); `Microsoft.Maui.Controls` fijo en 10.0.60 (`:79`) |
| Patrón UI | code-behind + `DisplayAlertAsync` | code-behind + controles de `MotorDsl.Maui` | MVVM + overlay de estado |
| Imprime en iOS | no (stubs) | no (UI oculta en iOS) | no (`IsSupported == false`) |
| CD iOS | — | `.github/workflows/cd-ios-printer.Ejemplo_MotorDSL.yml` → [09](09_CI-CD-y-Build.md) | — |

Los siete paquetes MotorDsl: `Core`, `Parser`, `Rendering`, `Extensions`, `Printing.Abstractions`, `Bluetooth`, `Maui`. El comentario del csproj aclara que se consumen como NuGet «en lugar de ProjectReference» para validar lo publicado (`MD/Ejemplo_MotorDSL.csproj:83`).

⚠️ `TP/Ejemplo_ThermalPrinter.csproj:65-80` repite dentro del bloque `<!--resources-->` los mismos `PropertyGroup` de Android (`:30-33`) e iOS Release (`:51-61`); es redundante, no cambia el resultado.

## 3. `Ejemplo_ThermalPrinter` — ESC/POS a mano

### 3.1 Piezas

| Archivo | Rol |
|---|---|
| `TP/MauiProgram.cs:30-39` | `AddsServices`: `IThermalPrinterService` → `ThermalPrinterService` (singleton), `MainPage` (transient) |
| `TP/Services/IThermalPrinterService.cs:14-89` | contrato: `IsConnected`, `ScanDevicesAsync`, `ConnectAsync(mac)`, `DisconnectAsync`, `PrintTextAsync`, `PrintReceiptAsync`, `PrintBarcodeAsync`, `PrintQRCodeAsync`, `PrintImageAsync`, `FeedLinesAsync`, `CutPaperAsync` |
| `TP/Services/IThermalPrinterService.cs:94-117` | DTO `BluetoothDevice` (Name, Address, IsPaired) y enum `BarcodeType` (9 simbologías) |
| `TP/Services/ThermalPrinterService.cs` | implementación; todo el cuerpo real va bajo `#if ANDROID` |
| `TP/Pages/MainPage.xaml(.cs)` | 4 secciones: Buscar → Conectar → Pruebas → Texto personalizado + `MessageLabel` de estado |

### 3.2 Transporte (Android)

| Paso | Código | Detalle |
|---|---|---|
| Adaptador | `ThermalPrinterService.cs:34` | `BluetoothAdapter.DefaultAdapter` (API deprecada en Android 12+; DLG usa `BluetoothManager`, §5.3) |
| «Escanear» | `:39-73` | **no descubre**: devuelve sólo `BondedDevices` (`:55`); lanza si no hay adaptador (`:44-47`) o está apagado (`:49-52`) |
| Conectar | `:75-116` | `GetRemoteDevice(mac)` → `CreateRfcommSocketToServiceRecord` con UUID SPP `00001101-0000-1000-8000-00805F9B34FB` (`:93-95`) → `Connect()` en `Task.Run` (`:97`) → envía `Initialize()` (`:102`) |
| Escribir | `:379-395` | `WriteAsync` + `FlushAsync` sobre `_socket.OutputStream` |
| Guardia | `:370-376` | `EnsureConnected()` lanza `"No hay conexión con la impresora. Conecta primero."` |
| No-Android | `:69-71`, `:112-115` | `ScanDevicesAsync` devuelve lista vacía; `ConnectAsync` devuelve `false` |

### 3.3 Comandos ESC/POS que emite

Se arman con el emisor `ESCPOS_NET.Emitters.EPSON`; en `PrintTextAsync`, `PrintReceiptAsync` y los de QR/barras se concatenan con `ByteSplicer.Combine` antes de un único `WriteBytes` (`:189`, `:275`), mientras `Initialize` al conectar, `FeedLinesAsync` y `CutPaperAsync` escriben su comando suelto. La columna «Bytes» viene del manual IA `DOCS/Manuales/02_MANUAL_TECNICO_ESC_POS.md` (línea indicada); qué bytes exactos produce cada método de ESCPOS_NET **no verificado** contra el fuente de la librería.

| Método `EPSON` | Dónde | Comando según el manual |
|---|---|---|
| `Initialize()` | `:102`, `:209` | `ESC @` = `1B 40` (02:156) |
| `CodePage(CodePage.PC858_EURO)` | `:156` | `ESC t` = `1B 74 0C` (02:450) |
| `LeftAlign/CenterAlign/RightAlign()` | `:161-165`, `:248` | `ESC a n` = `1B 61 0/1/2` (02:165) |
| `SetStyles(PrintStyle.Bold/FontB/None)` | `:171`, `:177`, `:186` | negrita `1B 45`, fuente `1B 4D` (02:178, 02:181) |
| `Print(text)` + `PrintLine("")` | passim | texto + `LF 0A` (02:157) |
| `ResetLineSpacing()` | `:185` | no documentado en el manual |
| `FeedLines(n)` | `:272`, `:351` | `ESC d n` = `1B 64` (02:203) |
| `FullCutAfterFeed(5)` | `:273`, `:362` | `GS V` = `1D 56` (02:217-219) |

### 3.4 Qué hace realmente cada botón

| Botón (`MainPage.xaml`) | Handler | Qué imprime |
|---|---|---|
| Imprimir Texto de Prueba (`:42-50`) | `OnPrintTestClicked` `MainPage.xaml.cs:176-198` | 7 `PrintTextAsync` + `FeedLinesAsync(3)` |
| Imprimir Ticket de Ejemplo (`:52-60`) | `:200-233` | `PrintReceiptAsync`: encabezado en negrita, ítems a 32 columnas (`ThermalPrinterService.cs:236`), totales a la derecha, corte |
| Imprimir Código QR (`:62-70`) | `:235-253` | ⚠️ **el dato como texto plano**: `PrintQRCodeAsync` no emite QR (`ThermalPrinterService.cs:313-315`) |
| Imprimir Código de Barras (`:72-80`) | `:255-272` | ⚠️ ídem: `PrintBarcodeAsync` imprime el dato (`:289-291`); el `BarcodeType` se ignora |
| Imprimir Texto Personalizado (`:133-141`) | `:274-300` | `fontSize = SelectedIndex + 1` (`:286`), negrita y centrado según los checkbox |

`PrintImageAsync` lanza `NotImplementedException` envuelta (`ThermalPrinterService.cs:325-344`) y ningún botón la usa. `PAPER_WIDTH_58MM = 384` (`:20`) se declara pero no se usa.

### 3.5 Manejo de errores (patrón v1)

Cada capa envuelve el mensaje en otro `Exception` genérico (`"Error al conectar: …"` `:110`, `"Error al escribir en la impresora: …"` `:393`), y cada handler de la página hace `try/catch` → `ShowMessage(...)` + `DisplayAlertAsync("Error", $"…: {ex.Message}")`. El usuario ve el texto técnico encadenado (p. ej. `Error al imprimir: Error al escribir en la impresora: …`). Es el punto de partida que §7 reemplaza.

## 4. `Ejemplo_MotorDSL` — documento JSON → motor → bytes

### 4.1 Registro (`MD/MauiProgram.cs`)

| Línea | Registro |
|---|---|
| `:29` | `AddMotorDslEngine()` — pipeline (el comentario `:27` cita renderers PDF, ESC/POS bitmap y SkiaSharp) |
| `:30-33` | `.AddTemplates(...)` **comentado**: el documento no se registra como template; se pasa el JSON directo |
| `:34-37` | `.AddProfiles`: `DeviceProfile("thermal_58mm", 32, "escpos-bitmap")` |
| `:38` | `.AddMotorDslMaui()` |
| `:41` | `AddBluetoothPrinterTransport()` — «Android Classic SPP» |

⚠️ El comentario `:28` dice «El template registrado es un JSON integrado», pero el `.AddTemplates` está comentado; en DLG el comentario se repite (`DLG/MauiProgram.cs:40`) y ni siquiera queda la llamada comentada.

### 4.2 Página (`MD/Pages/MainPage.xaml(.cs)`)

| Elemento | Dónde | Nota |
|---|---|---|
| `muic:PrinterStatusBadge` / `muic:PrinterPickerView FilterKind="bluetooth"` | `MainPage.xaml:11`, `:13` | controles de `MotorDsl.Maui`; `IsVisible` sólo en Android (`OnPlatform`) |
| Botón Imprimir | `MainPage.xaml:18` | también oculto en iOS |
| Inyección | `MainPage.xaml.cs:25-30` | `IDocumentEngine` + `MotorDsl.Printing.IThermalPrinterService` |
| Arranque | `:32-52` | conecta los controles al servicio; en Android pide permisos y luego `DevicePicker.ScanAsync()`; en iOS no hace nada |
| Imprimir | `:119-171` | ver flujo abajo |

```
OnImprimirClicked
 1. profile = DeviceProfile("58HB6", 32, "escpos-bitmap")             :129
      + supports_bitmap=true, bitmap_max_width_px=320, threshold=128   :130-132
 2. result = _engine.Render(MultaIntegratedDsl.Document, profile)      :133
      └─ falla → "RENDER FALLÓ:\n<primer error, 200 chars>" + Console   :135-144
 3. !_printer.IsConnected → "Render OK (N bytes) pero no hay impresora" :150-154
 4. Task.Delay(500); SendBytesAsync(bytes) → "Impreso OK — N bytes"     :156-158
 catch → TIPO / MSG / INNER / INNER2 / STACK(200) en MessageLabel      :160-170
```

«Render SIEMPRE primero» (`:127`): el diagnóstico del documento no depende de tener impresora. ⚠️ El perfil `thermal_58mm` registrado en DI no se usa en esta página: arma su propio `58HB6` (también en DLG).

El `catch` vuelca el detalle técnico completo en pantalla: es un PoC de diagnóstico, no un mensaje de usuario (el contraste con §7 es deliberado).

### 4.3 El documento de ejemplo (`MD/Samples/MultaIntegratedDsl.cs`)

Acta de infracción de tránsito en formato **integrado** (`"format": "integrated"`, `:29`): valores ya resueltos, sin `{{placeholders}}`, sin `loop` ni `conditional`, sin etapa de Evaluate (`:4-9`). Es un raw string `$$"""…"""` (`:25`), así que `{{LogoBase64}}` es interpolación de C#, no un placeholder del DSL.

| Nodo | Dónde | Detalle |
|---|---|---|
| `container` raíz, `layout: vertical` | `:30-33` | |
| `image` logo | `:34-40` | `data:image/bmp;base64,…`, `imageType: bitmap`, `width: 200`, centrado |
| `text` (algunos con `style`) | `:41-126` (passim) | `align`, `bold`; separadores de 32 `=` |
| 2 × `container` de infracción | `:81-98` | el `loop` ya expandido |
| `image` QR | `:112-116` | `imageType: qrcode`, `source` = URL de pago |
| `image` firma | `:119-125` | bitmap, `width: 150` |

⚠️ Los comentarios dicen «PNG 1x1 blanco» y «PNG pequeño» (`:12`, `:16`), pero los base64 son **BMP** (cabecera `BM`) de 225×90 y 225×38 px a 4 bpp (verificado decodificando `:14` y `:18`); la URI ya declara `image/bmp`.
⚠️ «Art. Art. 77» / «Art. Art. 48» (`:85`, `:94`): el prefijo duplicado está en el dato.

`DLG/Samples/MultaIntegratedDsl.cs` es idéntico salvo el namespace: `Ejemplo_MotorDSL.Templates` (línea 2), no `Ejemplo_MotorDSL_Dialog.*`; por eso `DLG/ViewModels/MainViewModel.cs:7` hace `using Ejemplo_MotorDSL.Templates;`.

## 5. `Ejemplo_MotorDSL_Dialog` — flujo con overlay y resultados tipados

### 5.1 Capas

| Capa | Archivo | Responsabilidad |
|---|---|---|
| DI | `DLG/MauiProgram.cs:41-49`, `:60-77` | motor + transporte como en MD; `PrinterService`, `PrinterOverlayViewModel`, `MainViewModel`, `MainPage` como **singleton** |
| Página | `DLG/Pages/MainPage.xaml:13`, `:18` | botón «Imprimir Prueba» → `ImprimirEjemploCommand`; `StatusOverlayView` enlazado a `PrinterOverlayViewModel` |
| VM principal | `DLG/ViewModels/MainViewModel.cs:29-42` | renderiza (mismo perfil `58HB6` que MD, `:33-36`) y delega en `PrinterOverlayViewModel.ImprimirAsync(render)` |
| VM overlay | `DLG/ViewModels/PrinterOverlayViewModel.cs` | máquina de flujo: render → permisos → descubrir → elegir → conectar → enviar; decide textos y botonera |
| Base overlay | `DLG/ViewModels/StatusOverlayViewModel.cs:28-78` | `OverlayMode {None, Busy, Error}` (`:11`), `OverlayAction(Text, Command, Style)` (`:21`), `ShowBusy` (`:54-61`), `ShowError` (`:64-74`), `Hide` (`:77`) |
| Vista overlay | `DLG/Controls/StatusOverlayView.xaml` | capa Busy (`:21-28`, imagen animada) y capa Error (`:31-58`, ícono + título + mensaje + botonera `BindableLayout`) |
| Servicio | `DLG/Services/PrinterService.cs` | permisos, descubrimiento, conexión, envío, preferencias; **no referencia ViewModels** (`:8-12`) |
| Modelos | `DLG/Models/*.cs` | `BluetoothPermissionResult`, `DiscoverResult`, `PrintResult`, `PrintFailure`, `PrinterErrorCatalog` |

`DLG/Pages/OverlayBlueToothThermalPrintPage.xaml(.cs)` es una página vacía, sin uso (marcador del proyecto). `DLG/README.md` sólo muestra el fragmento de registro de `MauiProgram`.

### 5.2 Tipos de resultado

| Tipo | Archivo | Casos |
|---|---|---|
| `BluetoothPermissionResult` (enum) | `Models/BluetoothPermissionResult.cs:5-11` | `Granted`, `DeniedCanRetry`, `Denied`, `Restricted` |
| `DiscoverResult` (record abstracto) | `Models/DiscoverResult.cs:13-20` | `Found(Devices)`, `Empty`, `BluetoothOff(PrintFailure)`, `PermissionRevoked(PrintFailure)`, `NotSupported` |
| `PrintResult` | `Models/PrintResult.cs:6-10` | `Success`, `Failure(PrintFailure)` |
| `PrintFailure` (record) | `Models/PrintFailure.cs:13-26` | `Code`, `Title`, `UserMessage`, `TechnicalMessage`, `Exception?`; `DisplayMessage = "{UserMessage}\n\n{Code}"` (`:25`) |

El comentario de `DiscoverResult.cs:8-12` deja la regla: «cada variante debe ser ALCANZABLE»; `BluetoothOff` existió sin que nada la construyera hasta que `DiscoverAsync` empezó a chequear el adaptador (§5.3).

### 5.3 `PrinterService` (DLG)

| Miembro | Líneas | Qué hace |
|---|---|---|
| `IsSupported` | `:26-31` | `true` sólo en Android (BT Classic SPP) |
| `EnsurePermissionsAsync` | `:37-54` | `CheckStatusAsync` → `RequestAsync` → `Restricted` / `ShouldShowRationale` ⇒ `DeniedCanRetry` o `Denied`; fuera de Android devuelve `Granted` |
| `DiscoverAsync` | `:67-110` | `NotSupported` si no hay soporte o adaptador; **antes de llamar a la librería** chequea `adapter.IsEnabled` (`:81-83`) y, en API 31+, `BLUETOOTH_CONNECT` (`:87-93`); luego `DiscoverDevicesAsync(kind: "bluetooth")` (`:98`) → `Empty` / `Found` |
| `GetDefaultIfPresent` | `:121-126` | la predeterminada (Preferences `default_printer_id`, `:15`) sólo si está en la lista; «presente» = **emparejada**, no encendida |
| `ConnectAsync` | `:129-134` | conecta y, si anda, guarda la predeterminada (`SaveDefault`, `:155-159`) |
| `SendAsync` | `:137-153` | `SendBytesAsync`; en error clasifica `ex.InnerException ?? ex` con el catálogo (`:149-151`) |
| `ClearDefault` / `GetAlias` / `SetAlias` | `:166-177` | olvidar predeterminada; alias por MAC (`printer_alias_{id}`) |
| `OpenAppSettings` / `OpenBluetoothSettings` | `:179`, `:191-198` | ajustes de la app vs. `Settings.ActionBluetoothSettings` del SO (no enciende el adaptador: `Enable()` deprecado desde API 33, `:184-189`) |
| `Log` | `:204-205` | sólo `Debug.WriteLine("[PRINT] …")`; punto único para un destino de log futuro |

⚠️ El `catch` de `DiscoverAsync` (`:103-109`) es una red de seguridad que el propio comentario declara **hoy inalcanzable**: el `ThermalPrinterService` de la librería se traga las excepciones de los transportes y devuelve lista vacía (`:58-65`). Sin los chequeos previos, «Bluetooth apagado» llegaría como «no se encontraron impresoras».

### 5.4 Flujo del overlay (`PrinterOverlayViewModel`)

```
ImprimirAsync(render)                                              :44-71
 ├─ render falló ──────────────► Error PRN-DOC-RENDER [Cerrar*]     :48-55
 ├─ !IsSupported ──────────────► Error "Impresión no disponible" [Cerrar*]   :58-64
 ├─ Busy "Preparando impresión" → EnsurePermissionsAsync            :66-68
 │    └─ ≠ Granted ────────────► MostrarPermiso                      :249-273
 └─ BuscarYImprimirAsync(forzarSelector=false)                      :79-125
      ├─ Found ── ¿predeterminada presente? → ConectarEImprimir(esPredeterminada:true)
      │        ── ¿una sola?               → ConectarEImprimir(esPredeterminada:false)
      │        └─ si no / forzarSelector   → MostrarSelector          :127-145
      ├─ Empty ───────────► PRN-DEV-NONE  [Reintentar][Emparejar impresora][Cerrar]
      ├─ BluetoothOff ────► PRN-BT-OFF    [Activar Bluetooth][Reintentar][Cerrar]
      ├─ PermissionRevoked► PRN-BT-PERM   [Abrir configuración][Reintentar][Cerrar]
      └─ NotSupported ────► "Impresión no disponible" [Cerrar*]
ConectarEImprimirAsync(device, esPredeterminada)                   :153-167
 ├─ Busy "Conectando a <etiqueta>…"   (nombra la impresora)
 ├─ falla y era predeterminada → PRN-DEV-ABSENT                     :174-188
 ├─ falla y la eligió el usuario → PRN-CONN-FAIL                    :190-197
 └─ OK → EnviarAsync → Success ⇒ Hide() | Failure ⇒ MostrarFalloImpresion   :201-247
(*) «Cerrar» único ⇒ estilo Primary (`Cerrar(unico: true)`, :299-301)
```

Estado guardado para reintentos (`:25-36`): `_bytes`, `_lastDevice`, `_lastRender`, `_lastEraPredeterminada`, `_lastDevices`. Comandos (`:303-336`): `CerrarOverlay`, `AbrirAjustes`, `AbrirAjustesBluetooth`, `PedirPermiso` (reusa el último render), `ReintentarBuscar`, `ElegirOtra` (`forzarSelector: true`), `ReintentarConectar` (conserva el rol de predeterminada), `ReintentarImprimir` (sólo reenvía bytes), `OlvidarYEmparejar` (`ClearDefault` + ajustes BT + pantalla «Emparejá la impresora»).

Selector de impresoras (`:127-145`, `:276-291`): etiqueta = alias del usuario si existe; si el nombre BT se repite (dos «58HB6»), agrega los últimos 2 octetos de la MAC.

Íconos: `IconGlyph` recibe nombres de ligadura de Material Icons (`"error"`, `"print_disabled"`, `"bluetooth_disabled"`, `"print"`, `"block"`) que la vista dibuja con `FontFamily="MaterialIconsOutlined"` (`StatusOverlayView.xaml:34`; fuente registrada en `DLG/MauiProgram.cs:34`). La capa Busy usa `timer.gif` (`DLG/Resources/Images/timer.gif`).

## 6. Permisos Bluetooth por plataforma

### 6.1 Manifiestos Android

| Permiso | TP `AndroidManifest.xml` | MD / DLG `AndroidManifest.xml` |
|---|---|---|
| `BLUETOOTH`, `BLUETOOTH_ADMIN` (legado < API 31) | `:14-15` | `:14-15` |
| `BLUETOOTH_SCAN` | `:16` sin flags | `:17-18` con `usesPermissionFlags="neverForLocation"` |
| `BLUETOOTH_CONNECT` | `:17` | `:19` |
| `ACCESS_FINE/COARSE_LOCATION` | `:20-21` | `:22-23` |
| `<queries>` `REQUEST_ENABLE` | `:23-27` | `:25-29` |
| `INTERNET`, `ACCESS_NETWORK_STATE` | `:10-11` | `:10-11` |
| `uses-sdk` | `min 25 / target 36` (`:30`) | no declara (lo pone el csproj) |

### 6.2 Pedido en tiempo de ejecución — tres generaciones

| Proyecto | Dónde | Estrategia | Trampa |
|---|---|---|---|
| TP | `TP/Pages/MainPage.xaml.cs:36-83` (desde `OnAppearing`) | siempre pide `LocationWhenInUse` (`:40-53`); en API 31+ además `ActivityCompat.RequestPermissions(SCAN, CONNECT)` (`:56-74`) | ⚠️ el pedido de API 31+ no se espera ni se verifica: devuelve `true` igual (`:76`); en Android 12+ pide ubicación aunque no haga falta |
| MD | `MD/Pages/MainPage.xaml.cs:57-107` | API 31+: `RequestPermissions` y **`Task.Delay(3000)`** antes de volver a chequear (`:72-78`); < 31: `LocationWhenInUse` (`:86-97`) | ⚠️ si el usuario tarda más de 3 s, se informa «Permisos BT denegados… presioná Reescanear» (`:82`) aunque después acepte |
| DLG | `DLG/Services/BluetoothPermissions.cs:9-21` + `PrinterService.EnsurePermissionsAsync` | permiso propio `Permissions.BasePlatformPermission` (MAUI no trae `Permissions.Bluetooth`): API 31+ `BLUETOOTH_SCAN`+`BLUETOOTH_CONNECT`, antes `ACCESS_FINE_LOCATION`; awaitable y con `ShouldShowRationale` | se re-chequea `BLUETOOTH_CONNECT` en cada descubrimiento por si se revocó desde Ajustes (§5.3) |

Respuesta en DLG a cada resultado (`PrinterOverlayViewModel.cs:249-273`):

| `BluetoothPermissionResult` | Ícono | Título | Botones |
|---|---|---|---|
| `DeniedCanRetry` | `bluetooth_disabled` | «Permiso Bluetooth necesario» | Pedir permiso · Cerrar |
| `Denied` | `bluetooth_disabled` | «Acceso Bluetooth denegado» | Abrir configuración · Cerrar |
| `Restricted` | `block` | «Acceso restringido» | Cerrar (primario) |

### 6.3 iOS

Ningún ejemplo imprime en iOS: TP compila stubs (§3.2), MD oculta badge/picker/botón con `OnPlatform` (`MD/Pages/MainPage.xaml:11-18`) y DLG corta en `IsSupported` (`PrinterService.cs:26-31`). Ningún `Info.plist` del dominio declara `NSBluetoothAlwaysUsageDescription`.
⚠️ `DLG/Platforms/iOS/Info.plist:58-64` declara permisos de **cámara y fotos** con el comentario «captura de medios»: arrastre de plantilla, no los usa la impresión. `MD`/`DLG` `Entitlements*.plist` piden `aps-environment=development` (push), tampoco relacionado.

## 7. Catálogo de errores → mensaje al usuario

### 7.1 Reglas del catálogo (`DLG/Models/PrinterErrorCatalog.cs:5-16`)

- Códigos **estables** de cara a soporte: se agregan, no se renombran.
- **Un código por acción distinta del usuario**, no por causa técnica: dos causas que se resuelven con el mismo gesto comparten código.
- La UI muestra sólo `DisplayMessage` (mensaje accionable + código al final para dictarlo); `TechnicalMessage` y `Exception` quedan para log y nunca se muestran crudos (`PrintFailure.cs:3-7`, `:20-25`).

### 7.2 Los 14 códigos

| Código | Constante | Fábrica / origen | Título | Mensaje al usuario (resumen) |
|---|---|---|---|---|
| `PRN-DOC-NET` | `DocNetwork` | `DocumentoSinConexion` `:41-45` | Sin conexión | revisá la conexión y reintentá |
| `PRN-DOC-CONTRACT` | `DocContract` | `DocumentoInvalido` `:47-51` | Comprobante inválido | avisá a soporte con el código |
| `PRN-DOC-RENDER` | `DocRender` | `RenderFallido` `:53-57` | No se pudo generar el documento | avisá a soporte con el código |
| `PRN-BT-OFF` | `BtOff` | `BluetoothApagado` `:59-63` | Bluetooth desactivado | activá el Bluetooth |
| `PRN-BT-PERM` | `BtPermission` | `PermisoRevocado` `:65-69` | Permiso Bluetooth revocado | habilitalo desde los ajustes de la app |
| `PRN-DEV-NONE` | `DeviceNone` | `SinImpresoras` `:71-75` | No se encontraron impresoras | encendida y emparejada |
| `PRN-DEV-ABSENT` | `DeviceAbsent` | `PredeterminadaAusente(etiqueta)` `:83-88` | La impresora no responde | apagada/fuera de alcance **o** no es la emparejada |
| `PRN-CONN-FAIL` | `ConnFail` | `NoSePudoConectar(etiqueta)` `:91-95` | No se pudo conectar | encendida y cerca |
| `PRN-HW-PAPER` | `HwPaper` | `Describe` → `Hardware` `:139-142` | Sin papel | cargá un rollo nuevo |
| `PRN-HW-COVER` | `HwCover` | `Hardware` `:144-147` | Tapa abierta | cerrala y reintentá |
| `PRN-HW-OTHER` | `HwOther` | `Hardware` `:149-151` | Problema en la impresora | revisala y reintentá |
| `PRN-LINK-LOST` | `LinkLost` | `Describe` `:115-117` | Se perdió la conexión | encendida y cerca |
| `PRN-TIMEOUT` | `Timeout` | `Describe` `:119-121` | La impresora no responde | reintentá |
| `PRN-UNKNOWN` | `Unknown` | `Describe` `:124-126` | Falló la impresión | reintentá |

`PRN-DOC-NET` y `PRN-DOC-CONTRACT` **no se usan en DLG**: el PoC renderiza una muestra local (`PrinterOverlayViewModel.cs:17-20`). Sólo los produce la app híbrida, que baja el comprobante por red ([08](08_App-Hibrida-Integrada.md), `PrintCommandHandler.cs:61`, `:68`).

### 7.3 Clasificación de un fallo de envío

```
SendBytesAsync lanza  Exception("Print failed after N attempt(s): …", inner)   ← librería
        │  PrinterService.SendAsync toma inner ?? ex                         PrinterService.cs:149
        ▼
PrinterErrorCatalog.Describe(original, technical)                            :107-128
        │  PrintError.FromException(original, 1, 1)  ← clasificación de la librería, no se reimplementa
        ├─ Hardware   → mira original.Message: "paper" → HW-PAPER · "cover" → HW-COVER · otro → HW-OTHER
        ├─ Connection → LINK-LOST
        ├─ Timeout    → TIMEOUT
        └─ Protocol / Unknown → UNKNOWN (no tienen acción propia del usuario)
```

Verificado en el fuente de la librería fuera de este repo (`/home/fernando/workspaces/workspace-dev/Libs/NugetOrg/PrintThermal_Motor_Maui`, HEAD `0f8413d`; **no** necesariamente igual a los paquetes 1.0.12/1.0.13): `PrintError.FromException` mapea `PrinterHardwareException`→Hardware, `PrintNotConfirmedException`→Protocol, `IOException`/`SocketException`→Connection, `TimeoutException`/`TaskCanceledException`→Timeout (`src/MotorDsl.Core/Models/PrintError.cs:40-55`); los literales `"paper out"`/`"cover open"` salen de sondear `DLE EOT` (`10 04 04` / `10 04 02`) en `src/MotorDsl.Bluetooth/BluetoothPrinterTransport.Detection.cs:128-140`; la envoltura «Print failed after…» está en `src/MotorDsl.Printing.Abstractions/Services/ThermalPrinterService.cs:227`.

⚠️ Única dependencia de texto: si la librería cambia esos literales, el fallo degrada a `PRN-HW-OTHER` (pierde precisión, no corrección) (`PrinterErrorCatalog.cs:135-138`).

### 7.4 Botonera según el código (`MostrarFalloImpresion`, `:219-247`)

| Código | Botón primario | Secundarios |
|---|---|---|
| `PRN-HW-PAPER` | «Ya cargué papel — Reintentar» | Cerrar |
| `PRN-HW-COVER` | «Ya la cerré — Reintentar» | Cerrar |
| `PRN-LINK-LOST`, `PRN-TIMEOUT` | Reintentar | Elegir otra (sólo si hay > 1 emparejada) · Cerrar |
| resto | Reintentar | Cerrar |
| `PRN-DEV-ABSENT` (`:174-188`) | Reintentar | Elegir otra impresora (si > 1) · Olvidar y emparejar otra · Cerrar |
| `PRN-CONN-FAIL` (`:190-197`) | Reintentar | Elegir otra · Cerrar |

El fraseo «Ya cargué papel» convierte el botón en la confirmación de un gesto físico, no en una repetición ciega (`:213-218`). Atasco de papel e impresión desvanecida quedan **sin código ni detección** a propósito: `DLE EOT` no los reporta (CHANGELOG.md:300-301).

### 7.5 Comparación de los tres modelos de error

| | TP | MD | DLG |
|---|---|---|---|
| Mecanismo | excepciones envueltas + `DisplayAlertAsync` | `try/catch` + `MessageLabel` | resultados tipados, sin `try/catch` en el VM |
| Qué ve el usuario | `ex.Message` encadenado | tipo, mensaje, inner y stack (diagnóstico) | `UserMessage` + código estable |
| Original técnico | se pierde en el texto | en pantalla y `Console` | `TechnicalMessage`/`Exception` + `Debug.WriteLine` |

## 8. `Ejemplo_Docs_Printer` — la prosa del dominio

| Carpeta / archivo | Contenido | Estado |
|---|---|---|
| `DOCS/Readme.md` | índice de la carpeta, GIF de prueba, dos enlaces de YouTube | |
| `DOCS/Prompts_Generacion_Apk_ejemplo/Readme.md` | prompt a claude.ai que originó TP y ajuste posterior con Copilot («no compila…», `:86`) | |
| `DOCS/Prompts_Generacion_Apk_ejemplo/ficheros_claude.ia/*` | los 10 archivos que devolvió la IA (servicio, página, ejemplos avanzados de tickets, csproj, README, instalación) | ⚠️ apuntan a net8, `ESCPOS_NET` 4.5.0 y `Plugin.BLE` 3.1.0 (`Thermalprintermaui.csproj:55`, `:58`); usan `_printer.QrCode(...)` (`Thermalprinterservice.cs:340`), que TP terminó reemplazando por texto plano. `Androidmanifest.XML` está **vacío** (0 bytes) |
| `DOCS/Prompts_generacion_documentacion/` | `01_Prompt_chatgpt.md` (qué manuales hacen falta), `02_Prompt_copilot.md` (genera 01–03), `03_Prompt_copilot_manual_borrador.md` (genera 04) | metodología explicada en su `Readme.md` |
| `DOCS/Manuales/01_MANUAL_USUARIO.md` | uso físico: LED, botón, carga, papel, emparejamiento, problemas | generado por IA |
| `DOCS/Manuales/02_MANUAL_TECNICO_ESC_POS.md` | tablas hex de comandos, fuentes A/B, code pages, QR, buffer, estados | generado por IA, ver ⚠️ abajo |
| `DOCS/Manuales/03_GUIA_INTEGRACION_ESCPOS_NET.md` | integración de ESCPOS_NET 2.2.1 en MAUI (USB y Bluetooth, ejemplos, troubleshooting) | generado por IA |
| `DOCS/Manuales/04_Borrador_Manual.md` | manual único «sin planeamiento»: características, usuario, interfaz, desarrollador | borrador |
| `DOCS/Manuales/Readme.md` | resumen de los tres manuales | se autodeclara «lista para producción» (`:223`) |
| `DOCS/Borradores/Readme.md` | borrador de README de TP: comandos usados, permisos, troubleshooting | menciona `Plugin.BLE` (`:16`), que TP no usa |
| `DOCS/Pruebas_realizadas/Readme.md` | TP corrido en un Android 13: GIF de la app y foto del ticket (`:5-12`) | única evidencia de hardware **documentada** del dominio; el GIF es también la portada del `README.md` raíz (`README.md:9`) |
| `DOCS/Ejemplo_MotorDSL/flowchat.bmp` / `.png` | diagrama de flujo del motor (imagen) | no indexado (binario) |

Hardware de referencia: térmica portátil 58 mm modelo «58HB6» (en la publicación de venta que citan los prompts, «58hb4»: `DOCS/Prompts_generacion_documentacion/Readme.md:51`, `:68-70`), 384 puntos de ancho, 32 caracteres por línea en fuente normal (`DOCS/Manuales/Readme.md:166-171`). El nombre `58HB6` es también el del `DeviceProfile` de MD/DLG.

⚠️ Inconsistencias internas de `02_MANUAL_TECNICO_ESC_POS.md` (son de la redacción IA, no del código):
- Titula «Bluetooth 5.0 (BLE…)» pero el perfil es SPP (`:56-60`); el código usa RFCOMM/SPP Classic, no BLE.
- Tabla de `GS !`: `11H` aparece como «1×2» y como «2×2» (`:192-194`).
- `GS k`: la estructura dice tipos `01-09` y la tabla `00-08` (`:231`, `:236-244`).
- `0C` figura como «Avance a inicio» y como «Página nueva» (`:204`, `:206`).
- Los estados usan `GS r` (`:418-429`); la librería MotorDsl sondea con `DLE EOT` (§7.3).
Todo el contenido del hardware (LED, batería, MTU, PIN) es **no verificado**: los propios manuales advierten que «se asumen características estándar» (`DOCS/Manuales/Readme.md:202-203`).

`Ejemplos_Devices/Ejemplos_Devices.slnx:111` lista `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md`, ruta que no existe (el borrador real es `DOCS/Borradores/Readme.md`) → detalle en [09](09_CI-CD-y-Build.md).

## 9. Relación con la app híbrida

`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/MotorDSL/` es la copia de DLG dentro de la app híbrida. Comparado con DLG (diff verificado):

| Archivo | Diferencia |
|---|---|
| `Models/PrinterErrorCatalog.cs`, `PrintFailure.cs`, `PrintResult.cs`, `DiscoverResult.cs`, `BluetoothPermissionResult.cs`, `Services/BluetoothPermissions.cs`, `Pages/OverlayBlueToothThermalPrintPage.xaml` | sólo el namespace (`LibApp.Devices.MotorDSL.*`) |
| `Services/PrinterService.cs` | namespace + implementa `IPrinterService` (costura para tests; `Services/IPrinterService.cs` sólo existe en la híbrida) |
| `ViewModels/PrinterOverlayViewModel.cs` | depende de `IPrinterService`; toma la base `StatusOverlayViewModel` de `LibApp.Devices.Common.ViewModels`; agrega `_reintentarDocumento`, `MostrarObteniendoDocumento`, `MostrarFalloDocumento` y `ReintentarDocumentoCommand` (caso «no se pudo obtener el documento», `PRN-DOC-*`) |
| `Models/DocumentResult.cs` | sólo en la híbrida (`Ok` · `NetworkError` · `InvalidContract`) |

El disparo (`action=print`, `PrintCommandHandler`), los DTOs `Print/*`, el backend `Ejemplo_ws_Blazor` y los tests (`PrinterErrorCatalogTests.cs`, `PrinterOverlayTests.cs`) se documentan en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md). Historia de los cambios de UX de impresión (2026-07-16; hallazgos H-1 a H-4, H-6 y H-7, no hay H-5): `CHANGELOG.md:240-302` → [10](10_Documentacion-Transversal.md); el análisis que cita (`PrintThermal_Motor_Maui.Documentacion/Analisis/Analisis-UX-UI.md`) vive en otro repo del workspace (`Libs/NugetOrg/PrintThermal_Motor_Maui.Documentacion/Analisis/`).

## 10. Preguntas frecuentes y trampas

| Pregunta / trampa | Respuesta |
|---|---|
| ¿Por qué hay dos `IThermalPrinterService`? | ⚠️ Mismo nombre, tipos distintos: `Ejemplo_ThermalPrinter.Services.IThermalPrinterService` (propio, §3) y `MotorDsl.Printing.IThermalPrinterService` (de la librería, usado por MD/DLG). |
| «Escanear» no encuentra mi impresora | TP y DLG sólo listan dispositivos **emparejados** (`BondedDevices`); hay que emparejarla antes desde Ajustes de Android (`TP/Pages/MainPage.xaml.cs:105-108`). |
| En TP elegí «Grande (2x)» y salió más chica | ⚠️ `fontSize > 1` sólo activa `PrintStyle.FontB` (`ThermalPrinterService.cs:169-172`), que según el manual es la fuente chica (9×17 vs 12×24, `02_MANUAL_TECNICO_ESC_POS.md:139`); 2x y 3x se comportan igual. |
| En TP «Código QR impreso correctamente» pero no hay QR | ⚠️ QR y código de barras se imprimen como texto (§3.4). El QR real llega con MotorDsl (`imageType: qrcode`, §4.3). |
| El ticket de TP no suma | ⚠️ Los ítems suman 82,25 (2×15,50 + 25,00 + 3×8,75) pero `subtotal` está fijo en 67,25 (`TP/Pages/MainPage.xaml.cs:206-215`); son datos de ejemplo cableados. |
| ¿Por qué MD/DLG imprimen todo como imagen? | El perfil usa el renderer `"escpos-bitmap"` con `supports_bitmap` y ancho máx. 320 px (`MD/Pages/MainPage.xaml.cs:129-132`): el documento se rasteriza, por eso logo, QR y firma salen sin comandos nativos. |
| ¿La predeterminada se reusa sin preguntar? | Sí, si está emparejada (`GetDefaultIfPresent`); la salida para cambiarla aparece sólo cuando falla (`PRN-DEV-ABSENT` → «Olvidar y emparejar otra»). |
| «Elegir otra» volvía a la misma impresora | Corregido con `forzarSelector: true` (`PrinterOverlayViewModel.cs:73-79`, `:314`). |
| ¿Las esperas se pueden cancelar? | No: la capa Busy no tiene botones (`StatusOverlayViewModel.cs:54-61`); el CHANGELOG lo registra como deuda (`CHANGELOG.md:237-238`). |
| ¿Funciona en iOS? | No en ninguno de los tres (§6.3), aunque MD tenga workflow de CD para iOS. |
| ¿Se probó en hardware? | La única evidencia en el repo es TP en un Android 13 (`DOCS/Pruebas_realizadas/Readme.md:5-12`). El CHANGELOG llama a la impresión «único dominio ya validado en dispositivo» (`CHANGELOG.md:230-231`) sin decir con qué ejemplo (**no verificado**), y para los cambios de UX del 2026-07-16 deja pendiente la verificación en Android real, en particular `ActionBluetoothSettings` (`CHANGELOG.md:302`). |
| ¿Por qué la UI de MD pide «Reescanear» si acepté el permiso? | ⚠️ Espera fija de 3 s tras pedirlo (`MD/Pages/MainPage.xaml.cs:75`); DLG lo resuelve con un permiso awaitable (§6.2). |
| Diferencia de versión MotorDsl | MD usa 1.0.12 y DLG 1.0.13 (§2); el salto de DLG está en `CHANGELOG.md:327` y la entrada de UX deja la librería «no se modificó (sigue en v1.0.13)» (`CHANGELOG.md:244`). |
