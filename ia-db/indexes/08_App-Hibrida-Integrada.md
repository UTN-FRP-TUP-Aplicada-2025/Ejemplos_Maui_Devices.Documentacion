# 08 — App híbrida integrada

> **Propósito:** la app que integra todos los dispositivos en un `WebView`: el puente de comandos por URL, los cuatro overlays armonizados, el backend Blazor que la ejercita y la suite de tests.
> **Fuente primaria:** `Ejemplos_Devices/Integrada/` — 3 proyectos, ~7.000 líneas.
> **Índices relacionados:** [01_Camara](01_Camara.md) · [02_QR](02_QR.md) · [03_Impresion-Termica](03_Impresion-Termica.md) · [04_GPS](04_GPS.md) · [06_Telefonia](06_Telefonia.md) · [07_Red-Conectividad](07_Red-Conectividad.md) (los dominios que integra) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md).

---

## 1. Los tres proyectos

| Proyecto | Qué es | Tamaño |
|----------|--------|--------|
| `Ejemplo_Maui_Hibrida` | App MAUI: un `WebView` a pantalla completa + `LibApp/` con todos los dispositivos | ~3.700 líneas |
| `Ejemplo_ws_Blazor` | Backend Blazor Server + API que la app abre y que dispara los comandos | ~530 líneas de Razor + 3 controllers |
| `Ejemplo_Maui_Hibrida.Tests` | 78 tests xUnit sobre ViewModels y puente, **sin emulador** | ~1.440 líneas |

`ApplicationId` de la app: `com.ejemplos.devices.integrada.hibrida`. Es el proyecto con más paquetes del repositorio: `BarcodeScanning.Native.Maui` 3.0.4, `CommunityToolkit.Maui{,.Core,.Camera}` 14.2.0/6.1.0, `CommunityToolkit.Mvvm` 8.4.2, `MetadataExtractor` 2.9.0, `SkiaSharp` 3.119.2, `AdamE.Google.iOS.GoogleUtilities` 8.1.0.3, MAUI 10.0.80 y los **7 paquetes `MotorDsl.*` 1.0.13**.

## 2. La idea: la web pide, el nativo ejecuta

La app no tiene UI propia más allá del `WebView` y cuatro botones: **la pantalla la pone la web**. Cuando la página necesita una capacidad del dispositivo, **navega a una URL con un marcador**; la app intercepta la navegación, la cancela, ejecuta el comando nativo y devuelve el resultado.

```
Panel.razor  ──NavigateTo("/panel?photo=photo&param=imgFoto1", forceLoad:true)──▶  WebView
                                                                                     │ Navigating
                                                            MainViewModel.Navigating ▼
                                          Plan(url)  ──▶ e.Cancel = true  (fase SÍNCRONA)
                                          ExecuteAsync(plan, url)          (fase asíncrona)
                                                     ▼
                                          CameraCommandHandler → cámara → base64
                                                     ▼
                                          IWebViewBridge.RunScript(js)  ──▶ DOM de la página viva
```

## 3. El protocolo de URL

Cada comando es un par `clave=valor` en la query. `param={id}` nombra el elemento del DOM donde inyectar el resultado.

| Marcador | Handler | Entrega | Ejemplo de la web |
|----------|---------|---------|-------------------|
| `coordenadas=coordenadas` | `GpsCommandHandler` | `Injection` **con** `param`, `Substitution` **sin** `param` | `/geolocalizacion?coordenadas=coordenadas` · `/panel?coordenadas=coordenadas&param=contenidoCoordenada` |
| `phone=phone` | `CallCommandHandler` | `None` | `/panel?phone=phone` |
| `photo=photo&param={id}` | `CameraCommandHandler` | `Injection` | `/panel?photo=photo&param=imgFoto1` |
| `selfie=selfie&param={id}` | `SelfieCommandHandler` | `Injection` | `/panel?selfie=selfie&param=imgSelfie1` |
| `qr=qr&param={id}` | `QrCommandHandler` | `Injection` | `/panel?qr=qr&param=contenidoQR` |
| `sendAPI=sendAPI&httpMethod=…&url={enc}&param={callId}&body={enc}` | `SendApiCommandHandler` | `Injection` (hook `window.recibirRespuestaApi`) | `/panel?sendAPI=…` |
| `action=print` | `PrintCommandHandler` | `None` | `/panel?action=print` |

