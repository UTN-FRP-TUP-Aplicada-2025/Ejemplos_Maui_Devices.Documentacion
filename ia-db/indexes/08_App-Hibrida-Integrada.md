# 08 · App híbrida integrada (WebView + Blazor remoto + puente por URL)

> **Propósito:** destilar el dominio «app híbrida integrada»: una app .NET MAUI que hospeda en un
> `WebView` estándar una web **Blazor Interactive Server remota** y le da acceso a GPS, cámara,
> selfie, QR, teléfono, relay REST e impresora térmica mediante **comandos en la URL**
> interceptados en `Navigating`, con **overlays nativos** de estado.
> **Fuente primaria:** `Ejemplos_Devices/Integrada/**` y `Ejemplos_Devices/Docs/web-hibrida/**`
> (commit `285f6fb`; después, `85384c2` y `cc80ce1` sólo corrigen mayúsculas en rutas del `.slnx` y en
> dos enlaces de `Docs/web-hibrida/maui-hibrido.md`; no cambian código). Visión general del repo: [00_MASTER-INDEX](00_MASTER-INDEX.md).
>
> **Convención de rutas** (relativas al repo de código):
> `Hib/` = `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/` ·
> `Web/` = `Ejemplos_Devices/Integrada/Ejemplo_ws_Blazor/` ·
> `Tst/` = `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida.Tests/` ·
> `Doc/` = `Ejemplos_Devices/Docs/web-hibrida/`.

---

## §1 Mapa de proyectos del dominio

| Proyecto | Ruta | Tipo / TFM | Técnica | Archivos clave | En `.slnx` | Workflow CI |
|---|---|---|---|---|---|---|
| `Ejemplo_Maui_Hibrida` | `Hib/` | App MAUI `net10.0-android` (+ `net10.0-ios` sólo en macOS) | `WebView` remoto + puente por URL + overlays + `LibApp.*` | `MauiProgram.cs`, `ViewModels/MainViewModel.cs`, `Pages/MainPage.xaml`, `LibApp/UrlCommands/**` | Sí, con `<Deploy />` | Sí: `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` (sólo iOS simulador) |
| `Ejemplo_ws_Blazor` | `Web/` | ASP.NET Core `net10.0` (`Microsoft.NET.Sdk.Web`) | Blazor Interactive Server + controllers REST + OpenAPI/Scalar | `Program.cs`, `Components/Pages/Panel.razor`, `Controllers/*.cs` | Sí, **sin** `<Deploy />` | No |
| `Ejemplo_Maui_Hibrida.Tests` | `Tst/` | xUnit `net10.0` con `UseMaui=true` | Linkeo de fuentes platform-free + fakes a mano | `Ejemplo_Maui_Hibrida.Tests.csproj`, `Invariantes.cs`, `Fakes/Fakes.cs` | **No** | No |
| `Ejemplo_Docs_Integrada` | `Ejemplos_Devices/Integrada/Ejemplo_Docs_Integrada/` | Carpeta de docs | — | `Readme.md` (**vacío**, 0 líneas) | Sí, como `<File>` en `/Ejemplos/Integrada/Docs/` | — |
| Docs `web-hibrida` | `Doc/` | 5 `.md` | Explicación por comando | `maui-hibrido.md`, `lectura-qr.md`, `captura-foto.md`, `llamada.md`, `envio-api.md` | Carpeta `/Docs/web-hibrida/` declarada **vacía** | — |

Fuente: `Ejemplos_Devices/Ejemplos_Devices.slnx` (carpeta `/Ejemplos/Integrada/`), `.github/workflows/`.

### §1.1 NuGets por proyecto (leídos de los `.csproj`)

| Proyecto | Paquete | Versión |
|---|---|---|
| Hibrida | `Microsoft.Maui.Controls` | 10.0.80 |
| Hibrida | `Microsoft.Extensions.Logging.Debug` | 10.0.9 |
| Hibrida | `CommunityToolkit.Maui` / `.Core` | 14.2.0 / 14.2.0 |
| Hibrida | `CommunityToolkit.Maui.Camera` | 6.1.0 |
| Hibrida | `CommunityToolkit.Mvvm` | 8.4.2 |
| Hibrida | `BarcodeScanning.Native.Maui` | 3.0.4 |
| Hibrida | `AdamE.Google.iOS.GoogleUtilities` (sólo RID `iossimulator-x64`) | 8.1.0.3 |
| Hibrida | `MetadataExtractor` · `SkiaSharp` | 2.9.0 · 3.119.2 |
| Hibrida | `MotorDsl.Core`, `.Parser`, `.Rendering`, `.Extensions`, `.Printing.Abstractions`, `.Bluetooth`, `.Maui` | 1.0.13 (los 7) |
| Tests | `Microsoft.NET.Test.Sdk` · `xunit` · `xunit.runner.visualstudio` | 17.11.1 · 2.9.2 · 2.8.2 |
| Tests | `CommunityToolkit.Mvvm` · `MotorDsl.Core` · `MotorDsl.Printing.Abstractions` | 8.4.2 · 1.0.13 · 1.0.13 |
| Blazor | `Microsoft.AspNetCore.OpenApi` · `Scalar.AspNetCore` | 10.0.8 · 2.14.14 |
| Blazor (tool) | `dotnet-ef` en `dotnet-tools.json` (no hay EF en el código) | 10.0.8 |

Totales: Hibrida 17 paquetes; Tests 6; Blazor 2. Fuentes: `Hib/Ejemplo_Maui_Hibrida.csproj:89-123`,
`Tst/Ejemplo_Maui_Hibrida.Tests.csproj:19-29`, `Web/Ejemplo_ws_Blazor.csproj:10-13`, `Web/dotnet-tools.json`.

### §1.2 Identidad y plataformas de la app

| Dato | Valor | Fuente |
|---|---|---|
| `ApplicationId` / `CFBundleIdentifier` | `com.ejemplos.devices.integrada.hibrida` | `Hib/Ejemplo_Maui_Hibrida.csproj:18`, `Hib/Platforms/iOS/Info.plist:47` |
| `ApplicationTitle` / `CFBundleDisplayName` | `Web Hibrida` | `csproj:16`, `Info.plist:44` |
| Mínimos | iOS 15.0 · Android API 25 (manifest: `targetSdkVersion 36`) | `csproj:25-26`, `AndroidManifest.xml:114` |
| XAML | `MauiXamlInflator=SourceGen` | `csproj:14` |
| Android | `max-page-size=16384`, RIDs arm/arm64/x86/x64, APK (no AAB) | `csproj:33-50` |
| iOS Release | intérprete + trimming `SdkOnly`, AOT sólo en `ios-arm64` | `csproj:54-64` |

