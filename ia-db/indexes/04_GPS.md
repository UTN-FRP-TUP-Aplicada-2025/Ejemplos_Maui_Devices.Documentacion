# 04 — GPS: geolocalización, permisos y geocodificación inversa

> **Propósito**: destilar el ejemplo autónomo `Ejemplo_Maui_GPS` (lectura de posición, permisos de ubicación, overlay de estado, geocodificación inversa con Google, apertura del mapa en el navegador), el manejo de la API key que impide compilarlo recién clonado y los dos modos con que la app híbrida entrega la coordenada a la web.
> **Fuente primaria**: `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/**`, `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/{Readme,secret,servicio}.md`, `.gitignore:420-421`; para la comparación, `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/GPS/**` y `LibApp/UrlCommands/Handlers/GpsCommandHandler.cs` (commit `285f6fb`).
> **Índices relacionados**: [00_MASTER-INDEX](00_MASTER-INDEX.md) · [05_Mapas](05_Mapas.md) (mapa embebido) · [06_Telefonia](06_Telefonia.md) (mismo patrón coordinador + resultado tipado + overlay en `Ejemplo_Maui_DirectCall`) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (puente de comandos por URL, overlay híbrido, tests) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) (workflow iOS, `.bat`, Maestro) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md) (README, CHANGELOG, documentos del repo de documentación).

Todas las rutas son relativas a la raíz del repo de código `Ejemplos_Maui_Devices`. Abreviatura: `GPS/` = `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/`.

---

## 1. Inventario del dominio

```
Ejemplos_Devices/GPS/
├── Ejemplo_Docs_GPS/
│   ├── Readme.md          ← sólo un link a la doc oficial de MAUI Geolocation (net-maui-10.0)
│   ├── secret.md          ← 5 estrategias de manejo de API keys; la cátedra elige la 3 (template)
│   └── servicio.md        ← 5 alternativas para que un servicio dispare el GPS + overlay
└── Ejemplo_Maui_GPS/
    ├── Ejemplo_Maui_GPS.csproj
    ├── MauiProgram.cs     ← DI (§3)
    ├── App.xaml(.cs) · AppShell.xaml(.cs)   ← Shell con una sola ruta: MainPage
    ├── Controls/GpsStatusOverlayView.xaml(.cs)   ← ContentView del overlay (§6)
    ├── Pages/MainPage.xaml(.cs)
    ├── Services/
    │   ├── ApiKeys.cs.template   ← versionado; ApiKeys.cs NO (ignorado)
    │   ├── GoogleMapService.cs   ← geocodificación inversa (§7)
    │   ├── GpsCoordinator.cs     ← singleton dueño del overlay y la cancelación (§5)
    │   ├── GpsResult.cs          ← record con 8 variantes (§4)
    │   ├── GpsService.cs         ← permiso + lectura → GpsResult (§4)
    │   ├── LocationPermissionResult.cs · LocationPermissionService.cs   ← permisos (§4)
    │   └── TestService.cs        ← envoltorio de prueba, registrado pero sin consumidor
    ├── ViewModels/ AsyncRelayCommand.cs (+RelayCommand) · GpsOverlayViewModel.cs · MainPageViewModel.cs · ViewModelBase.cs
    ├── Platforms/Android/AndroidManifest.xml · Platforms/iOS/Info.plist · Entitlements*.plist   (§8)
    └── Properties/launchSettings.json
```

Fuera de la carpeta, pero del dominio: `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml`, `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat`, `Utilities/end2end/com.ejemplos.devices.gps.yaml` (detalle en [09](09_CI-CD-y-Build.md); ver §10 y §11).

## 2. Ficha del proyecto