**El orden de registro en `MauiProgram.cs:116-122` es el orden de precedencia** (política *first-match-wins*): GPS → Call → Camera → Selfie → QR → SendApi → Print. Por eso el combo «Llamar y reportar» de la web (`phone` + `sendAPI` en la misma URL) **solo dispara la llamada**: `phone` se evalúa antes. La propia web lo documenta en el comentario de `Panel.razor:210-212`.

## 4. El puente de comandos (`LibApp/UrlCommands/`)

### 4.1 El contrato

`IUrlCommandHandler` (44 l.) — «agregar un comando nuevo = una clase que implemente esto + una línea de DI»:

| Miembro | Default | Para qué |
|---------|---------|----------|
| `bool CanHandle(string url)` | — | ¿Esta URL es mía? |
| `bool CancelsNavigation` | `true` | ¿Esta URL-comando cancela la navegación? |
| `CommandDelivery DeliveryFor(string url)` | `None` | Cómo devuelve el resultado **esta invocación** |
| `void OnMatchedSync(string url)` | no-op | Gancho síncrono, antes de cualquier `await` |
| `Task<BridgeOutcome> HandleAsync(string url)` | — | Ejecutar |

Los tres del medio son **default interface members**: se agregaron sin editar ninguno de los 7 handlers existentes, que conservan el comportamiento anterior (cancelar siempre, sin entrega declarada).

`BridgeOutcome(bool CancelNavigation, string? NavigateTo = null)` es el resultado de interpretar la URL.

### 4.2 `CommandDelivery` — los tres modos de devolver un resultado

| Modo | Mecanismo | Quiénes |
|------|-----------|---------|
| `None` | No hay resultado para la web: la respuesta es la UI nativa (overlay) | llamada, impresión |
| `Injection` | Se inyecta en el DOM de la página viva vía `IWebViewBridge.RunScript`. **Requiere navegación cancelada**: si la página recarga, el elemento destino desaparece | foto, selfie, QR, sendAPI, GPS con `param` |
| `Substitution` | Se **re-navega** la misma URL con el query param de comando sustituido por query params de valor (patrón heredado de APPGDA) | GPS sin `param` |

Es propiedad del **comando concreto, no del handler**: el mismo handler puede operar en dos modos según la URL — por eso se consulta con `DeliveryFor(url)` y no con una propiedad sin argumentos.

> **INVARIANTE:** un comando que declara `Substitution` **debe** devolver `NavigateTo` no nulo, pase lo que pase con el dispositivo. Si no, la navegación queda muerta: se canceló y no se re-navegó.

### 4.3 `UrlPlan` y el dispatcher

`UrlPlan(IReadOnlyList<IUrlCommandHandler> Matches, bool Cancel)` es el resultado de **clasificar** una URL, calculado **una sola vez y de forma síncrona**.

Por qué existe (comentario de `UrlPlan.cs:5-10`): antes `CanHandle` se evaluaba **dos veces por navegación** —una en `IsCommand` y otra en `DispatchAsync`— y era inocuo porque los 7 handlers eran funciones puras de la URL. Con un handler que consulte o mute estado (un token persistido, p. ej.) deja de serlo: las dos lecturas darían distinto y el comportamiento dependería del orden.

`UrlCommandDispatcher` (99 l.) separa clasificación de ejecución:

| Método | Qué hace |
|--------|----------|
| `Plan(url)` | **100 % síncrono a propósito**: el WebView lee `e.Cancel` apenas retorna el handler de `Navigating`. Evalúa `CanHandle` una vez por handler, hace el **OR de `CancelsNavigation`** sobre los que matchean y corre `OnMatchedSync` para **todos** ellos (así el gancho no depende de *first-match-wins*) |
| `ExecuteAsync(plan, url)` | Ejecuta `plan.Primary` (el primero que matcheó) |
| `DispatchAsync(url)` | Conveniencia para los botones nativos, que no vienen de un `Navigating` |
| `IsCommand(url)` | Se conserva por compatibilidad de firma; `MainViewModel` ya no la usa |

Dos aserciones `#if DEBUG` protegen el invariante de continuación, **fallando ruidoso en vez de manifestarse como un WebView colgado**:
1. En `Plan`: si se canceló pero el handler que va a ejecutarse (first-match-wins) **no** es de los que pidieron cancelar → `Debug.Fail("… Navegación muerta")`.
2. En `ExecuteAsync`: si el handler declara `Substitution` y devolvió `NavigateTo == null` → `Debug.Fail`.