---

## §2 Arquitectura y flujo

### §2.1 Dos canales encadenados

```
CONTENEDOR MAUI (Hib/)                                   WEB REMOTA (Web/, https://aplicada.somee.com)
┌──────────────────────────────────────────────┐        ┌─────────────────────────────────────────┐
│ MainPage: RefreshView > WebView Source={Url}  │──GET──►│ Blazor Interactive Server (prerender ON) │
│   Canal A: circuito SignalR de Blazor (web)   │◄──────►│ @onclick → NavigateTo(url, forceLoad)    │
│   Canal B: Navigating → MainViewModel         │◄─URL───│   "/panel?qr=qr&param=contenidoQR"       │
│            → UrlCommandDispatcher.Plan/Execute│        │                                         │
│            → IUrlCommandHandler (7)           │──JS───►│ document.getElementById(id)…            │
│            → overlays StatusOverlayView (4)   │──URL──►│ ?Latitud=…&Longitud=…&_ts=N (GPS sust.)  │
└──────────────────────────────────────────────┘        └─────────────────────────────────────────┘
```

- La web dispara el Canal B desde un `@onclick` (Canal A): si el circuito no conecta, los botones
  web que piden dispositivos tampoco responden (`Doc/maui-hibrido.md` §1, §3).
- Tres botones nativos del pie (`Geo Pos`, `Llamar`, `Leer QR`) llaman a `DispatchAsync` sin
  pasar por la web; el cuarto, `Volver`, no usa el dispatcher: reasigna `Url = "https://aplicada.somee.com"`
  (`Hib/Pages/MainPage.xaml:52-57`, `Hib/ViewModels/MainViewModel.cs:51-79`, `:146-150`). La URL
  inicial la fija `Hib/Pages/MainPage.xaml.cs:17` (las alternativas quedan comentadas en `:13-15`).

### §2.2 Árbol de `LibApp` (namespaces `LibApp.*`, distintos del `RootNamespace` `Ejemplo_Maui_Hibrida`)

```
Hib/LibApp/
├── CustomWebView/
│   ├── Behaviors/   IWebViewBridge · WebViewBridge (singleton de eventos) · WebViewBridgeBehavior (toca el control)
│   └── Converts/    WebNavigatingEventArgsConverter + WebNavigatedEventArgsConverter (mismo archivo)
├── UrlCommands/     IUrlCommandHandler · UrlCommandDispatcher · UrlPlan · BridgeOutcome · CommandDelivery
│   └── Handlers/    Gps · Call · Camera · Selfie · Qr · SendApi · Print  (+ Readme.md vacío)
└── Devices/         (README.md vacío)
    ├── Common/      StatusOverlayView(.xaml) · StatusOverlayViewModel · IUiDispatcher/MainThreadDispatcher
    ├── GPS/         GpsService/IGpsService · GpsOverlayViewModel · Models (GpsResult, GpsFailure, GpsErrorCatalog…) · ApiRelayService
    ├── Phone/       CallService/ICallService · CallOverlayViewModel · Models (CallResult, CallFailure+CallErrorCatalog, CallMode, ApiCallResult)
    ├── Networks/    NetworkService/INetworkService · NetworkOverlayViewModel · NetworkResult
    ├── MotorDSL/    PrinterService/IPrinterService · BluetoothPermissions · PrinterOverlayViewModel · Models · DTOs/Print + DTOs/LocationDto · Pages/OverlayBlueToothThermalPrintPage (vacía) · README.md
    ├── Camera/      Pages/MyMediaPickerPage · MyMediaSelfiePickerPage · README.md
    ├── Images/      IImageService · ImageDeviceAutoRotateService · SelfieMaskDrawable · README.md
    └── QRLector/    Pages/QRLectorPage · Models/QRContent  (carpeta Services/ declarada vacía, csproj:149-151)
```

### §2.3 Composición (DI) — `Hib/MauiProgram.cs`

| Registro | Ciclo | Línea |
|---|---|---|
| `UseMauiCommunityToolkit`, `…Core`, `…Camera`, `UseBarcodeScanning` | — | `:43-45`, `:57` |
| `AddMotorDslEngine` + perfiles `thermal_58mm`(32, escpos-bitmap), `a4-pdf`(80, pdf), `pdf`(48, pdf) + `AddMotorDslMaui` + `AddBluetoothPrinterTransport` | — | `:61-70` |
| `IPrinterService→PrinterService`, `PrinterOverlayViewModel` | Singleton | `:83-84` |
| `IGpsService`, `Connectivity.Current`, `IUiDispatcher→MainThreadDispatcher`, `INetworkService`, `ICallService`, `ApiRelayService` | Singleton | `:93-98` |
| `IWebViewBridge→WebViewBridge`, `IImageService→ImageDeviceAutoRotateService` | Singleton | `:102-103` |
| `MyMediaPickerPage`, `MyMediaSelfiePickerPage` | Transient | `:104-105` |
| `Gps/Network/CallOverlayViewModel` | Singleton | `:109-111` |
| 7 × `IUrlCommandHandler` (orden = precedencia) + `UrlCommandDispatcher` | Singleton | `:116-123` |
| `MainViewModel`, `MainPage` | Singleton | `:127-128` |

Rutas Shell: `MyMediaPickerPage`, `MyMediaSelfiePickerPage`, `QRLectorPage` (`Hib/AppShell.xaml.cs:12-14`).
`IThermalPrinterService` (que consume `PrinterService`) lo aporta MotorDsl (no verificado: el registro
vive dentro de los NuGets).

---

## §3 Puente de comandos por URL: clasificación vs ejecución

### §3.1 Piezas

