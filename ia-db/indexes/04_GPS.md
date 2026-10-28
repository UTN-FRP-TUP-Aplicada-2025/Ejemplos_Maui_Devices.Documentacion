# 04 — GPS y ubicación

> **Propósito:** mapa navegable del dominio «GPS y ubicación» del repositorio `Ejemplos_Maui_Devices`: el proyecto `Ejemplo_Maui_GPS` (lectura de `Geolocation`, permisos de ubicación normalizados, geocodificación inversa con Google, overlay MVVM de espera/permisos) y su prosa de apoyo (`Ejemplo_Docs_GPS`).
> **Fuente primaria:** `Ejemplos_Devices/GPS/**` (commit `285f6fb`). Contexto secundario: `.gitignore`, `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml`, `Ejemplos_Devices/Ejemplos_Devices.slnx`, `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat`, `Utilities/end2end/com.ejemplos.devices.gps.yaml`.
> Todas las rutas son relativas a la raíz del repo de código. La versión consolidada del GPS dentro de la app híbrida (`LibApp/Devices/GPS/`) se documenta en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md); el control `Map` se documenta en [05_Mapas](05_Mapas.md).

---

## §1 Mapa del dominio

| Proyecto | Ruta | Técnica | Archivos clave | En `.slnx` | Workflow CI |
|---|---|---|---|---|---|
| `Ejemplo_Maui_GPS` | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/` | `Geolocation` + `Permissions.LocationWhenInUse` + resultado tipado (`GpsResult`) + coordinador singleton dueño del overlay + Geocoding HTTP de Google | `Services/GpsService.cs`, `Services/GpsCoordinator.cs`, `Services/LocationPermissionService.cs`, `Services/GoogleMapService.cs`, `ViewModels/MainPageViewModel.cs`, `Controls/GpsStatusOverlayView.xaml` | Sí (`Ejemplos_Devices.slnx:57`) | Sí: `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml` (solo `workflow_call`, ver §9) |
| `Ejemplo_Docs_GPS` (no es proyecto) | `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/` | Prosa: enlace a la doc oficial, manejo de API keys, alternativas de diseño servicio↔overlay | `Readme.md`, `secret.md`, `servicio.md` | Parcial (⚠️ ruta errónea, ver §10) | — |

Árbol versionado (sin recursos binarios ni `Resources/Styles`):

```
Ejemplos_Devices/GPS/
├── Ejemplo_Docs_GPS/
│   ├── Readme.md            ← 1 enlace a learn.microsoft.com (geolocation)
│   ├── secret.md            ← 5 opciones de manejo de API keys; la cátedra usa la opción 3
│   └── servicio.md          ← 5 alternativas para que un servicio dispare el overlay
└── Ejemplo_Maui_GPS/
    ├── Ejemplo_Maui_GPS.csproj
    ├── MauiProgram.cs       ← registro DI (AddServices)
    ├── App.xaml(.cs) · AppShell.xaml(.cs)
    ├── Controls/GpsStatusOverlayView.xaml(.cs)
    ├── Pages/MainPage.xaml(.cs)
    ├── Services/
    │   ├── ApiKeys.cs.template          ← versionado; ApiKeys.cs NO (gitignore)
    │   ├── GoogleMapService.cs          ← geocodificación inversa
    │   ├── GpsCoordinator.cs            ← punto único de captura + overlay
    │   ├── GpsResult.cs                 ← record jerárquico de resultados
    │   ├── GpsService.cs                ← permisos + lectura GPS
    │   ├── LocationPermissionResult.cs  ← enum normalizado de permisos
    │   ├── LocationPermissionService.cs
    │   └── TestService.cs               ← ejemplo de servicio consumidor (sin uso)
    ├── ViewModels/ AsyncRelayCommand.cs · GpsOverlayViewModel.cs · MainPageViewModel.cs · ViewModelBase.cs
    ├── Platforms/Android/ AndroidManifest.xml · MainActivity.cs · MainApplication.cs
    ├── Platforms/iOS/ Info.plist · Entitlements.plist · Entitlements.Development.plist · AppDelegate.cs · Program.cs
    └── Resources/ Images/satelite.gif · gps.png · Fonts/MaterialIconsOutlined-Regular.otf · OpenSans-*