⚠️ Cuando **ningún** handler matchea, `ExecuteAsync` devuelve `NavigateTo = null` deliberadamente. Devolver la propia `url` haría que `MainViewModel` reasigne `Url = e.Url` en **toda** navegación normal (incluido el reload del pull-to-refresh), lo que reasigna `WebView.Source` y dispara una segunda navegación superpuesta: `Navigated` nunca cierra el `RefreshView` y `IsRefreshing` no vuelve a `false` limpiamente (`UrlCommandDispatcher.cs:66-72`).

### 4.4 El caso GPS: un handler, dos modos

`GpsCommandHandler` (107 l.) es el ejemplo que motivó `CommandDelivery`:

- **Con `param`** → `Injection`: arma `"Latitud: …, Longitud: …"`, lo inyecta con `element.textContent` y **no recarga** — así el overlay de espera queda visible sobre la página estática, sin parpadeo ni el problema de «URL idéntica no re-navega». Sin coordenada no inyecta nada: la continuación es el overlay de error, y la página sigue viva porque se canceló la navegación.
- **Sin `param`** → `Substitution`: re-navega **siempre**, haya o no señal. Sin coordenada sustituye por el **centinela `0.0/0.0`**, que la web ya interpreta como «sin coordenada» (`GeoLocalizacion.razor` lo pone si llegan vacías). Antes del rediseño, un fallo del dispositivo devolvía `(cancel: true, navigateTo: null)` = **página congelada**.

Dos detalles que el código justifica:
- **`CultureInfo.InvariantCulture`** al formatear lat/lng: con la cultura del dispositivo (`es-*` → coma) la coordenada llegaría sin separador decimal (`-31,74` → `-3174`).
- **Nonce `_ts` monótono** (`Interlocked.Increment`) al final de la URL re-navegada: si la coordenada es idéntica a la anterior, `WebView.Source` no dispara `PropertyChanged` y la página no re-navega. Blazor ignora el parámetro extra.

El resto de los handlers también serializa con `JsonSerializer` lo que va al script, «para no romper el JS / evitar inyección» — salvo `CameraCommandHandler`, que interpola el `targetId` y el data-URI directamente en el string de JS (`:61-62`).

## 5. El bridge del WebView

Tres piezas que mantienen al VM y a los handlers **desacoplados del control**:

| Pieza | Rol |
|-------|-----|
| `IWebViewBridge` | Única abstracción de las acciones imperativas: `Reload()`, `RunScript(js)` y dos eventos |
| `WebViewBridge` (singleton) | Solo dispara los eventos; no conoce ningún `WebView` |
| `WebViewBridgeBehavior : Behavior<WebView>` | Traduce los eventos en acciones sobre el control, **siempre** en el UI thread y fire-and-forget |

⚠️ Trampa documentada en la behavior (`:22-26`): **una `Behavior` no está en el árbol visual, así que su `BindingContext` no se hereda solo**. Hay que propagárselo desde el control (`BindingContext = bindable.BindingContext` + suscripción a `BindingContextChanged`) o el binding de `Bridge` **queda en `null` sin ningún error visible**.

## 6. `MainViewModel` — el interceptor de navegación

150 líneas. Expone `Url`, `IsRefreshing`, los cuatro overlays, `WebBridge` y los comandos.

### 6.1 Las dos fases de `Navigating`

```csharp
[RelayCommand(AllowConcurrentExecutions = true)]
private async Task Navigating(WebNavigatingEventArgs e)
{
    // ── Fase SÍNCRONA: todo lo que decide e.Cancel va acá arriba ──
    var plan = _dispatcher.Plan(e.Url);
    if (plan.Cancel) e.Cancel = true;
    if (!plan.HasMatches) { IsRefreshing = false; return; }
    if (plan.Cancel && comandoEnCurso) { IsRefreshing = false; return; }
    // ── Fase ASÍNCRONA ──
    …await _dispatcher.ExecuteAsync(plan, e.Url)…
}
```

Dos reglas que el código deja escritas:

1. **Regla de mantenimiento:** «no introducir ningún `await` por encima de la asignación de `e.Cancel`». El cuerpo de un método `async` corre sincrónicamente hasta el primer `await`; después de él, el WebView ya leyó `e.Cancel`.
2. **`AllowConcurrentExecutions = true` es imprescindible.** Con el default de `AsyncRelayCommand`, el comando quedaría bloqueado por «estar en ejecución» y el `EventToCommandBehavior` **no lo invocaría en el 2.º `Navigating`**: `e.Cancel` no se fijaría y el WebView haría la navegación real, perdiendo la inyección del resultado.

El **guard de reentrada** (`comandoEnCurso`) descarta un segundo click mientras hay un comando en curso, pero **solo para planes que cancelan**: un plan no-cancelable deja seguir la navegación igual, así que bloquearlo no protegería nada y sí podría descartar trabajo.

### 6.2 Los demás comandos

| Comando | Qué hace |
|---------|----------|
| `TakeGPSCommand` («Geo Pos») | Fuerza `coordenadas=coordenadas` sobre la `Url` actual y delega en el dispatcher, sin duplicar la lógica de reescritura |
| `TakePhoneCommand` («Llamar») | `DispatchAsync("phone=phone")` — usa el protocolo real de la web |
| `TakeQRCommand` («Leer QR») | `DispatchAsync("qr=qr&param=contenidoQR")` |
| `RefreshCommand` | `WebBridge.Reload()`; el spinner se cierra en `Navigated` |
| `VolverCommand` | `Url = "https://aplicada.somee.com"` |
| `NavigatedCommand` | `IsRefreshing = false` y notifica al overlay de Red el éxito o el fallo de la navegación |

## 7. Los cuatro overlays

`MainPage.xaml` apila cuatro `StatusOverlayView` sobre el `WebView`. **El orden de declaración es la prioridad visual** (el último queda arriba): GPS → Red → Llamada → **Impresión** (máxima prioridad).

El `WebView` se ve **solo cuando el overlay de Red está oculto** (`IsVisible` con `InvertedBoolConverter`): así nunca se muestra la página de error del navegador.

Los cuatro dominios comparten la misma estructura de cinco piezas:

| Dominio | Servicio | Resultado tipado | Catálogo (prefijo) | ViewModel |
|---------|----------|------------------|--------------------|-----------|
| GPS | `IGpsService` / `GpsService` | `GpsResult` | `GpsErrorCatalog` (**`GPS-*`**) | `GpsOverlayViewModel` |
| Teléfono | `ICallService` / `CallService` | `CallResult`, `CallPermissionResult`, `CallMode` | `CallErrorCatalog` (**`TEL-*`**) | `CallOverlayViewModel` |
| Red | `INetworkService` / `NetworkService` | `NetworkResult` | — | `NetworkOverlayViewModel` |
| Impresión | `IPrinterService` / `PrinterService` | `PrintResult`, `DiscoverResult`, `DocumentResult` | `PrinterErrorCatalog` (**`PRN-*`**) | `PrinterOverlayViewModel` |

Códigos de GPS: `GPS-PERM-ASK`, `GPS-PERM-DENIED`, `GPS-PERM-MDM`, `GPS-OFF`, `GPS-UNSUPPORTED`, `GPS-NO-SIGNAL`, `GPS-CANCELLED`, `GPS-UNKNOWN`. De teléfono: `TEL-PERM-ASK`, `TEL-PERM-DENIED`, `TEL-PERM-MDM`, `TEL-UNSUPPORTED`, `TEL-BAD-NUMBER`, `TEL-CANCELLED`, `TEL-NO-ACTIVITY`, `TEL-UNKNOWN`. Los de impresión, en el [índice 03 §6](03_Impresion-Termica.md).

> **Por qué `GpsFailure`, `CallFailure` y `PrintFailure` son tres records con la misma forma** (`GpsFailure.cs:6-12`): se evaluó unificarlos en un tipo de `Common/` y **se descartó** — obligaba a reabrir el dominio de impresión, el único ya validado en dispositivo, y a reprobarlo entero, además de desincronizar la app del PoC `Ejemplo_MotorDSL_Dialog`, que no tiene carpeta `Common`. «Lo que armoniza el patrón son los invariantes, no un tipo compartido.»

### 7.1 `IUiDispatcher`