| Pieza | Rol | Fuente |
|---|---|---|
| `IUrlCommandHandler` | `CanHandle(url)` + `HandleAsync(url)`; *default interface members*: `CancelsNavigation => true`, `DeliveryFor(url) => None`, `OnMatchedSync(url) {}` | `Hib/LibApp/UrlCommands/IUrlCommandHandler.cs:10-43` |
| `UrlPlan` | Resultado de clasificar **una vez**: `Matches`, `Cancel`, `Primary` (= `Matches[0]`), `UrlPlan.Ninguno` | `…/UrlPlan.cs:11-22` |
| `UrlCommandDispatcher.Plan` | **100 % síncrono**: evalúa `CanHandle` una vez por handler; `Cancel` = OR de `CancelsNavigation`; corre `OnMatchedSync` para **todos** los matches | `…/UrlCommandDispatcher.cs:24-60` |
| `UrlCommandDispatcher.ExecuteAsync` | Ejecuta `plan.Primary` (*first-match-wins*); sin handler devuelve `BridgeOutcome(false, null)` | `:62-89` |
| `DispatchAsync(url)` | `ExecuteAsync(Plan(url), url)`; lo usan los botones nativos | `:92` |
| `IsCommand(url)` | Conservado «por compatibilidad de firma»; `MainViewModel` ya no lo usa | `:98` |
| `BridgeOutcome(CancelNavigation, NavigateTo?)` | `NavigateTo` sólo para modo `Substitution` | `…/BridgeOutcome.cs:5` |
| `CommandDelivery` | `None` (overlay), `Injection` (JS al DOM), `Substitution` (re-navegar con valores en la query) | `…/CommandDelivery.cs:8-33` |

### §3.2 `MainViewModel.Navigating` (el interceptor)

```
Navigating(e)                                   [RelayCommand(AllowConcurrentExecutions = true)]  :90
 ├─ FASE SÍNCRONA (nada de await antes de e.Cancel)
 │   plan = _dispatcher.Plan(e.Url)                                                            :97
 │   if plan.Cancel → e.Cancel = true                                                          :99-100
 │   if !plan.HasMatches → IsRefreshing=false; return   (navegación normal)                    :102-106
 │   if plan.Cancel && comandoEnCurso → return           (descarta el click duplicado)         :112-116
 └─ FASE ASÍNCRONA
     comandoEnCurso = plan.Cancel; outcome = await ExecuteAsync(plan, e.Url)                   :119-123
     if outcome.NavigateTo != null → Url = outcome.NavigateTo                                  :125-126
     finally: comandoEnCurso=false; IsRefreshing=false                                         :128-132
```

- `AllowConcurrentExecutions = true` es imprescindible: con el default el 2.º `Navigating` no
  ejecutaría el comando, `e.Cancel` no se fijaría y la página recargaría (ADR-0009, citado en `:86-89`).
- `Navigated` sólo cierra el spinner y avisa al overlay de Red (`:136-144`); **no actualiza `Url`**.

### §3.3 Invariantes que el dispatcher verifica en `DEBUG`

| Invariante | Dónde | Efecto |
|---|---|---|
| Si se cancela, el `Primary` debe ser cancelable (si no: «navegación muerta») | `UrlCommandDispatcher.cs:46-57` | `Debug.Fail` |
| Un handler que declara `Substitution` debe devolver `NavigateTo` no nulo | `:77-85` | `Debug.Fail` |

Los tests que fuerzan esa combinación se saltean en Debug (`Tst/UrlCommandDispatcherTests.cs:79-83`).
Los comentarios citan «Plan 1 §3–§7»; ese documento no está en `Analisis/` del repo hermano
(sólo `Plan-Armonizacion-Overlays.md`) → «Plan 1» (no verificado).

---

## §4 Catálogo de comandos

Orden de evaluación = orden de registro (`Hib/MauiProgram.cs:116-122`). Todos son cancelables (default).

| # | Marcador en la URL (`Contains`, ignore case) | Handler | Entrega | Destino / efecto | Disparo web (`Web/…/Panel.razor`) | Botón nativo |
|---|---|---|---|---|---|---|
| 1 | `coordenadas=coordenadas` | `GpsCommandHandler` | `Injection` con `param`, `Substitution` sin `param` | `el.textContent="Latitud: …, Longitud: …"` o re-navegación | `:174` (sin param), `:181` (`param=contenidoCoordenada`) | `Geo Pos` (`MainViewModel.cs:68-79`) |
| 2 | `phone=phone` | `CallCommandHandler` | `None` | Llamada directa al número fijo `3434807427` + overlay | `:205`, combo `:207-216` | `Llamar` (`:53-56`) |
| 3 | `photo=photo` (+`param`) | `CameraCommandHandler` | `Injection` (no lo declara) | `id.src` y `id.value` = `data:image/jpeg;base64,…` | `:228` (`param=imgFoto1`) | — |
| 4 | `selfie=selfie` (+`param`) | `SelfieCommandHandler` | `Injection` (no lo declara) | ídem foto, cámara frontal | `:252` (`param=imgSelfie1`) | — |
| 5 | `qr=qr` (+`param`) | `QrCommandHandler` | `Injection` (no lo declara) | `id.textContent` = JSON de `List<QRContent>` | `:285-286` (`param=contenidoQR`) | `Leer QR` (`:60-63`, URL `qr=qr&param=contenidoQR`) |
| 6 | `sendApi=sendApi` (+`httpMethod`,`url`,`param`,`body`) | `SendApiCommandHandler` | `Injection` (no lo declara) | `id.textContent` = `{ok,status,body}` | `:273-276` | — |
| 7 | `action=print` | `PrintCommandHandler` | `None` | GET comprobante → MotorDsl → impresión BT + overlay | `:319` | — |

Sólo `GpsCommandHandler` sobrescribe `DeliveryFor` (`GpsCommandHandler.cs:41-43`); los demás heredan
`None`, por lo que la aserción de `Substitution` no se les aplica. ⚠️ El XML-doc de
`CommandDelivery.Injection` lista como «casos actuales» foto, selfie, QR y sendAPI
(`Hib/LibApp/UrlCommands/CommandDelivery.cs:19`), pero esos handlers devuelven `None` por el default
de la interfaz (`IUrlCommandHandler.cs:29`): inyectan de hecho, sin declararlo. Detalle de cada flujo en `Doc/*.md`
(ver §12 para su vigencia).

### §4.1 Mecánica por handler