| Atributo | Valor | Fuente |
|---|---|---|
| TFM | `net10.0-android`; `net10.0-ios` sólo si se compila en macOS | `GPS/Ejemplo_Maui_GPS.csproj:4-5` |
| ApplicationId / título | `com.ejemplos.devices.gps` / `GPS` | `csproj:16,18` |
| Versión | `ApplicationDisplayVersion 1.0`, `ApplicationVersion 1` | `csproj:20-21` |
| Mínimos | iOS 15.0 · Android API 25 | `csproj:25-26`; `AndroidManifest.xml:34` (target 36) |
| Android | ABI arm/arm64/x86/x64, páginas de 16 KB (`max-page-size=16384`), APK (no AAB) | `csproj:33-50` |
| iOS Release | intérprete + trimming `SdkOnly`, AOT sólo en `ios-arm64` | `csproj:54-64` |
| Paquetes | `Microsoft.Maui.Controls` (`$(MauiVersion)`), `Microsoft.Extensions.Logging.Debug 10.0.2` | `csproj:81-84` |
| Fuentes | OpenSans ×2 + `MaterialIconsOutlined` (ícono `location_off` del overlay) | `MauiProgram.cs:15-20` |
| MVVM | manual, sin CommunityToolkit: `ViewModelBase.SetProperty`, `AsyncRelayCommand`, `RelayCommand` | `ViewModels/ViewModelBase.cs:8-24`, `ViewModels/AsyncRelayCommand.cs:9-76` |

Geolocalización, permisos, `Browser` y `AppInfo` son MAUI Essentials: no hay NuGet extra de GPS.

## 3. Composición (DI) y capas

`MauiProgram.AddServices` (`GPS/MauiProgram.cs:29-45`):

| Registro | Vida | Línea |
|---|---|---|
| `LocationPermissionService`, `GpsService`, `GpsCoordinator`, `TestService`, `GoogleMapService` | Singleton | 32-36 |
| `ViewModels.MainPageViewModel` | Transient | 39 |
| `Pages.MainPage` | Transient | 42 |

```
LocationPermissionService ──► LocationPermissionResult (enum de 4)
        │
GpsService ─────────────────► GpsResult (8 variantes)            ← sin UI
        │
GpsCoordinator (singleton) ─► posee GpsOverlayViewModel + CancellationTokenSource
        │                     Aplicar(result) decide sólo el overlay
        ├── MainPageViewModel ─► traduce GpsResult a texto (Coordenadas)
        └── MainPage.xaml.cs  ─► botón «Prueba»: llama al coordinador sin pasar por el VM
GoogleMapService (independiente) ◄── MainPageViewModel (Obtener Domicilio)
```

`MainPage` recibe por constructor el VM **y** el coordinador (`Pages/MainPage.xaml.cs:11-17`); `App.CreateWindow` instancia `new AppShell()` y el Shell resuelve la página por `DataTemplate` (`App.xaml.cs:10-13`, `AppShell.xaml:10`).

## 4. Obtención de la ubicación y permisos

### 4.1 Permisos: `LocationPermissionService.RequestAsync` (`Services/LocationPermissionService.cs:13-41`)

```
CheckStatusAsync<LocationWhenInUse>
 ├─ Granted ─────────────────────────────► Granted
 └─ otro → RequestAsync<LocationWhenInUse>        (iOS: diálogo sólo la 1.ª vez)
      ├─ Granted ────────────────────────► Granted
      ├─ Restricted ─────────────────────► Restricted        (iOS: MDM/control parental)
      └─ otro → #if ANDROID ShouldShowRationale
             ├─ true  ──────────────────► DeniedCanRetry     (se puede volver a pedir)
             └─ false / iOS ────────────► Denied             (sólo por ajustes)
```

- Sólo se pide **`LocationWhenInUse`** (primer plano); el background está comentado en ambos manifests (§8).
- `OpenAppSettings()` = `AppInfo.ShowSettingsUI()` (`:43`): abre los ajustes **de la app**, no los de ubicación del SO.
- `CheckAsync()` + `Map()` (`:7-11`, `:45-50`) no tienen llamadores en el proyecto. (En la híbrida se eliminaron; ver `CHANGELOG.md:214`.)