`LibApp/Devices/Common/Services/IUiDispatcher.cs` — abstracción de `MainThread.BeginInvokeOnMainThread`. Existe «porque `MainThread` es un estático de plataforma: un ViewModel que lo invoque directo **no se puede ejercitar fuera de un dispositivo**, aunque no tenga una sola directiva `#if`». Lo necesita **solo el overlay de Red**, que es el único reactivo; los demás se disparan desde el hilo de UI.

### 7.2 `NetworkService` — la sonda de internet real

`IsOnline` mira `NetworkAccess`, pero `CheckUrlAsync` hace una **sonda activa** a `http://www.msftconnecttest.com/connecttest.txt` (10 s de tope) y valida que el cuerpo contenga `"Microsoft Connect Test"`. Si hay enlace pero el cuerpo no coincide, es un **portal cautivo u operadora sin crédito que redirige con 200 OK** → `Offline`.

Detalle de UX documentado (`:37-47`): la petición **siempre** va a la URL de la sonda, pero lo que se **reporta** es el host de la URL que el usuario quiso abrir. Antes se reportaba el de la sonda y el mensaje de DNS nombraba `www.msftconnecttest.com`, un dominio que el usuario nunca visitó.

`NetworkResult`: `Online` · `Offline` · `Timeout(Url)` · `DnsFailure(Host)` · `HttpFailure(StatusCode, Url)` · `RequestFailure(Message)`.

### 7.3 `PrintCommandHandler` — el caso con documento remoto

A diferencia del PoC `Ejemplo_MotorDSL_Dialog` ([índice 03](03_Impresion-Termica.md)), acá el documento **viene por red**: `GET https://aplicada.somee.com/api/Tikects/comprobante` (30 s de tope) → validación de contrato → render con `IDocumentEngine` → impresión. Por eso `DocumentResult` y los códigos `PRN-DOC-NET` / `PRN-DOC-CONTRACT` **sí son alcanzables** en la app. El handler se pasa a sí mismo como reintento del overlay, de modo que «Reintentar» ante un fallo de red **vuelva a la red**.

Perfiles registrados en la app (tres, contra uno del PoC): `("thermal_58mm", 32, "escpos-bitmap")`, `("a4-pdf", 80, "pdf")`, `("pdf", 48, "pdf")`.

## 8. `LibApp/` — el paquete de dispositivos

Todo el código reutilizable cuelga de `LibApp/` con namespace raíz **`LibApp.*`** (no `Ejemplo_Maui_Hibrida.LibApp.*`), alineado con el link por comodín del `.csproj` de tests.

```
LibApp/
├── CustomWebView/Behaviors/     IWebViewBridge, WebViewBridge, WebViewBridgeBehavior
│              /Converts/        WebNavigating/NavigatedEventArgsConverter
├── Devices/
│   ├── Camera/Pages/            MyMediaPickerPage, MyMediaSelfiePickerPage  → índice 01
│   ├── Common/                  StatusOverlayView(+Model), IUiDispatcher
│   ├── GPS/                     Models, Services, ViewModels, ApiRelayService → índice 04
│   ├── Images/                  IImageService, ImageDeviceAutoRotateService, SelfieMaskDrawable
│   ├── MotorDSL/                DTOs/Print, Models, Pages, Services, ViewModels → índice 03
│   ├── Networks/                Models, Services, ViewModels                   → índice 07
│   ├── Phone/                   Models, Services, ViewModels                   → índice 06
│   └── QRLector/                QRContent, QRLectorPage                        → índice 02
└── UrlCommands/                 contrato, enum, plan, dispatcher, Handlers/
```

`ApiRelayService` vive en `Devices/GPS/` aunque lo usa el comando `sendAPI` (relay REST genérico), no el GPS.

El QR de la híbrida usa **`BarcodeScanning.Native.Maui`** — la recomendación 🥇 del estudio de NuGets ([índice 02 §2](02_QR.md)) — con `.UseBarcodeScanning()` en el arranque.

## 9. La suite de tests

`Ejemplo_Maui_Hibrida.Tests` — **67 `[Fact]` + 11 `[Theory]`** repartidos en 9 archivos.

**El TFM es `net10.0` plano, no `net10.0-android`**: la suite corre en el runner de escritorio y en CI, sin emulador ni dispositivo. Es viable «porque los ViewModels no tienen una sola directiva `#if`: todo el código de plataforma vive en los servicios, detrás de `I*Service`».