```

---

## §2 Stack y configuración del proyecto

| Aspecto | Valor | Fuente |
|---|---|---|
| TFM | `net10.0-android`; agrega `net10.0-ios` solo si se compila en macOS | `Ejemplo_Maui_GPS.csproj:4-5` |
| Windows | No hay TFM de Windows (`WindowsPackageType=None`); el perfil «Windows Machine» de `launchSettings.json` es residuo de plantilla | `Ejemplo_Maui_GPS.csproj:23`, `Properties/launchSettings.json:3` |
| `ApplicationId` | `com.ejemplos.devices.gps` | `Ejemplo_Maui_GPS.csproj:18` |
| Mínimos | iOS 15.0 · Android 25 (API 25) | `Ejemplo_Maui_GPS.csproj:25-26` |
| Android | `targetSdkVersion=36`; RIDs arm/arm64/x86/x64; páginas de 16 KB (`-Wl,-z,max-page-size=16384`); APK (no AAB) | `AndroidManifest.xml:34`, `Ejemplo_Maui_GPS.csproj:33-50` |
| iOS Release | intérprete + trimming `SdkOnly`; AOT solo para `ios-arm64` | `Ejemplo_Maui_GPS.csproj:54-64` |
| NuGets | `Microsoft.Maui.Controls` = `$(MauiVersion)` (lo aporta el workload; no hay `Directory.*.props` ni `global.json` versionados) · `Microsoft.Extensions.Logging.Debug` 10.0.2 | `Ejemplo_Maui_GPS.csproj:82-83` |
| Fuentes | OpenSans Regular/Semibold + `MaterialIconsOutlined` (glifo `location_off` del overlay); `EnableMauiFontProcessing=true` | `MauiProgram.cs:17-19`, `Ejemplo_Maui_GPS.csproj:28` |
| Librerías MVVM | Ninguna: `ViewModelBase` y `AsyncRelayCommand`/`RelayCommand` propios | `ViewModels/ViewModelBase.cs:8`, `ViewModels/AsyncRelayCommand.cs:9-76` |

---

## §3 Arquitectura y flujo

### §3.1 Registro DI

| Tipo | Ciclo de vida | Fuente |
|---|---|---|
| `LocationPermissionService` | Singleton | `MauiProgram.cs:32` |
| `GpsService` | Singleton | `MauiProgram.cs:33` |
| `GpsCoordinator` | Singleton (dueño del `GpsOverlayViewModel`) | `MauiProgram.cs:34` |
| `TestService` | Singleton (registrado, sin consumidores) | `MauiProgram.cs:35` |
| `GoogleMapService` | Singleton | `MauiProgram.cs:36` |
| `MainPageViewModel` | Transient | `MauiProgram.cs:39` |
| `MainPage` | Transient (Shell la resuelve por `DataTemplate`) | `MauiProgram.cs:42`, `AppShell.xaml:10` |

### §3.2 Capas

```
MainPage (XAML)  ──BindingContext──▶ MainPageViewModel ──▶ GpsCoordinator (singleton)
   │                                   │  (texto: Coordenadas)     │  ├─ Overlay: GpsOverlayViewModel
   │                                   └─▶ GoogleMapService        │  └─ _cts (cancelación)
   └─ GpsStatusOverlayView ◀─Binding Overlay─────────────────────────┘
                                                                    ▼
                                                     GpsService ──▶ LocationPermissionService
                                                        │              └─ Permissions.*<LocationWhenInUse>
                                                        └─ Geolocation.GetLastKnownLocationAsync / GetLocationAsync