### 4.2 Lectura: `GpsService.ObtenerUbicacionAsync` (`Services/GpsService.cs:15-69`)

| Paso | Código | Línea |
|---|---|---|
| Permiso → `GpsResult` de rechazo | `Restricted`→`PermissionRestricted`; `DeniedCanRetry`→`PermissionDenied(true)`; `Denied`/default→`PermissionDenied(false)` | 18-30 |
| Vía rápida | `Geolocation.GetLastKnownLocationAsync()`; se acepta si `Timestamp` < 1 min | 43-44 |
| Fix nuevo | `GeolocationRequest(GeolocationAccuracy.Medium, 10 s)` + `GetLocationAsync(req, ct)` | 46-47 |
| Sin dato | `location is null` → `NoSignal` | 50 |
| Excepciones → variantes | `OperationCanceledException`→`Cancelled`, `FeatureNotEnabledException`→`GpsDisabled`, `FeatureNotSupportedException`→`NotSupported`, resto→`Failure(ex.Message)` | 52-67 |

⚠️ La variante «precisa» (`GeolocationAccuracy.Best` con `DefaultTimeout` de **15 s**) está comentada (`:36-41`); `DefaultTimeout` (`:8`) sólo se usa ahí. El timeout efectivo es **10 s** y la precisión **Medium**.

⚠️ La resolución de permisos (`:18`) está **fuera** del `try` (`:34`): una excepción de `Permissions.*` (p. ej. iOS sin la clave de `Info.plist`) no se convierte en `GpsResult`; sube al coordinador, que oculta el overlay y la relanza (`GpsCoordinator.cs:41-45`).

### 4.3 Resultado tipado: `GpsResult` (`Services/GpsResult.cs:5-15`)

`Success(Location)` · `PermissionDenied(bool CanRetry)` · `PermissionRestricted` · `GpsDisabled` · `NotSupported` · `NoSignal` · `Cancelled` · `Failure(string Message)`. Es el patrón transversal «servicio → resultado tipado → overlay» del repo (ver [00_MASTER-INDEX](00_MASTER-INDEX.md)).

## 5. `GpsCoordinator`: punto único de captura y cancelación

`Services/GpsCoordinator.cs:10-71`. Comentario de cabecera (`:5-9`): cualquier caller hace `await _coord.CapturarAsync()` y el overlay aparece/desaparece solo.

```mermaid
sequenceDiagram
    participant C as Caller (VM / code-behind)
    participant K as GpsCoordinator
    participant O as GpsOverlayViewModel
    participant S as GpsService
    C->>K: CapturarAsync(ct)
    K->>K: cancela y descarta _cts previo; crea nuevo (o linked si ct ≠ default)
    K->>O: ShowBusy() en MainThread
    K->>S: ObtenerUbicacionAsync(_cts.Token)
    S-->>K: GpsResult
    K->>O: Aplicar(result) en MainThread
    K-->>C: GpsResult (el texto lo decide el caller)
```

| `GpsResult` | Efecto en overlay (`Aplicar`, `:56-70`) |
|---|---|
| `PermissionDenied d` | `ShowPermissionDenied(d.CanRetry)` |
| `PermissionRestricted` | `ShowRestricted()` |
| todo lo demás (incluso `GpsDisabled`, `NoSignal`, `Failure`) | `Hide()` — el error queda sólo en el texto del VM |

- Callbacks del overlay atados al coordinador, no a una página (`:23-25`): reintentar = `CapturarAsync()`, abrir ajustes = `_permissions.OpenAppSettings()`.
- `Cancelar()` = `_cts?.Cancel()` (`:53`). Una nueva captura cancela la anterior (`:30-32`).
- `AsyncRelayCommand.Execute` siempre pasa `CancellationToken.None` (`ViewModels/AsyncRelayCommand.cs:47`), así que en la práctica el coordinador crea su propio CTS.