**Linkeo de fuentes, no `ProjectReference`**: un proyecto `net10.0` no puede referenciar uno `net10.0-android`, y extraer `LibApp/` a una librería multi-target contradiría la decisión de que cada ejemplo sea autocontenido y copiable. Se linkea solo lo *platform-free*:

| Entra | Queda afuera |
|-------|--------------|
| `StatusOverlayViewModel`, los cuatro `*OverlayViewModel` | `PrinterService`, `GpsService`, `CallService`, `NetworkService` (tienen `#if ANDROID` y estáticos de MAUI: `Preferences`, `Permissions`, `AppInfo`) |
| Todos los `Models/` (resultados tipados y catálogos) | El resto de `Handlers/` (tocan `Shell`/`Application`) |
| Solo las **interfaces** de servicio (los tests las implementan con fakes) | |
| `LibApp/UrlCommands/*.cs` **por comodín** (todo el paquete raíz es platform-free) | |
| `GpsCommandHandler` explícito, para cubrir el round-trip de cultura de las coordenadas | |
| `MainViewModel`, sujeto de prueba del guard de reentrada | |

### 9.1 Los cinco invariantes ejecutables

`Invariantes.cs` (81 l.) — «cada aserción de acá nació de un defecto real encontrado en el código»:

| # | Invariante | El defecto que lo originó |
|---|-----------|---------------------------|
| **I-1** | Toda variante no-`Success` produce **exactamente una pantalla**, con título, mensaje y al menos una salida | Un `case` que solo hacía `Hide()` dejaba al usuario sin respuesta: tocar «Imprimir» y que no pasara nada |
| **I-2** | Toda variante del resultado tipado tiene su pantalla — `VariantesDe<T>()` las descubre por reflexión sobre los `sealed record` anidados | «El defecto más caro del patrón: modelar una variante y no producirla nunca». C# no lo verifica: un `switch` sin un caso compila igual |
| **I-3** | Ningún mensaje crudo del sistema llega al usuario | El usuario leía literalmente `Print failed after 1 attempt(s): paper out` |
| **I-4** | Toda pantalla de error tiene **exactamente un** botón primario | `Primary` es «el `DataTrigger` de `Secondary` no disparó»: omitirlo no da error de compilación, simplemente ninguna acción destaca |
| **I-5** | *(ver `Analisis/Plan-Armonizacion-Overlays.md` §2 en el repo de documentación)* | |

Archivos de test: `BaseOverlayTests` (7), `PrinterOverlayTests` (19), `PrinterErrorCatalogTests` (11), `GpsOverlayTests` (11), `NetworkOverlayTests` (9), `CallOverlayTests` (7), `GpsCommandHandlerTests` (6), `UrlCommandDispatcherTests` (5), `NavigatingReentrancyTests` (3).

## 10. `Ejemplo_ws_Blazor` — el backend que la ejercita

Blazor Server (`AddInteractiveServerComponents`) + controllers + OpenAPI con UI de **Scalar** en `/scalar`. Paquetes: `Microsoft.AspNetCore.OpenApi` 10.0.8, `Scalar.AspNetCore` 2.14.14.

| Ruta | Página | Rol |
|------|--------|-----|
| `/` y `/datos` | `Datos.razor` | Entrada |
| `/panel` | `Panel.razor` (322 l.) | **El banco de pruebas**: una tarjeta por comando |
| `/geolocalizacion` | `GeoLocalizacion.razor` | Destino del modo `Substitution` del GPS; si llegan vacías usa el centinela `0.0/0.0` |
| `/redirigir` | `Redirigir.razor` | Redirección a sitio externo (en iOS se espera que abra Safari) |
| `/not-found`, `/Error` | | `UseStatusCodePagesWithReExecute("/not-found")` |

| Controller | Endpoint | Para qué |
|------------|----------|----------|
| `GeoReporterController` | `POST /api/GeoReporter/track` | Recibe un `LocationDto`; es el destino del comando `sendAPI` |
| `TikectsController` | `GET /api/Tikects/comprobante` | Devuelve un `PrintDocument` hardcodeado — el mismo formato que consume MotorDsl. Espeja el patrón real de `PrintController.PrintActaById` de GDA.Core.API, pero sin base de datos |
| `PagoFakeController` | `POST /api/pagofake/pago`, `GET /api/pagofake/pago-form` | Reproduce dos formas de pago externo: 302 cross-host y HTML que auto-envía un form POST a otro host |