| Handler | Espera del dispositivo | Serialización del JS | Fuente |
|---|---|---|---|
| Camera / Selfie | `Shell.GoToAsync(nameof(MyMedia…Page), {OnPhotoCallback})` + `TaskCompletionSource<string?>`; normaliza con `IImageService.ProcesarPhotoAsync(Stream)` y borra el temporal | **Interpolación cruda** entre comillas simples (`'{targetId}'`, `'{dataUri}'`) | `CameraCommandHandler.cs:34-64`, `SelfieCommandHandler.cs:33-61` |
| Qr | `Navigation.PushAsync(new QRLectorPage())` sobre `Application.Current.Windows[0].Page` y `await page.ResultadoTask.Task` | `JsonSerializer.Serialize` del id y del valor | `QrCommandHandler.cs:39-48` |
| Gps | `GpsOverlayViewModel.SolicitarGeolocalizacion()` | `JsonSerializer.Serialize` + `if(el)` | `GpsCommandHandler.cs:48-66` |
| SendApi | `ApiRelayService.SendAsync` (allowlist `geolocate.somee.com`, timeout 30 s, body sólo en POST) | `'{JsonSerializer.Serialize(payload)}'` entre comillas simples | `SendApiCommandHandler.cs:29-85`, `Hib/LibApp/Devices/GPS/ApiRelayService.cs:15-64` |
| Call | `CallOverlayViewModel.LlamarAsync(numero, CallMode.Direct)` | — | `CallCommandHandler.cs:22-26` |
| Print | GET `https://aplicada.somee.com/api/Tikects/comprobante` (30 s) → valida contra DTO `PrintDocument` → `IDocumentEngine.Render(json, DeviceProfile("58HB6",32,"escpos-bitmap"))` → `PrinterOverlayViewModel.ImprimirAsync` | — | `PrintCommandHandler.cs:17-130` |

---

## §5 GPS: los dos modos de `CommandDelivery`

| Aspecto | `Injection` (URL con `param`) | `Substitution` (URL sin `param`) |
|---|---|---|
| Quién lo usa | Botón web «Solicitar GeoPosicion» (`Panel.razor:181`) | Botón web «Solicitar coordenadas» (`Panel.razor:174`, va a `/geolocalizacion`) y botón nativo `Geo Pos` |
| Éxito | Inyecta `Latitud: x, Longitud: y` en `#param`; `NavigateTo=null` | Reemplaza `coordenadas=coordenadas` por `Latitud=x&Longitud=y&` y agrega `_ts=N` (nonce `Interlocked`) |
| Fallo del GPS | No inyecta ni re-navega: la continuación es el overlay de error | **Re-navega igual** con el centinela `0.0`/`0.0` |
| Cultura | `InvariantCulture` (punto decimal) en ambos | ídem |
| Fuente | `GpsCommandHandler.cs:50-69` | `:79-91` |

Lectura de la web: `Panel.razor:186-199` (`[SupplyParameterFromQuery] double`, muestra sólo si ambos
≠ 0) y `GeoLocalizacion.razor:21-36` (`string`, `0.0` si faltan). La lectura es **fresca**
(`GetLocationAsync`, `Medium`, 15 s, sin `GetLastKnownLocationAsync`) y el overlay se oculta en
`GpsOverlayViewModel.Procesar` sin depender de `Navigated` (Opción A, ADR-0009;
`GpsService.cs:37-43`, `GpsOverlayViewModel.cs:40-54`).

---

## §6 Overlays de dispositivo

### §6.1 Base común

`StatusOverlayViewModel` (abstracta): máquina `OverlayMode {None, Busy, Error}`; `ShowBusy(título,
subtítulo, gif)` limpia la botonera; `ShowError(glyph, título, mensaje, params OverlayAction[])`;
`Hide()`. `OverlayAction(Text, Command, Style = Primary)`; el estilo `Secondary` lo pinta un
`DataTrigger` (`Hib/LibApp/Devices/Common/ViewModels/StatusOverlayViewModel.cs:11-77`,
`…/Controls/StatusOverlayView.xaml:21-58`). Gifs: `satelite.gif`, `reconexion.gif`, `timer.gif` en
`Hib/Resources/Images/`.

### §6.2 Los cuatro overlays

| Overlay (orden en `MainPage.xaml:44-50`, el último queda arriba) | Disparo | Particularidad | Códigos |
|---|---|---|---|
| GPS | `GpsCommandHandler` | Devuelve **la misma variante** de `GpsResult` que recibió (I-5) | `GPS-PERM-ASK`, `-PERM-DENIED`, `-PERM-MDM`, `-OFF`, `-UNSUPPORTED`, `-NO-SIGNAL`, `-CANCELLED`, `-UNKNOWN` (`GpsErrorCatalog.cs:16-23`) |
| Red | **Reactivo**: `INetworkService.ConnectivityChanged` + `Navigated` fallido | Único que desmonta el `WebView` (`RefreshView.IsVisible` = inverso de `NetworkOverlayViewModel.IsVisible`, `MainPage.xaml:24-26`); `_needsReload` distingue «recargar» de «sólo destapar»; sonda a `msftconnecttest.com` y reporta el host del sitio | sin códigos; variantes `NetworkResult` (`NetworkService.cs:15-16`, `NetworkOverlayViewModel.cs:27-129`) |
| Llamada | `CallCommandHandler` | Guarda último número/modo para reintentar (VM singleton) | `TEL-PERM-ASK`, `-PERM-DENIED`, `-PERM-MDM`, `-UNSUPPORTED`, `-BAD-NUMBER`, `-CANCELLED`, `-NO-ACTIVITY`, `-UNKNOWN` (`CallFailure.cs:24-31`) |
| Impresión (máxima prioridad) | `PrintCommandHandler` | Predeterminada en `Preferences`, alias, sufijo de MAC (2 octetos) para homónimas, «Olvidar y emparejar otra», reintento del GET | `PRN-DOC-NET/-CONTRACT/-RENDER`, `PRN-BT-OFF/-PERM`, `PRN-DEV-NONE/-ABSENT`, `PRN-CONN-FAIL`, `PRN-HW-PAPER/-COVER/-OTHER`, `PRN-LINK-LOST`, `PRN-TIMEOUT`, `PRN-UNKNOWN` (`PrinterErrorCatalog.cs:20-39`) |

Patrón común de fallos: `XxxFailure(Code, Title, UserMessage, TechnicalMessage)` con
`DisplayMessage = "{UserMessage}\n\n{Code}"`; tres tipos gemelos **no unificados a propósito**
(`GpsFailure.cs:7-12`). Un código por acción del usuario; se agregan, no se renombran.

### §6.3 Las páginas de captura

| Página | Navegación | Cámara | Devuelve |
|---|---|---|---|
| `MyMediaPickerPage` | Shell + `[QueryProperty("OnPhotoCallback")]` | Trasera, seleccionada en `OnNavigatedTo` | Path temporal en `CacheDirectory`; navega `..` **antes** del callback (`MyMediaPickerPage.xaml.cs:301-304`) |
| `MyMediaSelfiePickerPage` | ídem | Frontal, preseleccionada antes de adjuntar al árbol; máscara oval `SelfieMaskDrawable` | Callback **antes** de navegar `..` (`MyMediaSelfiePickerPage.xaml.cs:374-377`) |
| `QRLectorPage` | `PushAsync` directo (no usa la ruta Shell registrada) | `BarcodeScanning` `CameraView`, `QRCode,Code39` | `ResultadoTask` (`Interlocked` evita doble entrega); «Volver» entrega lista vacía (`QRLectorPage.xaml.cs:80-85`) |