`Ejemplo_Docs_GPS/servicio.md` primero descarta inyectar `MainPageViewModel` en el servicio (invierte la dependencia, ciclos de DI, acople a una pantalla; `:3`). Después discute cinco formas de que un servicio dispare GPS + overlay (devolver `GpsResult`, `IProgress<GpsStatus>`, `IGpsUiNotifier`, mensajería, estado singleton; `:9`, `:37`, `:66`, `:103`, `:112`) y recomienda la 1, escalando a 3 o 5 (`servicio.md:127-129`). Lectura de este índice (el documento no lo dice): el coordinador actual, con su overlay compartido en un singleton, se parece más a la opción 5.

⚠️ `servicio.md:11` enlaza `MainPageViewModel.cs#L60-L82` y el documento habla de `AplicarResultado` en el VM (`:29`, `:32-33`, `:58`, `:129`). Ese método ya no existe: hoy la lógica del overlay vive en `GpsCoordinator.Aplicar` y el VM sólo tiene `ActualizarTexto` (`ViewModels/MainPageViewModel.cs:83-97`). En las líneas 60-80 está `MostrarEnMapaAsync`.

## 6. UI: página, VM y overlay

### 6.1 `MainPage.xaml` (`Pages/MainPage.xaml:8-34`)

| Control | Binding / evento | Acción |
|---|---|---|
| Label | `Coordenadas` (`:14`) | salida de texto principal |
| «Obtener ubicación» | `ObtenerUbicacionCommand` (`:15`) | captura y muestra `Lat/Lng` con 6 decimales |
| «Cancelar» | `CancelarCommand` (`:16`) | `_coord.Cancelar()` |
| «Mostrar en mapa» | `MostrarEnMapaCommand` (`:17`) | §7.2 |
| «Prueba» | `Clicked="Button_Clicked"` (`:19`) | code-behind llama al coordinador y escribe en `lbSalida` (`MainPage.xaml.cs:21-25`) |
| Label `Domicilio` + «Obtener Domicilio» | `Domicilio`, `ObtenerDominicilioCommand` (`:25-26`) | §7.1 |
| `controls:GpsStatusOverlayView` | `BindingContext="{Binding Overlay}"` (`:32`) | overlay reutilizable |

El contenido entero se oculta con `IsVisible="{Binding MostrandoContenido}"` (`:10`), y `MostrandoContenido => !Overlay.IsVisible` (`MainPageViewModel.cs:28`), refrescado al escuchar `Overlay.PropertyChanged` (`:41-45`).

### 6.2 `GpsOverlayViewModel` (`ViewModels/GpsOverlayViewModel.cs`)

| Estado | `IsBusy` | `IsDenied` | `CanRetry` | Título |
|---|---|---|---|---|
| `ShowBusy()` (`:76-80`) | true | false | — | — |
| `ShowPermissionDenied(true)` (`:82-93`) | false | true | true | «Permiso de ubicación necesario» |
| `ShowPermissionDenied(false)` | false | true | false | «Acceso a la ubicación denegado» |
| `ShowRestricted()` (`:95-102`) | false | true | false | «Acceso restringido» |
| `Hide()` (`:104-108`) | false | false | — | — |

`IsVisible = _isBusy || _isDenied` (`:66`); `MustOpenSettings = !_canRetry` (`:63`). Comandos: `PedirPermisoCommand` (async, reintento), `AbrirAjustesCommand`, `CerrarOverlayCommand` = `Hide` (`:10-15`).

### 6.3 `GpsStatusOverlayView.xaml` (`Controls/GpsStatusOverlayView.xaml`)

- Panel de espera (`:18-22`): `satelite.gif` animado, «Buscando posición GPS», «Aguarde unos segundos, y será redirigido automáticamente».
- Panel de permisos (`:25-37`): fondo `#CC000000`, ícono `location_off`, título/mensaje, botones «Pedir permiso» (visible si `CanRetry`), «Abrir configuración» (si `MustOpenSettings`), «Cerrar».