```

- **Servicio** (`GpsService`) compone permisos + lectura y traduce a `GpsResult` las excepciones de la **lectura** (`Services/GpsService.cs:34-67`); ⚠️ la llamada a permisos (`:18`) queda **fuera** del `try`, así que una excepción de `Permissions.*` sí se propaga (ver §10 #12).
- **Coordinador** (`GpsCoordinator`) es el «punto único de entrada»: muestra el overlay, llama al servicio, aplica el resultado al overlay y lo devuelve al caller (`Services/GpsCoordinator.cs:5-9`, `:28-51`).
- **VM de página** solo traduce el resultado a texto (`ViewModels/MainPageViewModel.cs:6`, `:83-97`).
- **Overlay** es un `ContentView` autónomo cuyo `BindingContext` es el `GpsOverlayViewModel` del coordinador (`Pages/MainPage.xaml:32`, `Controls/GpsStatusOverlayView.xaml:12-13`).

### §3.3 Secuencia «Obtener ubicación»

| Paso | Qué ocurre | Fuente |
|---|---|---|
| 1 | Botón → `ObtenerUbicacionCommand` (`AsyncRelayCommand`) | `Pages/MainPage.xaml:15`, `ViewModels/MainPageViewModel.cs:47` |
| 2 | VM pone «Obteniendo ubicación GPS...» y llama `_coord.CapturarAsync(ct)` | `ViewModels/MainPageViewModel.cs:52-57` |
| 3 | Coordinador cancela/descarta un `_cts` previo y crea uno nuevo (o enlazado al `ct` externo) | `Services/GpsCoordinator.cs:30-32` |
| 4 | `Overlay.ShowBusy()` en el hilo de UI → se ve `satelite.gif` + «Buscando posición GPS» | `Services/GpsCoordinator.cs:34`, `Controls/GpsStatusOverlayView.xaml:18-22` |
| 5 | `GpsService.ObtenerUbicacionAsync`: pide permiso; si no es `Granted`, devuelve `PermissionRestricted`/`PermissionDenied(CanRetry)` | `Services/GpsService.cs:18-30` |
| 6 | Lectura: última posición conocida si tiene < 1 min; si no, `GetLocationAsync(Medium, 10 s, ct)` | `Services/GpsService.cs:43-48` |
| 7 | Excepciones → `Cancelled` / `GpsDisabled` / `NotSupported` / `Failure` | `Services/GpsService.cs:52-67` |
| 8 | Coordinador aplica al overlay: denegado → panel de permisos; restringido → panel restringido; resto → `Hide()` | `Services/GpsCoordinator.cs:56-70` |
| 9 | VM traduce el `GpsResult` a texto en `Coordenadas` | `ViewModels/MainPageViewModel.cs:83-97` |

Otros disparadores del mismo coordinador:

| Disparador | Qué hace con el resultado | Fuente |
|---|---|---|
| «Mostrar en mapa» (`MostrarEnMapaCommand`) | Si `Success`, abre `https://maps.google.com/?q=lat,lng` con `Browser.Default.OpenAsync(..., SystemPreferred)` | `ViewModels/MainPageViewModel.cs:60-80` |
| «Obtener Domicilio» (`ObtenerDominicilioCommand`) | Si `Success`, llama `GoogleMapService.GetDomicilioAsync` | `ViewModels/MainPageViewModel.cs:101-114` |
| «Prueba» (code-behind) | Llama `_coord.CapturarAsync()` desde fuera del VM y escribe en `lbSalida` — demuestra que el overlay funciona sin el VM | `Pages/MainPage.xaml:19-21`, `Pages/MainPage.xaml.cs:19-25` |
| «Pedir permiso» del overlay | Callback `onRetry` → `CapturarAsync()` del coordinador | `Services/GpsCoordinator.cs:25`, `ViewModels/GpsOverlayViewModel.cs:12` |
| «Cancelar» | `_coord.Cancelar()` → `_cts.Cancel()`; ⚠️ el botón está dentro del contenido que se oculta mientras el overlay está visible (ver §10 #3) | `ViewModels/MainPageViewModel.cs:48`, `Services/GpsCoordinator.cs:53`, `Pages/MainPage.xaml:10`, `:16` |

---

## §4 APIs de MAUI usadas

| API | Uso | Fuente |
|---|---|---|
| `Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>()` | Estado actual | `Services/LocationPermissionService.cs:9`, `:15` |
| `Permissions.RequestAsync<Permissions.LocationWhenInUse>()` | Pide al SO | `Services/LocationPermissionService.cs:22` |
| `Permissions.ShouldShowRationale<...>()` (solo `#if ANDROID`) | Distingue «se puede reintentar» de «no volver a preguntar» | `Services/LocationPermissionService.cs:35-37` |
| `AppInfo.ShowSettingsUI()` | Botón «Abrir configuración» | `Services/LocationPermissionService.cs:43` |
| `Geolocation.GetLastKnownLocationAsync()` | Caché del SO (frescura < 1 min) | `Services/GpsService.cs:43-44` |
| `Geolocation.GetLocationAsync(GeolocationRequest, ct)` | Lectura nueva, `GeolocationAccuracy.Medium`, timeout 10 s | `Services/GpsService.cs:46-47` |
| `FeatureNotEnabledException` / `FeatureNotSupportedException` | GPS apagado / sin hardware | `Services/GpsService.cs:56-63` |
| `MainThread.InvokeOnMainThreadAsync` | Mutaciones del overlay en hilo de UI | `Services/GpsCoordinator.cs:34`, `:38`, `:43` |
| `Browser.Default.OpenAsync(url, BrowserLaunchMode.SystemPreferred)` | Abrir Google Maps web | `ViewModels/MainPageViewModel.cs:69` |

### §4.1 Resultados tipados

`GpsResult` es un `abstract record` con casos anidados (`Services/GpsResult.cs:5-15`):

| Caso | Origen | Texto que muestra el VM (`MainPageViewModel.cs:85-95`) | Overlay |
|---|---|---|---|
| `Success(Location)` | lectura OK | `Lat: …, Lng: …` (F6) | se oculta |
| `PermissionDenied(CanRetry)` | `Denied` / `DeniedCanRetry` | «Permiso de ubicación necesario.» | panel de permisos |
| `PermissionRestricted` | `Restricted` | «Acceso a la ubicación restringido.» | panel restringido |
| `GpsDisabled` | `FeatureNotEnabledException` | «El GPS está desactivado…» | se oculta |
| `NotSupported` | `FeatureNotSupportedException` | «Este dispositivo no soporta GPS.» | se oculta |
| `NoSignal` | `location is null` | «…(GPS sin señal).» | se oculta |
| `Cancelled` | `OperationCanceledException` | «Operación cancelada por el usuario.» | se oculta |
| `Failure(Message)` | cualquier otra excepción | `Error: …` | se oculta |

`LocationPermissionResult` (`Services/LocationPermissionResult.cs:5-11`) normaliza Android+iOS en cuatro valores: `Granted`, `DeniedCanRetry` (Android con rationale), `Denied` (Android «no volver a preguntar» / iOS denegado), `Restricted` (iOS: MDM/control parental).

### §4.2 Overlay (`GpsOverlayViewModel` + `GpsStatusOverlayView`)

| Estado | Propiedades | Botones visibles | Fuente |
|---|---|---|---|
| Espera | `IsBusy=true` | — (animación) | `ViewModels/GpsOverlayViewModel.cs:76-80`, `Controls/GpsStatusOverlayView.xaml:18-22` |
| Denegado reintentable | `IsDenied=true`, `CanRetry=true` | «Pedir permiso», «Cerrar» | `ViewModels/GpsOverlayViewModel.cs:82-93`, `Controls/GpsStatusOverlayView.xaml:32`, `:35` |
| Denegado definitivo / restringido | `IsDenied=true`, `MustOpenSettings=true` | «Abrir configuración», «Cerrar» | `ViewModels/GpsOverlayViewModel.cs:63`, `:95-102`, `Controls/GpsStatusOverlayView.xaml:34` |
| Oculto | `IsVisible = IsBusy \|\| IsDenied` = false | — | `ViewModels/GpsOverlayViewModel.cs:66`, `:104-108` |

La página oculta su contenido mientras el overlay está visible: `MostrandoContenido => !Overlay.IsVisible`, propagado por suscripción a `PropertyChanged` (`ViewModels/MainPageViewModel.cs:28`, `:41-45`; `Pages/MainPage.xaml:10`).

---

## §5 Geocodificación inversa (`GoogleMapService`)

| Aspecto | Detalle | Fuente |
|---|---|---|
| Endpoint | `https://maps.googleapis.com/maps/api/geocode/json?latlng={lat},{lng}&key={ApiKey}&language=es` | `Services/GoogleMapService.cs:12`, `:22` |
| Formato numérico | `ToString(CultureInfo.InvariantCulture)` para lat/lng (evita coma decimal) | `Services/GoogleMapService.cs:20-21` |
| Cliente | `static readonly HttpClient` (instancia única) + `GetFromJsonAsync<GeocodeResponse>` | `Services/GoogleMapService.cs:14`, `:24` |
| DTOs | `GeocodeResponse` (`status`, `error_message`, `results`) y `GeocodeResult` (`formatted_address`), privados y `sealed` | `Services/GoogleMapService.cs:43-59` |
| Errores | Se devuelven como **texto** (no excepción): «Sin respuesta…», `Error: {status} - {error_message}`, «Domicilio no encontrado», `Error al obtener domicilio: …` | `Services/GoogleMapService.cs:26-40` |
| Resultado | Primer `formatted_address` | `Services/GoogleMapService.cs:32-33` |

La guía de la cátedra recomienda restringir la key por package name + SHA-1 y por API (solo Geocoding) y poner cuotas (`Ejemplo_Docs_GPS/secret.md:112-113`).

---

## §6 API keys: por qué no compila recién clonado

| Pieza | Estado en el repo | Fuente |
|---|---|---|
| `Services/ApiKeys.cs.template` | **Versionado**; define `internal static class ApiKeys { const string GoogleMaps = "REEMPLAZAR_CON_TU_API_KEY"; }` | `Services/ApiKeys.cs.template:11-14` |
| `Services/ApiKeys.cs` | **Ignorado** por `**/Services/ApiKeys.cs` | `.gitignore:420-421` |
| Consumidor | `private const string ApiKey = ApiKeys.GoogleMaps;` | `Services/GoogleMapService.cs:11` |

⚠️ **Recién clonado no compila**: `GoogleMapService.cs:11` referencia la clase `ApiKeys`, que solo existe en el archivo ignorado. Hasta copiar la plantilla, el build falla con un error de símbolo inexistente (código de error exacto no verificado). Es el comportamiento buscado por la opción 3 de `secret.md`.

Pasos (según `Ejemplo_Docs_GPS/secret.md:80-106`):

```powershell
# desde Ejemplos_Devices/GPS/Ejemplo_Maui_GPS
Copy-Item Services/ApiKeys.cs.template Services/ApiKeys.cs
# editar ApiKeys.cs y poner la key propia
git check-ignore -v Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/ApiKeys.cs   # desde la raíz
```

Notas:
- Con el placeholder sin reemplazar el proyecto **compila**, pero Google responde con un `status` de error que `GetDomicilioAsync` devuelve como texto (`Services/GoogleMapService.cs:29-30`).
- En CI la plantilla se materializa con `sed` sobre el secreto `GOOGLE_MAPS_API_KEY` (`.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:279-285`).

Opciones que describe `secret.md` (`Ejemplo_Docs_GPS/secret.md:7-67`):

| # | Opción | Recomendación del doc |
|---|---|---|
| 1 | Archivo local ignorado (`secrets.json`) | simple, riesgo de subirlo |
| 2 | User Secrets de .NET | «recomendado para desarrollo serio», solo Debug |
| 3 | Clase estática + `.template` | **la que usa la cátedra** (`secret.md:73`) |
| 4 | `SecureStorage` | claves obtenidas en runtime |
| 5 | Backend intermedio | la más segura |

---

## §7 Permisos y configuración por plataforma

### §7.1 Android (`Platforms/Android/AndroidManifest.xml`)

| Elemento | Valor | Línea |
|---|---|---|
| `ACCESS_FINE_LOCATION` | declarado | `:14` |
| `ACCESS_COARSE_LOCATION` | declarado (acompañante de FINE desde API 31, según comentario) | `:18-20` |
| `ACCESS_BACKGROUND_LOCATION` | **comentado** (solo para segundo plano) | `:22-26` |
| `uses-feature android.hardware.location.gps` | `required="false"` (instalable sin GPS) | `:32` |
| `uses-sdk` | min 25 · target 36 | `:34` |
| `INTERNET` | **no declarado** (ver §10) | — |

### §7.2 iOS (`Platforms/iOS/Info.plist`)

| Clave | Valor | Línea |
|---|---|---|
| `NSLocationWhenInUseUsageDescription` | «…registrar la posición de las infracciones de tránsito.» (obligatoria: sin ella la app se cierra al pedir ubicación, según el comentario) | `:60-65` |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | mismo texto, **activa** | `:66-67` |
| `CFBundleIdentifier` / versiones | `com.ejemplos.devices.gps` · 1.0 / 1.0 (leídas por el workflow para nombrar el zip) | `:44-51` |
| `MinimumOSVersion` | 15.0 | `:35-36` |
| Entitlements | `aps-environment=development`, `get-task-allow=true` (igual en ambos plist) | `Platforms/iOS/Entitlements.plist:5-9` |

---

## §8 Prosa de diseño: `servicio.md`

`Ejemplo_Docs_GPS/servicio.md` discute cómo hacer que un servicio (`TestService`) use el GPS y que el overlay reaccione, descartando inyectar el VM en el servicio (`servicio.md:3`).

| # | Alternativa | Estado en el código actual |
|---|---|---|
| 1 | El servicio devuelve `GpsResult` y el VM aplica (`AplicarResultado`) | `TestService` existe con esa forma (`Services/TestService.cs:3-10`) pero nadie lo consume |
| 2 | `IProgress<GpsStatus>` | no implementado |
| 3 | `IGpsUiNotifier` inyectable | no implementado |
| 4 | Event aggregator / `WeakReferenceMessenger` | no implementado |
| 5 | Estado observable compartido en singleton | **es lo más cercano a lo implementado**: `GpsCoordinator` singleton dueño del overlay (`Services/GpsCoordinator.cs:5-9`) |

⚠️ El doc recomienda la opción 1 (`servicio.md:127-129`) y enlaza `MainPageViewModel.cs#L60-L82` con un método `AplicarResultado` (`servicio.md:11`); en el código actual ese método **no existe**: la aplicación al overlay vive en `GpsCoordinator.Aplicar` (`Services/GpsCoordinator.cs:56`) y el VM solo tiene `ActualizarTexto` (`ViewModels/MainPageViewModel.cs:83`). El doc refleja una etapa previa del diseño.

---

## §9 Build, ejecución y CI

| Vía | Detalle | Fuente |
|---|---|---|
| Android local | `scripts/Ejemplo_Maui_GPS_launch.bat`: verifica `adb devices` y corre `dotnet build -f net10.0-android -t:Run` | `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat:5-7`, `:30-37` |
| iOS CI | `cd-ios-gps.Ejemplo_GPS.yml`: `macos-15`, Xcode 26.0 desde Google Drive, .NET SDK 10.0.300, workloads `ios maui maui-ios` 10.0.100, build Release `iossimulator-arm64`, firma ad-hoc, zip del `.app`, GIF por `Utilities/simular.sh` | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:23-47`, `:200`, `:295-317`, `:337-369`, `:411-419` |
| Disparo del CI | Solo `workflow_call`; el `push` está comentado y ningún workflow del repo lo invoca (`uses: ./…` no aparece) → no corre solo | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:3-15` |
| Paso exclusivo GPS | `PRE-BUILD. Generar ApiKeys.cs desde secreto` (activo); en `cd-ios-phone.Ejemplo_DirectCall.yml:279-285` el mismo bloque está comentado | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:279-285` |

---

## §10 Trampas y discrepancias ⚠️

| # | Trampa / discrepancia | Evidencia |
|---|---|---|
| 1 | ⚠️ **No compila recién clonado** sin `Services/ApiKeys.cs` (ver §6). | `Services/GoogleMapService.cs:11`, `.gitignore:421` |
| 2 | ⚠️ **«Obtener Domicilio» nunca llena su `Label`**: el setter de `Domicilio` escribe `_coordenadas` (y notifica `Domicilio`), y `ObtenerDominicilioAsync` asigna el resultado a `Coordenadas`. El domicilio aparece en el label de coordenadas; el label ligado a `Domicilio` queda vacío. | `ViewModels/MainPageViewModel.cs:21-26`, `:106-108`; `Pages/MainPage.xaml:25` |
| 3 | ⚠️ **La cancelación no es alcanzable desde la UI**: `AsyncRelayCommand.Execute` siempre pasa `CancellationToken.None`, y el botón «Cancelar» (→ `GpsCoordinator.Cancelar()`) vive dentro del `VerticalStackLayout` con `IsVisible="{Binding MostrandoContenido}"`, que es `false` justo mientras dura la lectura (`IsBusy=true` → overlay visible). El panel de espera no tiene botón de cancelar; en la práctica la lectura termina por el timeout de 10 s. | `ViewModels/AsyncRelayCommand.cs:47`, `Pages/MainPage.xaml:10`, `:16`, `ViewModels/MainPageViewModel.cs:28`, `ViewModels/GpsOverlayViewModel.cs:66`, `Controls/GpsStatusOverlayView.xaml:18-22` |
| 4 | ⚠️ **«Mostrar en mapa» interpola lat/lng con la cultura actual**: en una cultura con coma decimal (p. ej. es-AR) la URL `?q=` queda mal formada. `GoogleMapService` sí usa `InvariantCulture`. Efecto en dispositivo no verificado. | `ViewModels/MainPageViewModel.cs:68` vs `Services/GoogleMapService.cs:20-21` |
| 5 | ⚠️ `DefaultTimeout` (15 s) declarado y sin uso; la lectura activa usa `Medium` + 10 s; la variante `Best` + `DefaultTimeout` quedó comentada. | `Services/GpsService.cs:8`, `:36-41`, `:46` |
| 6 | `GetLastKnownLocationAsync()` no recibe el `ct`: la cancelación solo afecta a `GetLocationAsync`. | `Services/GpsService.cs:43`, `:47` |
| 7 | ⚠️ El comentario de `Info.plist` dice que `NSLocationAlwaysAndWhenInUseUsageDescription` es «solo necesario» para segundo plano, pero la clave está **activa** (la versión comentada es un duplicado). El texto de propósito («infracciones de tránsito») viene de otra app. | `Platforms/iOS/Info.plist:64-71` |
| 8 | Comentario «Permisos necesarios para la captura de medios» sobre `LSRequiresIPhoneOS`: residuo de copia desde los ejemplos de cámara. | `Platforms/iOS/Info.plist:56-58` |
| 9 | ⚠️ El manifiesto Android no declara `INTERNET`, y el ejemplo hace HTTP a Google (geocoding y `Browser`). Es el **único** de los 22 `AndroidManifest.xml` versionados que no lo declara. Si el toolchain lo agrega solo en algún modo de build (p. ej. Debug) es algo no verificado en el repo. | `Platforms/Android/AndroidManifest.xml:1-36`, `Services/GoogleMapService.cs:24` |
| 10 | `TestService` está registrado en DI y no lo inyecta nadie: es material para `servicio.md`. | `MauiProgram.cs:35`, `Services/TestService.cs:3` |
| 11 | `LocationPermissionService.CheckAsync()` no se usa y siempre mapea los estados no concedidos a `DeniedCanRetry` (`puedeReintentar: true`). | `Services/LocationPermissionService.cs:7-11`, `:45-50` |
| 12 | ⚠️ El `catch` de `CapturarAsync` (oculta el overlay y relanza) se activa si falla `_permissions.RequestAsync()`, que en `GpsService` está **antes** del `try` (p. ej. una `PermissionException` si faltara la declaración en el manifiesto/Info.plist). La excepción relanzada llega al `async void Execute` de `AsyncRelayCommand` sin `catch`; el efecto en dispositivo (cierre de la app) no está verificado. | `Services/GpsCoordinator.cs:41-45`, `Services/GpsService.cs:18`, `:34`, `ViewModels/AsyncRelayCommand.cs:40-52` |
| 13 | El overlay de espera dice «será redirigido automáticamente», pero no hay navegación: el overlay solo se oculta. | `Controls/GpsStatusOverlayView.xaml:21`, `Services/GpsCoordinator.cs:66-67` |
| 14 | ⚠️ `.slnx` apunta a `GPS/Docs_GPS/Readme.md`, pero la carpeta real es `GPS/Ejemplo_Docs_GPS/`; además `secret.md` y `servicio.md` no figuran en la solución. | `Ejemplos_Devices/Ejemplos_Devices.slnx:62` |
| 15 | ⚠️ `Utilities/end2end/com.ejemplos.devices.gps.yaml` (flujo Maestro con el nombre del bundle GPS) toca «Geo Posicionar», «Llamar», «Leer QR»: botones de la app híbrida, no de `Ejemplo_Maui_GPS` («Obtener ubicación», «Cancelar»…). El workflow GPS usa `simular.sh` (sin Maestro), así que este flujo no se ejecuta. | `Utilities/end2end/com.ejemplos.devices.gps.yaml:19-31`, `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:418-419`, `Utilities/simular_ui.sh:28` |
| 16 | El README raíz marca GPS como «En construcción…» y `Ejemplo_Docs_GPS/Readme.md` es solo un enlace a la documentación oficial. | `README.md:20-21`, `Ejemplo_Docs_GPS/Readme.md:2` |
| 17 | `MainPage` recibe el `MainPageViewModel` y **también** el `GpsCoordinator` (para el botón «Prueba»): son el mismo singleton que usa el VM, así que el overlay de la página reacciona igual. | `Pages/MainPage.xaml.cs:11-17`, `MauiProgram.cs:34` |
| 18 | ⚠️ «Pedir permiso» del overlay llama `CapturarAsync()` y **descarta** el `GpsResult`: aunque el reintento obtenga la ubicación, el `Label` `Coordenadas` sigue mostrando «Permiso de ubicación necesario.» hasta volver a tocar «Obtener ubicación». | `Services/GpsCoordinator.cs:25`, `ViewModels/GpsOverlayViewModel.cs:12`, `ViewModels/MainPageViewModel.cs:88` |
| 19 | `CapturarAsync` cancela y hace `Dispose` del `_cts` previo (`:30-31`) y su `finally` pone `_cts = null` (`:48-49`): con dos capturas solapadas, la primera en terminar anula el `_cts` de la segunda y `Cancelar()` deja de tener efecto. En la UI actual el solapamiento casi no ocurre porque los botones se ocultan durante la lectura. | `Services/GpsCoordinator.cs:30-32`, `:46-50` |

---

## §11 Diferencias con otras variantes del repo

| Aspecto | `Ejemplo_Maui_GPS` | `Ejemplo_Maui_DirectCall` ([06_Telefonia](06_Telefonia.md)) | App híbrida ([08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md)) |
|---|---|---|---|
| Coordinador singleton dueño del overlay | `GpsCoordinator` | `CallCoordinator` (mismo patrón; `CallOverlayViewModel.cs:6` se declara «Espejo del GpsOverlayViewModel») | consolidado en `LibApp/Devices/GPS/` |
| Overlay | 2 paneles (espera / denegado) | panel con 4 botones (incluye «Reintentar») | ver 08 |
| Base MVVM | propia (`ViewModelBase`, `AsyncRelayCommand`) | idéntica salvo namespace | ver 08 |
| Geocoding | `GoogleMapService` + `ApiKeys` | — | ver 08 |

`AsyncRelayCommand.cs` y `ViewModelBase.cs` de GPS y DirectCall son idénticos salvo el namespace (verificado con `diff`). La duplicación es deliberada (un proyecto autocontenido por técnica).

---

## §12 Preguntas frecuentes

| Pregunta | Sección |
|---|---|
| ¿Por qué el proyecto GPS no compila recién clonado? | §6, §10 #1 |
| ¿Cómo configuro mi API key de Google Geocoding? | §6 |
| ¿Cómo inyecta la key el CI de iOS? | §6, §9 |
| ¿Qué precisión y timeout usa la lectura GPS? | §3.3 paso 6, §10 #5 |
| ¿Cómo se distingue «denegado» de «no volver a preguntar» en Android? | §4, §4.1 |
| ¿Qué estados puede devolver el servicio y qué muestra la UI en cada uno? | §4.1 |
| ¿Cómo dispara el GPS (y el overlay) un caller que no es el VM? | §3.3 (botón «Prueba»), §8 |
| ¿Por qué no aparece el domicilio en su label? | §10 #2 |
| ¿Cómo se cancela una lectura en curso? | §3.3, §10 #3, §10 #19 |
| ¿Por qué tras «Pedir permiso» el texto sigue diciendo que falta permiso? | §10 #18 |
| ¿Qué permisos declara Android y qué claves Info.plist usa iOS? | §7 |
| ¿El CI de GPS corre en cada push? | §9 |
| ¿Qué alternativas de diseño servicio↔overlay se evaluaron? | §8 |
| ¿Dónde está el GPS de la app híbrida? | §11, [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) |

---

## §13 Fuentes

| Archivo | Líneas | Rol |
|---|---|---|
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Ejemplo_Maui_GPS.csproj` | 86 | TFM, ID, NuGets |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/MauiProgram.cs` | 46 | DI, fuentes |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GpsService.cs` | 69 | permisos + lectura |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GpsCoordinator.cs` | 71 | punto único + overlay |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/LocationPermissionService.cs` | 51 | mecánica de permisos |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/LocationPermissionResult.cs` | 11 | enum normalizado |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GpsResult.cs` | 15 | resultado tipado |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GoogleMapService.cs` | 60 | geocodificación inversa |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/ApiKeys.cs.template` | 14 | plantilla de clave |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/TestService.cs` | 11 | ejemplo de consumidor |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/MainPageViewModel.cs` | 116 | VM de página |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/GpsOverlayViewModel.cs` | 109 | VM del overlay |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/AsyncRelayCommand.cs` | 76 | comandos propios |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/ViewModelBase.cs` | 24 | `INotifyPropertyChanged` |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Pages/MainPage.xaml(.cs)` | 36 / 26 | UI y botón «Prueba» |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Controls/GpsStatusOverlayView.xaml` | 39 | overlay |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Platforms/Android/AndroidManifest.xml` | 35 | permisos Android |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Platforms/iOS/Info.plist` | 73 | claves iOS |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Platforms/iOS/Entitlements*.plist` | 10 / 10 | entitlements |
| `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/secret.md` | 115 | manejo de API keys |
| `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/servicio.md` | 129 | alternativas de diseño |
| `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/Readme.md` | 2 | enlace a la doc oficial |
| `.gitignore` | 420-421 | exclusión de `ApiKeys.cs` |
| `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml` | 438 | CI iOS |
| `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat` | 48 | deploy Android |
| `Utilities/end2end/com.ejemplos.devices.gps.yaml` | 35 | flujo Maestro (no aplicado) |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | 56-62 | pertenencia a la solución |