---

## §7 Servicios de plataforma (`LibApp.Devices`)

| Interfaz → implementación | Plataforma | Puntos clave | Fuente |
|---|---|---|---|
| `IGpsService → GpsService` | Essentials | Permiso `LocationWhenInUse`; `ShouldShowRationale` sólo Android; `OpenLocationSettings` = intent Android, fallback a ajustes de app | `GPS/Services/GpsService.cs:16-115` |
| `ICallService → CallService` | `#if ANDROID` | `Direct` = `Intent.ActionCall` + permiso `Phone`; resto → `PhoneDialer` | `Phone/Services/CallService.cs:24-137` |
| `INetworkService → NetworkService` | `IConnectivity` | Evento normalizado a `bool`; sonda con marcador para portal cautivo; DNS → host del sitio | `Networks/Services/NetworkService.cs:10-97` |
| `IPrinterService → PrinterService` | BT Classic SPP sólo Android | Chequea adaptador (`BluetoothManager`) y `BLUETOOTH_CONNECT` antes de descubrir; desenvuelve `InnerException` | `MotorDSL/Services/PrinterService.cs:26-206` |
| `BluetoothPermissions` | Android | API ≥ 31: `BLUETOOTH_SCAN`+`CONNECT`; < 31: `ACCESS_FINE_LOCATION` | `MotorDSL/Services/BluetoothPermissions.cs:9-21` |
| `IImageService → ImageDeviceAutoRotateService` | SkiaSharp + MetadataExtractor | EXIF → rotación; 50 % o lado mayor ≤ 1000 px; JPEG q75 | `Images/ImageDeviceAutoRotateService.cs:9-89` |
| `ApiRelayService` (sin interfaz) | `HttpClient` | Allowlist, timeout, mapeo a `ApiCallResult` | `GPS/ApiRelayService.cs:12-65` |
| `IUiDispatcher → MainThreadDispatcher` | MAUI | Sólo lo usa el overlay de Red (para testear sin dispositivo) | `Common/Services/IUiDispatcher.cs:12-21` |
| `IWebViewBridge → WebViewBridge` + `WebViewBridgeBehavior` | MAUI | El singleton sólo emite eventos; la behavior hace `Reload`/`EvaluateJavaScriptAsync` en UI thread, *fire-and-forget*; propaga `BindingContext` a mano | `CustomWebView/Behaviors/*.cs` |

---

## §8 Backend Blazor (`Ejemplo_ws_Blazor`)

| Pieza | Detalle | Fuente |
|---|---|---|
| Pipeline | `AddInteractiveServerComponents(DetailedErrors)`, controllers, OpenAPI + Scalar **siempre** (el `if Development` está comentado), `UseForwardedHeaders` (listas `KnownNetworks/Proxies` vaciadas para el proxy de somee) antes de `UseHttpsRedirection`, antiforgery, `MapStaticAssets` | `Web/Program.cs:8-60` |
| `csproj` | `BlazorDisableThrowNavigationException=true` (todos los `NavigateTo(…, forceLoad: true)` del panel son navegaciones de página completa que la app intercepta) | `Web/Ejemplo_ws_Blazor.csproj:7` |
| Status codes | `UseStatusCodePagesWithReExecute("/not-found")`; `Routes.razor` declara `NotFoundPage` | `Web/Program.cs:49`, `Web/Components/Routes.razor:1` |
| Rutas | `/` y `/datos` → `Datos.razor` · `/panel` → `Panel.razor` · `/geolocalizacion` · `/redirigir` · `/not-found` · `/Error` | `Web/Components/Pages/*.razor` |
| Render | Por página: `@rendermode InteractiveServer` + `[StreamRendering]` (prerender ON) | `Panel.razor:7-8`, `Datos.razor:4-5` |
| `GeoReporterController` | `POST api/GeoReporter/track` (`LocationDto`) → `"lat-lng"` | `Web/Controllers/GeoReporterController.cs:17-41` |
| `TikectsController` | `GET api/Tikects/comprobante` → `PrintDocument` hardcodeado (texto, separadores, contenedores, logo bitmap, QR) | `Web/Controllers/TikectsController.cs:27-160` |
| `PagoFakeController` | `POST api/pagofake/pago` (302 cross-host) y `GET api/pagofake/pago-form` (autosubmit a httpbin) para reproducir el «salto a Safari» | `Web/Controllers/PagoFakeController.cs:11-26` |
| DTOs | `PrintDocument/PrintNode/PrintStyle/N` y `LocationDto` **idénticos** a los de `Hib/LibApp/Devices/MotorDSL/DTOs/` salvo el namespace | `diff` de ambos directorios |
| Perfiles de lanzamiento | `http://localhost:5284`, `https://localhost:8080` | `Web/Properties/launchSettings.json` |

### §8.1 Tarjetas de `Panel.razor`

| Tarjeta | Handler `@onclick` | URL generada | Elemento destino |
|---|---|---|---|
| Solicitar coordenadas | `OnSolicitarCoordenadas` | `/geolocalizacion?coordenadas=coordenadas` | (re-navega; div comentado en `:20`) |
| Solicitar GeoPosicion | `OnSolicitarGeoposicion` | `/panel?coordenadas=coordenadas&param=contenidoCoordenada` | `#contenidoCoordenada` (`:32`) |
| Llamar / Llamar y reportar | `OnLLamar` / `OnLLamarYSendAPI` | `/panel?phone=phone[&sendAPI=…]` | — |
| Tomar foto | `OnTomarFoto` | `/panel?photo=photo&param=imgFoto1` | `#imgFoto1`; «Mostrar contenido» lee `#inputFoto1` |
| Tomar selfie | `OnTomarSelfie` | `/panel?selfie=selfie&param=imgSelfie1` | `#imgSelfie1`; lee `#inputSelfie1` |
| Prueba de interacción | `OnTest` | — (sólo Canal A) | `@ContenidoTest` |
| Post API | `OnSendAPI` | `sendAPI=…&httpMethod=Post&url=https%3a…geolocate.somee.com…&param=contenidoSendAPI&body=…` | `#contenidoSendAPI` |
| Lectura QR | `OnQR` | `/panel?qr=qr&param=contenidoQR` | `#contenidoQR` |
| Redirigir | `OnRedirect` | `https://www.google.com` (no es comando) | — |
| Impresión de ticket | `OnImprimirTicker` (`OnGenerarTicket` vacío) | `/panel?action=print` | — |