## 7. Geocodificación inversa y mapa por navegador

### 7.1 `GoogleMapService.GetDomicilioAsync` (`Services/GoogleMapService.cs:16-41`)

| Aspecto | Detalle | Línea |
|---|---|---|
| Endpoint | `https://maps.googleapis.com/maps/api/geocode/json?latlng={lat},{lng}&key={ApiKey}&language=es` | 12, 22 |
| Cultura | lat/lng con `CultureInfo.InvariantCulture` (punto decimal) | 20-21 |
| Cliente | `static readonly HttpClient` + `GetFromJsonAsync<GeocodeResponse>` | 14, 24 |
| Respuesta | `status`, `error_message`, `results[].formatted_address` (DTO privados) | 43-59 |
| Errores | nunca lanza: devuelve **strings** («Sin respuesta del servicio», `Error: {Status} - {ErrorMessage}`, «Domicilio no disponible» si `formatted_address` es null, «Domicilio no encontrado», `Error al obtener domicilio: …`) | 26-40 |

Flujo en el VM (`ViewModels/MainPageViewModel.cs:101-114`): captura con el coordinador → si `Success`, geocodifica y escribe el domicilio en **`Coordenadas`**; si no, `ActualizarTexto(result)`.

⚠️ El setter de `Domicilio` escribe en `_coordenadas` (`MainPageViewModel.cs:21-26`), notificando `Domicilio` pero sin cambiar `_domicilio`; y `ObtenerDominicilioAsync` tampoco usa `Domicilio`. Resultado: el label ligado a `Domicilio` (`MainPage.xaml:25`) queda siempre vacío y el domicilio aparece en el label de coordenadas.

### 7.2 «Mostrar en mapa» (`ViewModels/MainPageViewModel.cs:60-80`)

Captura → si `Success`, `Browser.Default.OpenAsync("https://maps.google.com/?q={lat},{lng}", BrowserLaunchMode.SystemPreferred)` (`:68-69`); si falla la apertura, mensaje en `Coordenadas`; si no hay `Success`, `ActualizarTexto`. No hay mapa embebido: eso es [05_Mapas](05_Mapas.md).

⚠️ A diferencia de `GoogleMapService`, esta URL interpola los `double` **con la cultura del dispositivo** (`:68`). Con una cultura `es-*` el separador sería coma (`q=-31,63,-60,70`), el mismo defecto que la híbrida corrigió con `InvariantCulture` (`LibApp/UrlCommands/Handlers/GpsCommandHandler.cs:57-59`). Inferido de la lectura del código; no verificado en dispositivo.

### 7.3 Texto por variante (`ActualizarTexto`, `MainPageViewModel.cs:83-97`)

| Variante | Texto |
|---|---|
| `Success` | `Lat: {:F6}, Lng: {:F6}` |
| `PermissionDenied` | «Permiso de ubicación necesario.» |
| `PermissionRestricted` | «Acceso a la ubicación restringido.» |
| `GpsDisabled` | «El GPS está desactivado. Activalo desde ajustes.» |
| `NotSupported` | «Este dispositivo no soporta GPS.» |
| `NoSignal` | «No se pudo obtener ubicación (GPS sin señal).» |
| `Cancelled` | «Operación cancelada por el usuario.» |
| `Failure f` | `Error: {f.Message}` |

## 8. Manifests de plataforma