Los DTOs de impresión (`DTOs/Print/`: `PrintDocument`, `PrintNode`, `PrintStyle`, `N`) están **duplicados a propósito** en la app (`LibApp/Devices/MotorDSL/DTOs/Print/`) y en el backend: son el contrato entre ambos.

`Program.cs` configura `ForwardedHeaders` **vaciando `KnownNetworks`/`KnownProxies`** para que se acepten las cabeceras detrás del proxy de somee, con el tradeoff anotado en el propio archivo («solo hacelo si el borde de somee es la única vía de entrada»). El comentario de `:16-20` explica por qué `UseHttpsRedirection` importa: la lógica sensible al esquema puede derivar `ws://` en vez de `wss://`, y en iOS **ATS bloquea** un `ws://` desde una página `https`, mientras que Android con `usesCleartextTraffic` lo deja pasar.

## 11. Observaciones

- ⚠️ **Comentario XML que describe el modo contrario.** En `Panel.razor`, el comentario sobre `OnSolicitarCoordenadas` dice «la app … **INYECTA** el resultado en `#contenidoCoordenada` … Mismo patrón que foto/selfie/QR: `param={id}`», pero ese método es el del modo **`Substitution`** (navega a `/geolocalizacion?coordenadas=coordenadas`, sin `param`, y no inyecta). El texto quedó del camino anterior y se copió tal cual al método nuevo. Vale para `OnSolicitarGeoposicion`, no para `OnSolicitarCoordenadas`. **Guiarse por la URL, no por el comentario.**
- Las dos tarjetas de GPS del panel están rotuladas **«Tomar Coordenadas» las dos**: se distinguen por el `<h4>` («Solicitar coordenadas» vs. «Solicitar GeoPosicion») y por el `<p>`, no por el botón. El `<p>` de la segunda tampoco describe su modo: dice «en una página nueva», que es lo que hace la primera.
- `QrCommandHandler` conserva **comentado** el camino por `Shell.Current.GoToAsync` + `TaskCompletionSource` y usa `Application.Current.Windows[0].Page.Navigation.PushAsync`, distinto de cámara y selfie, que sí navegan por Shell.
- `CallCommandHandler` llama a un **número hardcodeado** (`NumeroPorDefecto = "3434807427"`): el comando `phone=phone` no toma el número de la URL.
- `MainViewModel.Volver` y `PrintCommandHandler` apuntan a `https://aplicada.somee.com`: es el despliegue del backend.
- Hay referencias a **ADR-0001 y ADR-0009** y a `Analisis/Plan-Armonizacion-Overlays.md` repartidas por el código; esos documentos viven en el repositorio de documentación ([índice 10](10_Documentacion-Transversal.md)), no en el del código.

## 12. Fuentes

| Ruta | Contenido |
|------|-----------|
| `Integrada/Ejemplo_Maui_Hibrida/LibApp/UrlCommands/` | Contrato, `CommandDelivery`, `UrlPlan`, dispatcher y los 7 handlers |
| `Integrada/Ejemplo_Maui_Hibrida/ViewModels/MainViewModel.cs` | Las dos fases de `Navigating`, guard de reentrada, comandos nativos |
| `Integrada/Ejemplo_Maui_Hibrida/Pages/MainPage.xaml` | `WebView`, `RefreshView`, apilado de los cuatro overlays |
| `Integrada/Ejemplo_Maui_Hibrida/MauiProgram.cs` | Orden de registro de handlers = precedencia; perfiles de impresión; DI completa |
| `Integrada/Ejemplo_Maui_Hibrida/LibApp/CustomWebView/` | El bridge y su behavior |
| `Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/` | Los cinco dominios de dispositivo con servicio, resultado, catálogo y overlay |
| `Integrada/Ejemplo_Maui_Hibrida.Tests/` | Los invariantes ejecutables, los fakes y el linkeo de fuentes del `.csproj` |
| `Integrada/Ejemplo_ws_Blazor/Components/Pages/Panel.razor` | El banco de pruebas: una tarjeta por comando |
| `Integrada/Ejemplo_ws_Blazor/Controllers/` | Los tres endpoints que la app consume |