---

## §9 Permisos y configuración por plataforma

### §9.1 Android (`Hib/Platforms/Android/`)

| Ítem | Valor | Línea |
|---|---|---|
| `application` | `usesCleartextTraffic="true"`, `networkSecurityConfig="@xml/network_security_config"` | `AndroidManifest.xml:8-9` |
| Ubicación | `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`; `uses-feature gps required=false` | `:16`, `:22`, `:34` (repetidos en `:104-105`) |
| Red | `ACCESS_NETWORK_STATE` (`:40`, repetido `:92`), `INTERNET` (`:93`) | |
| Teléfono | `CALL_PHONE` (`:47`), `READ_PHONE_STATE` (`:49`, sin uso visible en el código) | |
| Bluetooth | `BLUETOOTH`, `BLUETOOTH_ADMIN`, `BLUETOOTH_SCAN neverForLocation`, `BLUETOOTH_CONNECT` | `:57-62` (repetidos `:96-101`) |
| Cámara | `CAMERA`, `RECORD_AUDIO` | `:82-83` |
| `queries` | `DIAL tel`, `REQUEST_ENABLE` (×2), `IMAGE_CAPTURE` | `:68-76`, `:85-89`, `:107-111` |
| Trust anchors | `base-config` cleartext + `system`; `domain-config aplicada.somee.com` (con subdominios) = `system` + `@raw/isrg_root_x2` + `@raw/root_ye` + `@raw/ye2` (cadena según el comentario: `aplicada.somee.com` ← YE2 ← Root YE ← ISRG Root X2; se embeben raíz + intermedios) para Android viejo (Moto g42) | `Resources/xml/network_security_config.xml:3-22` (comentario `:9-13`); `.pem` en `Resources/raw/` |

Decisión: ADR-0008 «trust anchors embebidos» (repo hermano, `docs/04-decisions/0008-…md`) y
`Ejemplos_Devices/Docs/Certificados-SSL/README.md` (fuera de este dominio). El script que baja los
`.pem` (`Utilities/download_sectigo.ps1`) está en [09_CI-CD-y-Build](09_CI-CD-y-Build.md).

### §9.2 iOS (`Hib/Platforms/iOS/`)

| Ítem | Valor | Línea |
|---|---|---|
| `NSCameraUsageDescription` | QR + foto/selfie | `Info.plist:58-60` |
| `NSLocationWhenInUseUsageDescription` | texto heredado sobre «infracciones de tránsito» | `:62-64` |
| `LSApplicationQueriesSchemes` | `tel` | `:66-70` |
| ATS | **sin** `NSAppTransportSecurity` | todo el archivo |
| Bluetooth | sin claves (la impresión es sólo Android: `PrinterService.IsSupported`) | `PrinterService.cs:26-31` |
| Entitlements | `aps-environment=development` + `get-task-allow` en ambos `.plist` (no hay código de push) | `Entitlements*.plist` |

---

## §10 Pruebas (`Ejemplo_Maui_Hibrida.Tests`)

- **Linkeo de fuentes**, no `ProjectReference` (un `net10.0` no puede referenciar un
  `net10.0-android`; extraer `LibApp` contradiría ADR-0001). Se linkean `StatusOverlayViewModel`,
  `Models/*.cs` de GPS/Phone/Networks/MotorDSL, **sólo las interfaces** de servicio, `IUiDispatcher`,
  los 4 overlays VM, `UrlCommands/*.cs` (comodín), `GpsCommandHandler` y `MainViewModel`
  (`Tst/Ejemplo_Maui_Hibrida.Tests.csproj:31-84`).
- Fakes escritos a mano (sin librería de mocking): `FakeGpsService`, `FakeCallService`,
  `FakeNetworkService` (`EmitirConectividad`), `FakeUiDispatcher` (inline), `FakeWebViewBridge`,
  `FakePrinterService` (`Tst/Fakes/Fakes.cs`).
- 78 tests (`[Fact]`/`[Theory]`) en 9 clases; ojo al contar con `grep`: `UrlCommandDispatcherTests`
  tiene dos atributos para un mismo test bajo `#if DEBUG`/`#else` (`:79-83`):

| Clase | Tests | Qué fija |
|---|---|---|
| `BaseOverlayTests` | 7 | Máquina de 3 estados; `ShowBusy` limpia botonera; `Primary` por defecto |
| `UrlCommandDispatcherTests` | 5 | Eje A (OR de cancelación), `OnMatchedSync` para todos, *first-match-wins* (sólo Release) |
| `NavigatingReentrancyTests` | 3 | Guard `comandoEnCurso` (ADR-0009 D1) |
| `GpsCommandHandlerTests` | 6 | Cultura `es-AR`, nonce, centinela `0.0`, `DeliveryFor` |
| `GpsOverlayTests` · `CallOverlayTests` · `NetworkOverlayTests` | 11 · 7 · 9 | Invariantes I-1..I-5 por variante (reflexión sobre los `record` sellados) |
| `PrinterErrorCatalogTests` · `PrinterOverlayTests` | 11 · 19 | Clasificación `PRN-*`; selector, homónimas, H-1, H-2, H-4..H-6 (no hay H-3) |

Invariantes (`Tst/Invariantes.cs`): I-1 una pantalla con salida, I-2 toda variante cubierta,
I-3 sin texto técnico, I-4 un solo primario, I-5 (en `GpsOverlayTests`) no colapsar la variante.

---

## §11 CI y prueba end-to-end

Sólo lo propio de este dominio; toolchain (Xcode por `gdown`, SDK, workloads), firma ad-hoc y
artefactos se detallan en [09_CI-CD-y-Build](09_CI-CD-y-Build.md).

| Aspecto | Valor | Fuente |
|---|---|---|
| Workflow | `CD iOS - Integrada - Ejemplo_Maui_Hibrida`: el único con `push` a `main` (`paths: Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/**`, excluye `**.md`, `.gitignore`, `.gitattributes`) + `workflow_call`; build `Release` `iossimulator-arm64` | `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:3-15`, `:44-46` |
| E2E | Maestro 1.41.0 + `Utilities/simular_ui.sh` con el flujo `Utilities/end2end/com.ejemplos.devices.integrada.hibrida.yaml` (toca `Geo Pos`, `Llamar`, `Leer QR`, vuelve por coordenada `50%,92%`, `Volver`); paso `continue-on-error` | `:438-457`; flujo `:37-81` |
| No cubre | Android, `Ejemplo_ws_Blazor`, `Ejemplo_Maui_Hibrida.Tests` (ningún workflow ejecuta `dotnet test`) y cambios fuera de `Ejemplo_Maui_Hibrida/**` no disparan el workflow | inventario de `.github/workflows/` |