| Plataforma | Declaración | Fuente |
|---|---|---|
| Android | `ACCESS_FINE_LOCATION` + `ACCESS_COARSE_LOCATION` (el comentario: COARSE acompaña a FINE desde API 31) | `Platforms/Android/AndroidManifest.xml:14,20` |
| Android | `ACCESS_BACKGROUND_LOCATION` comentado | `:26` |
| Android | `uses-feature android.hardware.location.gps required="false"` (instalable sin GPS) | `:32` |
| iOS | `NSLocationWhenInUseUsageDescription` (comentario: sin ella crashea al pedir ubicación) | `Platforms/iOS/Info.plist:60-65` |
| iOS | `NSLocationAlwaysAndWhenInUseUsageDescription` **activa** | `Info.plist:66-67` |
| iOS | Entitlements: `aps-environment development` + `get-task-allow` (idénticos en ambos `.plist`) | `Platforms/iOS/Entitlements.plist`, `Entitlements.Development.plist` |

⚠️ Discrepancias de manifests (material reutilizado de otros ejemplos; describirlas, no «arreglarlas»):
- El texto de uso dice «registrar la posición de las infracciones de tránsito» (`Info.plist:65,67`), heredado de otra app.
- La clave «Always» está activa en `:66-67`, pero el comentario de `:68-71` dice que sólo hace falta para segundo plano y repite la misma clave comentada.
- El comentario «Permisos necesarios para la captura de medios» (`:56`) precede a `LSRequiresIPhoneOS`, que no es un permiso.
- `aps-environment` (push) no lo usa el GPS.
- `Properties/launchSettings.json` sólo trae el perfil «Windows Machine», pero el `.csproj` no apunta a Windows (`csproj:4-5`).

## 9. API keys: por qué no compila recién clonado

```
repo (versionado)                        máquina del dev / CI (no versionado)
Services/ApiKeys.cs.template  ──copiar──► Services/ApiKeys.cs
  GoogleMaps = "REEMPLAZAR_CON_TU_API_KEY"   GoogleMaps = "<key real>"
                                                  ▲
GoogleMapService.cs:11  const ApiKey = ApiKeys.GoogleMaps   (referencia en compilación)
```

| Hecho | Fuente |
|---|---|
| `ApiKeys.cs` ignorado por el patrón `**/Services/ApiKeys.cs` | `.gitignore:420-421` (verificado con `git check-ignore`) |
| El template define `internal static class ApiKeys { const string GoogleMaps }` en `Ejemplo_Maui_GPS.Services` | `Services/ApiKeys.cs.template:9-14` |
| `GoogleMapService` lo consume como `const` | `Services/GoogleMapService.cs:9-11` |
| En el clon limpio (`285f6fb`) **no existe** `Services/ApiKeys.cs` | listado de `GPS/Services/` |

⇒ Recién clonado, `ApiKeys` no existe y el proyecto **no compila** (error de símbolo inexistente en `GoogleMapService.cs:11`; código exacto del compilador no verificado). Arreglo local: copiar el template a `ApiKeys.cs` y poner la key (`Ejemplo_Docs_GPS/secret.md:99-106`). Con el placeholder compila, pero la geocodificación devuelve el texto de error de Google (§7.1); obtener ubicación y abrir el mapa no usan la key.

En CI, el workflow del GPS genera `ApiKeys.cs` desde el secreto `GOOGLE_MAPS_API_KEY` (`.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:279-285`); en qué otros workflows está ese paso (comentado o ausente): [09 §10](09_CI-CD-y-Build.md).

Estrategias de `secret.md` (`:7-67`) y elección de la cátedra (`:71-78`):

| # | Estrategia | Uso según el documento |
|---|---|---|
| 1 | archivo local ignorado (`secrets.json`) | lo más simple, fácil de subir por error |
| 2 | User Secrets de .NET | sólo Debug |
| 3 ⭐ | clase estática `ApiKeys.cs` ignorada + `.template` versionado | **elegida**: didáctica, sin NuGet, Debug y Release, una key por alumno |
| 4 | `SecureStorage` | keys obtenidas en runtime |
| 5 | backend intermedio | lo más seguro: la key no vive en el cliente |