---

## §12 Documentación `web-hibrida` y vigencia

| Documento | Tema | Vigencia frente al código actual |
|---|---|---|
| `Doc/maui-hibrido.md` (677 l.) | Canal A/B, flujo GPS, diagnóstico iOS (circuito SignalR vs ATS/cookies) | ⚠️ Describe `IsCommand`+`DispatchAsync`, GPS sólo por re-navegación, `GetLastKnownLocationAsync`, `Failure("")`, `UseForwardedHeaders` ausente, botón «Geo Posicionar», 6 handlers y rutas `Behaviors/`, `Services/`, `Models/` previas a `LibApp/` |
| `Doc/lectura-qr.md` (393 l.) | QR | ⚠️ §7 («callback `OnQrCallback` nunca invocado») ya no aplica: el handler usa `PushAsync` + `ResultadoTask` |
| `Doc/captura-foto.md` (326 l.) | Foto y selfie | ⚠️ §8.2 (`param=Selfie1`) y «la selfie no está en `AppShell`» ya no aplican; §8.1 (`inputFoto1` nunca se escribe) **sigue vigente** |
| `Doc/llamada.md` (222 l.) | `phone=phone` | Lógica vigente; números de línea desfasados |
| `Doc/envio-api.md` (244 l.) | `sendAPI` | Vigente en §7 (hook inerte, comillas simples); dice que SendApi es el último handler, hoy lo es `Print` |

Los `.md` declaran «describe, no corrige»; las líneas citadas (`Archivo.cs:NN`) corresponden a
versiones anteriores: usar este índice para ubicaciones actuales.

---

## §13 Trampas de lectura y discrepancias ⚠️

| # | Trampa / discrepancia | Evidencia |
|---|---|---|
| T1 | ⚠️ El comentario de `OnSolicitarCoordenadas` dice que la app **inyecta** en `#contenidoCoordenada`, pero la URL no lleva `param` → modo `Substitution` y re-navega a `/geolocalizacion`. A la inversa, la tarjeta «Solicitar GeoPosicion» (que sí inyecta, sin recargar) dice «en una página nueva». Ambos botones se rotulan «Tomar Coordenadas» | `Web/Components/Pages/Panel.razor:172-174`, `:18-19`, `:30-31` vs `Hib/LibApp/UrlCommands/Handlers/GpsCommandHandler.cs:41-43` |
| T2 | ⚠️ `QrCommandHandler` dice «"Volver" o lista vacía = cancelado (no inyecta)», pero inyecta siempre (con «Volver» escribe `[]` en el destino) | `QrCommandHandler.cs:11` vs `:41-48`; `QRLectorPage.xaml.cs:80-85` |
| T3 | ⚠️ `SendApiCommandHandler` documenta `tipoRequest=Post|Get`; el código lee `httpMethod` | `SendApiCommandHandler.cs:10` vs `:30` |
| T4 | ⚠️ Comentario «no interpolar el body crudo», pero el JSON se envuelve en comillas simples: una `'` o salto de línea en la respuesta rompe el JS | `SendApiCommandHandler.cs:71` vs `:77-78` |
| T5 | ⚠️ Camera/Selfie dicen llamar a `window.recibirFoto(...)`; esa línea está comentada y se escribe `src`/`value` del **mismo** id (un `<img>`): `#inputFoto1`/`#inputSelfie1` nunca reciben el base64 | `CameraCommandHandler.cs:9-10`, `:59-62`; `SelfieCommandHandler.cs:8-9`, `:56-59`; `Panel.razor:58-60`, `:232` |
| T6 | ⚠️ Hook `window.recibirRespuestaApi` escribe en `#apiOut`, que no existe, y ningún handler lo invoca (código inerte) | `Panel.razor:158-163`; `SendApiCommandHandler.cs:72-74` |
| T7 | ⚠️ `Rotate(bitmap, angle, ex, ey)` llama `canvas.Scale` **después** de `DrawBitmap`: los casos EXIF de espejo (2, 4, 5, 7) no se espejan pese a los comentarios «voltear» | `Hib/LibApp/Devices/Images/ImageDeviceAutoRotateService.cs:102-125`, `:178-180` |
| T8 | ⚠️ `GpsErrorCatalog.SinSenalGps` registra como técnico «GetLastKnownLocationAsync y GetLocationAsync devolvieron null», pero el servicio ya no llama a `GetLastKnownLocationAsync` | `GpsErrorCatalog.cs:67` vs `GpsService.cs:37-43` |
| T9 | ⚠️ `GpsCommandHandler` cita `Panel.razor:176` como el lugar que interpreta el centinela; está en `MostrarCoordeandas` (`:192-199`) | `GpsCommandHandler.cs:75` |
| T10 | ⚠️ Subtítulo del overlay GPS «será redirigido automáticamente» también en modo `Injection`, donde no hay redirección | `GpsOverlayViewModel.cs:29-31` |
| T11 | ⚠️ `Program.cs` comenta «UseHttpsRedirection (Program.cs:37)»; hoy está en la línea 52 | `Web/Program.cs:17`, `:52` |
| T12 | ⚠️ `Invariantes.cs` cita `Ejemplos_Maui_Devices.Documentos/Analisis/…`; el repo hermano se llama `…Documentacion`. Sus comentarios (y `CallOverlayTests.cs:51`) describen como «hoy» defectos de GPS/Telefonía ya corregidos | `Tst/Invariantes.cs:9`, `:18-19`, `:53-55` |
| T13 | ⚠️ READMEs de `LibApp` desfasados: MotorDSL registra `AddSingleton<PrinterService>()` (el código usa `IPrinterService`); Camera cita `CommunityToolkit.Maui.Camera 6.0.0`/`Core 14.0.0` (csproj 6.1.0/14.2.0) y titula «App.xaml.cs» un bloque de `AppShell`; Images cita `SkiaSharp 3.119.1` (csproj 3.119.2) | `LibApp/Devices/MotorDSL/README.md:34`; `Camera/README.md:12-13`, `:111`; `Images/README.md:12` |
| T14 | `Geo Pos` arma la URL con `MainViewModel.Url` (la última **asignada**), no con la página que muestra el `WebView`: `Navigated` no actualiza `Url`. Desde la home (`/` = `Datos.razor`) las coordenadas no se muestran | `MainViewModel.cs:68-79`, `:136-144`; `Web/Components/Pages/Datos.razor:2` (modo de binding por defecto de `WebView.Source`: no verificado) |
| T15 | `Leer QR` y `Llamar` nativos despachan URLs sin host: el QR inyecta en `#contenidoQR` de la página actual; si no es `/panel`, `getElementById` da `null` y el error JS se pierde | `MainViewModel.cs:60-63`; `QrCommandHandler.cs:47` |
| T16 | En el combo «Llamar y reportar» sólo corre la llamada (*first-match-wins*: Call antes que SendApi) — documentado en el propio Razor | `Panel.razor:207-216`; `MauiProgram.cs:117`, `:121` |
| T17 | El relay sólo permite `geolocate.somee.com`; un `sendAPI` a `aplicada.somee.com` (donde también corre este backend) da `Blocked` | `ApiRelayService.cs:15-18` |
| T18 | `ApiRelayService` vive en `LibApp.Devices.GPS` pero su resultado `ApiCallResult` en `LibApp.Devices.Phone.Models` | `GPS/ApiRelayService.cs:5`; `Phone/Models/ApiCallResult.cs:1` |
| T19 | Los perfiles MotorDsl registrados en DI no los usa `PrintCommandHandler`, que arma su propio `DeviceProfile("58HB6", …)` | `MauiProgram.cs:62-67` vs `PrintCommandHandler.cs:75-78` |
| T20 | `QRLectorPage` tiene ruta Shell registrada pero se abre con `new` + `PushAsync`; `OverlayBlueToothThermalPrintPage` está vacía y sin uso | `AppShell.xaml.cs:14`; `QrCommandHandler.cs:39-40`; `MotorDSL/Pages/*.xaml` |
| T21 | `pago-fake-web.html`: el botón hace `submit()` de `#fpost`, pero el form se llama `form1`. `Redirigir.razor` declara dos forms con `id="fpost"` | `Web/wwwroot/pago-fake-web.html:13`, `:16`; `Web/Components/Pages/Redirigir.razor:43`, `:73` |
| T22 | Permisos duplicados en `AndroidManifest.xml` (BT, ubicación, red, `queries`): copia deliberada de los manifiestos de los PoC, sin efecto funcional | `AndroidManifest.xml:16-22` vs `:104-105`; `:57-62` vs `:96-101` |
| T23 | `WebNavigatedEventArgsConverter` no tiene archivo propio: vive en `WebNavigatingEventArgsConverter.cs` | `CustomWebView/Converts/WebNavigatingEventArgsConverter.cs:14-21` |
| T24 | Camera/Selfie interpolan `targetId` y el data URI crudos en el JS (GPS y QR usan `JsonSerializer`) | `CameraCommandHandler.cs:61-62` vs `QrCommandHandler.cs:47` |

---

## §14 Preguntas frecuentes

| Pregunta | Sección |
|---|---|
| ¿Cómo se agrega un comando nuevo al puente? (clase + una línea de DI; el orden importa) | §3.1, §2.3 |
| ¿Por qué `e.Cancel` no puede ir después de un `await`? | §3.2 |
| ¿Por qué `AllowConcurrentExecutions = true` en `Navigating`? | §3.2 |
| ¿Qué diferencia hay entre `Injection` y `Substitution` en GPS? | §5 |
| ¿Por qué la URL re-navegada lleva `_ts=N`? | §5 |
| ¿Qué pasa si el GPS falla en cada modo? | §5 |
| ¿Qué overlay se ve si hay dos activos a la vez? | §6.2 |
| ¿Por qué el overlay de Red oculta el `WebView` y los otros no? | §6.2 |
| ¿Qué significan los códigos `GPS-*`, `TEL-*`, `PRN-*`? | §6.2 |
| ¿De dónde sale el comprobante que se imprime? | §4.1, §8 |
| ¿Por qué el relay REST devuelve «El host no está permitido»? | §4.1, §13 T17 |
| ¿Por qué Android necesita los `.pem` embebidos y iOS no? | §9.1, §12 |
| ¿Por qué los tests linkean fuentes en vez de referenciar el proyecto? | §10 |
| ¿Qué compila y prueba la CI de este dominio? | §11 |
| ¿Por qué «Mostrar contenido» de la foto queda vacío? | §13 T5 |
| ¿Están al día los `.md` de `web-hibrida`? | §12 |
| ¿Por qué `Geo Pos` desde la home no muestra coordenadas? | §13 T14 |

---

## §15 Fuentes

| Fuente (ruta relativa al repo) | Uso en este índice |
|---|---|
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | §1 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Ejemplo_Maui_Hibrida.csproj` | §1.1, §1.2 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/MauiProgram.cs`, `AppShell.xaml.cs`, `App.xaml(.cs)` | §2.3 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Pages/MainPage.xaml(.cs)`, `ViewModels/MainViewModel.cs` | §2.1, §3.2, §13 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/UrlCommands/**` | §3, §4, §5, §13 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/CustomWebView/**` | §2.2, §7 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/**` (Common, GPS, Phone, Networks, MotorDSL, Camera, Images, QRLector, READMEs) | §6, §7, §13 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Platforms/Android/AndroidManifest.xml`, `Resources/xml/network_security_config.xml` | §9.1 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Platforms/iOS/Info.plist`, `Entitlements*.plist` | §9.2 |
| `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida.Tests/**` | §10, §13 T12 |
| `Ejemplos_Devices/Integrada/Ejemplo_ws_Blazor/Program.cs`, `Components/**`, `Controllers/**`, `DTOs/**`, `wwwroot/*.html`, `Properties/launchSettings.json` | §8, §13 |
| `Ejemplos_Devices/Integrada/Ejemplo_Docs_Integrada/Readme.md` (vacío) | §1 |
| `Ejemplos_Devices/Docs/web-hibrida/*.md` | §12 |
| `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml`, `Utilities/simular_ui.sh`, `Utilities/end2end/com.ejemplos.devices.integrada.hibrida.yaml` | §11 |
| Repo hermano `Ejemplos_Maui_Devices.Documentacion`: ADR-0001, 0003, 0008, 0009 (`Ejemplos_Maui_Devices-docs/docs/04-decisions/`), `Analisis/Plan-Armonizacion-Overlays.md` | contexto citado por comentarios (§3, §5, §9, §10) |