Buenas prácticas listadas (`secret.md:110-115`): restringir la key por package + SHA-1 y por API (sólo Geocoding), cuotas/alertas, revocar si se filtró (borrar el commit no alcanza).

⚠️ La key viaja en la query string de una llamada directa del cliente (`GoogleMapService.cs:22`): es la opción 3, no la 5; queda en el binario. Es deliberado y está explicado en `secret.md`.

## 10. Modos de entrega de la coordenada (app híbrida)

El ejemplo autónomo sólo muestra texto. En `Ejemplo_Maui_Hibrida` la coordenada vuelve a la web por el puente de comandos de URL, en dos modos que decide `GpsCommandHandler.DeliveryFor` (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/UrlCommands/Handlers/GpsCommandHandler.cs:41-43`): **Injection** si la URL trae `param={id}` (JS sobre la página viva) y **Substitution** si no lo trae (re-navega con `Latitud=…&Longitud=…` o el centinela `0.0/0.0`, más nonce `_ts=`). El detalle (JS emitido, centinela, nonce, `InvariantCulture`, lectura en la web) está en [08 §6.2](08_App-Hibrida-Integrada.md); el mecanismo general (`CommandDelivery`, `UrlPlan`, dispatcher), en [08 §4 y §6.1](08_App-Hibrida-Integrada.md).

Diferencias del servicio híbrido respecto del autónomo (`LibApp/Devices/GPS/Services/GpsService.cs`):

| Aspecto | `Ejemplo_Maui_GPS` | `Ejemplo_Maui_Hibrida` |
|---|---|---|
| Lectura | última conocida si < 1 min; si no, Medium 10 s | siempre lectura fresca Medium 15 s, sin `GetLastKnownLocationAsync` (`:37-43`, ADR-0009 «Opción A») |
| Permisos | clase aparte `LocationPermissionService` | dentro del servicio (`:66-94`), detrás de `IGpsService` |
| Ajustes | sólo los de la app | además `OpenLocationSettings()` (intent `ACTION_LOCATION_SOURCE_SETTINGS` en Android, `:106-115`) |
| Errores | texto en el VM; overlay sólo para permisos | catálogo con códigos `GPS-*` (`LibApp/Devices/GPS/Models/GpsErrorCatalog.cs:16-23`) y pantalla por variante |

## 11. Ejecución, build y documentación

| Pieza | Qué es | Detalle en |
|---|---|---|
| `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat` | `dotnet build -f net10.0-android -t:Run` sobre `..\GPS\Ejemplo_Maui_GPS\Ejemplo_Maui_GPS.csproj`, comprueba antes que `adb` vea un dispositivo (`:6-7`, `:30-37`) | [09](09_CI-CD-y-Build.md) |
| `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml` | build para el simulador iOS, firma ad hoc, zip del `.app`, GIF con `Utilities/simular.sh`; sólo `workflow_call` (el `push` está comentado, `:3-15`) | [09](09_CI-CD-y-Build.md) |
| `Ejemplos_Devices/Ejemplos_Devices.slnx:56-63` | carpeta de solución `/Ejemplos/GPS/` + `Docs` | [09](09_CI-CD-y-Build.md) |
| `README.md:20-21` | enlaza `Ejemplo_Docs_GPS/Readme.md`, «En construcción …» | [10](10_Documentacion-Transversal.md) |

Documentos del repo de documentación sobre GPS (existencia verificada):
- `Ejemplos_Maui_Devices-docs/docs/pieces/gps/README.md` — PIECE-GPS-001, ficha reverse-engineered del ejemplo.
- `Ejemplos_Maui_Devices-docs/docs/04-decisions/0007-secretos-fuera-del-repo.md`: ADR-0007, que reconstruye la decisión de la plantilla `ApiKeys` más los secrets de CI (§9). El código no lo cita.
- `Ejemplos_Maui_Devices-docs/docs/04-decisions/0009-ciclo-vida-overlay-gps-y-cancelacion-navegacion.md` — ADR-0009, citado desde el código híbrido (`GpsService.cs:37`, `GpsCommandHandler.cs:12`).

⚠️ Trampas de build y ejecución:
- `Ejemplos_Devices.slnx:62` apunta a `GPS/Docs_GPS/Readme.md`, pero la carpeta real es `GPS/Ejemplo_Docs_GPS/`: la entrada de solución queda rota.
- `Utilities/end2end/com.ejemplos.devices.gps.yaml` toca «Geo Posicionar», «Llamar» y «Leer QR» (`:20,24,30`), botones de la app híbrida (y con el texto viejo; ver `CHANGELOG.md:143`) que **no existen** en `Ejemplo_Maui_GPS`. Además, el workflow GPS corre `simular.sh` (`cd-ios-gps.Ejemplo_GPS.yml:418-419`), no `simular_ui.sh`, que es el que ejecuta Maestro.
- `Ejemplo_Photo_MediaPicker_launch.bat:23` imprime el encabezado «Ejemplo_Maui_GPS» (copia del `.bat` de GPS); ver [09](09_CI-CD-y-Build.md).

## 12. Preguntas frecuentes y trampas de lectura

| Pregunta | Respuesta corta |
|---|---|
| ¿Por qué no compila recién clonado? | Falta `Services/ApiKeys.cs` (ignorado). Copiá el `.template` (§9). |
| ¿Hace falta la key para obtener la ubicación? | No; sólo para «Obtener Domicilio» (§7.1, §9). |
| ¿Qué precisión y timeout usa? | Medium, 10 s, salvo que la última posición conocida tenga menos de 1 min (§4.2). |
| ¿Dónde se decide el overlay? | En `GpsCoordinator.Aplicar`, no en el VM (§5). |
| ¿Por qué el overlay no muestra «GPS apagado»? | `Aplicar` sólo maneja permisos; el resto oculta el overlay y deja el texto (§5). La híbrida sí tiene pantalla por variante (§10). |
| ¿El botón «Cancelar» corta una búsqueda en curso? | ⚠️ Mientras el overlay está ocupado, todo el contenido (y con él «Cancelar») está oculto (`MainPage.xaml:10`, `MainPageViewModel.cs:28`), y el panel de espera no tiene botón de cancelar (`GpsStatusOverlayView.xaml:18-22`). Inferido del binding; no verificado en dispositivo. |
| Tras «Pedir permiso» en el overlay, ¿se actualiza el texto? | ⚠️ No: el reintento llama a `CapturarAsync()` desde el coordinador (`GpsCoordinator.cs:25`) y el VM no recibe ese resultado; `Coordenadas` conserva el mensaje anterior. |
| ¿Por qué el domicilio sale en el label de coordenadas? | ⚠️ Setter de `Domicilio` escribe `_coordenadas` y el método usa `Coordenadas` (§7.1). |
| ¿Para qué está `TestService`? | Envoltorio de `GpsService` de la opción 1 de `servicio.md`, registrado en DI (`MauiProgram.cs:35`) sin consumidores. Es didáctico. |
| ¿Por qué hay `ShouldShowRationale` con `#if ANDROID`? | Sólo Android distingue «se puede volver a pedir»; en iOS todo rechazo va a ajustes (§4.1). |
| ¿El texto «será redirigido automáticamente» aplica aquí? | ⚠️ Viene del contexto híbrido; en el ejemplo autónomo no hay redirección, sólo se oculta el overlay (`GpsStatusOverlayView.xaml:21`). |
| Nombres raros: `ObtenerDominicilio*` | Así está escrito en el código (`MainPageViewModel.cs:50,99-101`, `MainPage.xaml:26`); buscalo con esa grafía. |
| `gps.png` y `dotnet_bot.png` | Se empaquetan (`csproj:73-74`), pero ningún `.cs`/`.xaml` del ejemplo los referencia; el overlay usa `satelite.gif`. |
