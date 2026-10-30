# 04 — GPS (geolocalización y geocodificación inversa)

> **Propósito:** cómo `Ejemplo_Maui_GPS` obtiene la ubicación, resuelve permisos, traduce coordenadas a domicilio con Google Geocoding, maneja la API key fuera del repo y muestra un overlay de espera/permisos. Indexado sobre el HEAD `cc80ce1`.
> **Fuentes primarias:** `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/` · `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/` · `.gitignore`.
> La versión GPS de la app híbrida (`LibApp/Devices/GPS/`, catálogo `GPS-*`, ADR-0009) va en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md); el workflow `cd-ios-gps.Ejemplo_GPS.yml` en [09_CI-CD-y-Build](09_CI-CD-y-Build.md).

## 1. Proyecto

| Dato | Valor | Cita |
|---|---|---|
| TFM | `net10.0-android` (+ `net10.0-ios` sólo en macOS) | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Ejemplo_Maui_GPS.csproj:4-5` |
| ApplicationId | `com.ejemplos.devices.gps` | `…/Ejemplo_Maui_GPS.csproj:18` |
| Min OS | iOS 15 / Android 25 | `…/Ejemplo_Maui_GPS.csproj:25-26` |
| NuGet | `Microsoft.Maui.Controls` `$(MauiVersion)`, `Microsoft.Extensions.Logging.Debug` 10.0.2 (sin CommunityToolkit) | `…/Ejemplo_Maui_GPS.csproj:82-83` |
| En `.slnx` | sí | `Ejemplos_Devices/Ejemplos_Devices.slnx:57` |
| DI | Singletons `LocationPermissionService`, `GpsService`, `GpsCoordinator`, `TestService`, `GoogleMapService`; transient `MainPageViewModel`, `MainPage` | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/MauiProgram.cs:29-45` |
| Script local | `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat` (build + deploy Android por adb) | `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat:5-7` |

⚠️ El `.slnx` referencia `GPS/Docs_GPS/Readme.md` (`Ejemplos_Devices/Ejemplos_Devices.slnx:62`), ruta inexistente; la carpeta real es `GPS/Ejemplo_Docs_GPS/`.

```
MainPage ─BindingContext→ MainPageViewModel ──► GpsCoordinator (singleton, dueño del overlay y del CTS)
   │  GpsStatusOverlayView ◄─BindingContext─ Overlay (GpsOverlayViewModel)
   │                                              │
   │                                      GpsService.ObtenerUbicacionAsync → GpsResult
   │                                              └─ LocationPermissionService → LocationPermissionResult
   └─ ObtenerDominicilio → GoogleMapService.GetDomicilioAsync(lat,lng) → string
```

## 2. Obtención de la ubicación

`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GpsService.cs`:

| Paso | Detalle | Cita |
|---|---|---|
| 1. Permisos | `RequestAsync()` → `Restricted` / `DeniedCanRetry` / `Denied` cortan con `GpsResult` | `:18-30` |
| 2. Caché | `Geolocation.GetLastKnownLocationAsync()`; se acepta si tiene < 1 min | `:43-44` |
| 3. Fix fresco | si no: `GeolocationRequest(Medium, 10 s)` + `GetLocationAsync(req, ct)` | `:46-47` |
| 4. Resultado | `null` → `NoSignal`; si no `Success(location)` | `:50` |
| Excepciones → resultado | `OperationCanceled`→`Cancelled`, `FeatureNotEnabled`→`GpsDisabled`, `FeatureNotSupported`→`NotSupported`, resto→`Failure(msg)` | `:52-67` |

`GpsResult`: `Success(Location)`, `PermissionDenied(CanRetry)`, `PermissionRestricted`, `GpsDisabled`, `NotSupported`, `NoSignal`, `Cancelled`, `Failure(Message)` — `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GpsResult.cs:5-15`.

⚠️ La versión con `GeolocationAccuracy.Best` y `DefaultTimeout` (15 s) quedó comentada (`…/GpsService.cs:8`, `:36-41`); `DefaultTimeout` no se usa.
⚠️ Este ejemplo **conserva** el atajo de caché que [ADR-0009](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0009-ciclo-vida-overlay-gps-y-cancelacion-navegacion.md) (D3) eliminó en la híbrida por hacer parpadear el overlay; allí se lee siempre fresco (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/GPS/Services/GpsService.cs:43`). Divergencia deliberada o pendiente: no verificado.

### Coordinador y cancelación

`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GpsCoordinator.cs`:
- Crea el `GpsOverlayViewModel` con callbacks `onRetry: CapturarAsync`, `onOpenSettings: OpenAppSettings` (`:25`).
- `CapturarAsync`: cancela y descarta el CTS previo, crea uno (enlazado si viene `ct`), `ShowBusy` en el hilo principal, obtiene, aplica y devuelve el resultado (`:28-51`). `Cancelar()` → `_cts?.Cancel()` (`:53`).
- `Aplicar`: sólo `PermissionDenied` y `PermissionRestricted` dejan el overlay visible; todo lo demás `Hide()` y el texto lo decide el caller (`:56-70`).

⚠️ El botón «Cancelar» está dentro del contenido que se oculta mientras el overlay es visible (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Pages/MainPage.xaml:10`, `:16`; `MostrandoContenido => !Overlay.IsVisible` en `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/MainPageViewModel.cs:28`) y el panel de espera no tiene botón propio (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Controls/GpsStatusOverlayView.xaml:18-22`): durante la búsqueda no hay forma de cancelar desde la UI.
⚠️ El `AsyncRelayCommand` propio siempre pasa `CancellationToken.None` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/AsyncRelayCommand.cs:47`); la cancelación real depende sólo del CTS del coordinador.

## 3. Permisos

| Plataforma | Declaración | Cita |
|---|---|---|
| Android | `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`; `ACCESS_BACKGROUND_LOCATION` comentado; `uses-feature location.gps required=false`; `uses-sdk 25/36` | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Platforms/Android/AndroidManifest.xml:14`, `:20`, `:26`, `:32`, `:34` |
| iOS | `NSLocationWhenInUseUsageDescription` y `NSLocationAlwaysAndWhenInUseUsageDescription` activas; una variante «Always» comentada | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Platforms/iOS/Info.plist:64-67`, `:71` |

Runtime — `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/LocationPermissionService.cs` con `Permissions.LocationWhenInUse`:

| `PermissionStatus` tras `RequestAsync` | `LocationPermissionResult` | Overlay |
|---|---|---|
| Granted | `Granted` | — |
| Restricted (MDM/control parental) | `Restricted` | «Acceso restringido» + Abrir configuración |
| Denied y `ShouldShowRationale` = true (sólo Android) | `DeniedCanRetry` | «Permiso de ubicación necesario» + Pedir permiso |
| Denied otro caso / iOS | `Denied` | «Acceso a la ubicación denegado» + Abrir configuración |

Citas: `:13-41` (rationale `#if ANDROID` en `:35-37`), `:43` (`AppInfo.ShowSettingsUI`); enum `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/LocationPermissionResult.cs:5-11`.
⚠️ `CheckAsync` mapea todo no-concedido a `DeniedCanRetry` (`puedeReintentar: true`, `…/LocationPermissionService.cs:7-11`); no lo usa nadie en el flujo actual.
⚠️ El comentario «Permisos necesarios para la captura de medios» (`…/Platforms/iOS/Info.plist:56`) es copia de otro dominio.

## 4. Geocodificación inversa

`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GoogleMapService.cs`:
- Endpoint `https://maps.googleapis.com/maps/api/geocode/json` (`:12`), `HttpClient` estático (`:14`).
- URL `?latlng={lat},{lng}&key={ApiKey}&language=es` con coordenadas en `InvariantCulture` (`:20-22`).
- `GetFromJsonAsync<GeocodeResponse>`; `status != "OK"` → `"Error: {status} - {error_message}"`; si hay resultados → `results[0].formatted_address` (`:24-35`, DTOs `:43-59`).
- Nunca lanza: todo error vuelve como `string` (`:37-40`), no como resultado tipado.

⚠️ **Bug**: el setter de `Domicilio` escribe en `_coordenadas` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/MainPageViewModel.cs:25`) y `ObtenerDominicilioAsync` asigna el domicilio a `Coordenadas` (`:108`); el `Label` de `Domicilio` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Pages/MainPage.xaml:25`) queda vacío. Typo `Dominicilio` coherente entre VM y XAML (`…/MainPageViewModel.cs:99`, `…/MainPage.xaml:26`).
⚠️ «Mostrar en mapa» arma `https://maps.google.com/?q={Latitude},{Longitude}` con la cultura del dispositivo (`…/MainPageViewModel.cs:68`): en `es-*` el decimal sería coma — mismo defecto que ADR-0009 D2 corrigió en la híbrida (efecto en dispositivo: no verificado).

## 5. API key de Google (sin valores)

| Pieza | Detalle | Cita |
|---|---|---|
| Consumo | `private const string ApiKey = ApiKeys.GoogleMaps;` | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GoogleMapService.cs:11` |
| Plantilla versionada | `ApiKeys.cs.template`: `internal static class ApiKeys { const string GoogleMaps = "<placeholder>" }` + instrucciones | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/ApiKeys.cs.template:1-14` |
| Ignorado | `**/Services/ApiKeys.cs` | `.gitignore:421` |
| CI | genera `ApiKeys.cs` desde la plantilla con `secrets.GOOGLE_MAPS_API_KEY` → ver [09](09_CI-CD-y-Build.md) | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:279-285` |
| Alternativas evaluadas | 5 opciones (archivo ignorado, User Secrets, clase+template ⭐, SecureStorage, backend) y buenas prácticas (restringir key, cuotas, rotar) | `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/secret.md:7-115` |

**Por qué no compila recién clonado:** `ApiKeys.cs` no viene en el clone (ignorado) y el `.template` no es un `.cs`, así que `ApiKeys.GoogleMaps` no existe y el build falla al compilar `GoogleMapService` (error de símbolo inexistente; mensaje exacto no verificado). Arreglo: copiar `Services/ApiKeys.cs.template` a `Services/ApiKeys.cs` y poner una key propia (`Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/secret.md:99-106`). Es el único proyecto de esta carpeta con ese requisito.
La key viaja en query string hacia Google (`…/GoogleMapService.cs:22`).

## 6. Overlay

`GpsOverlayViewModel` (primera generación del patrón: `ViewModelBase` y comandos propios, sin CommunityToolkit) — `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/GpsOverlayViewModel.cs`:

| Estado | Propiedades | API | Cita |
|---|---|---|---|
| Oculto | `IsBusy=false, IsDenied=false` | `Hide()` | `:104-108` |
| Espera | `IsBusy=true` | `ShowBusy()` | `:76-80` |
| Permiso denegado | `IsDenied=true`, `CanRetry` ⇒ `MustOpenSettings = !CanRetry` | `ShowPermissionDenied(canRetry)` | `:82-93`, `:63` |
| Restringido | `IsDenied=true`, `CanRetry=false` | `ShowRestricted()` | `:95-102` |
| Visibilidad | `IsVisible => IsBusy || IsDenied` | — | `:66` |

Comandos `PedirPermiso` (callback retry), `AbrirAjustes`, `CerrarOverlay` (`:10-15`). Vista `GpsStatusOverlayView`: espera con `satelite.gif` y textos fijos; panel de denegado con ícono `location_off` (`MaterialIconsOutlined`) y botones condicionados por `CanRetry` / `MustOpenSettings` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Controls/GpsStatusOverlayView.xaml:18-37`); montada con `BindingContext="{Binding Overlay}"` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Pages/MainPage.xaml:32`).
Contraste: la generación con base genérica `StatusOverlayViewModel` (None/Busy/Error + botonera dinámica) está en Impresión — [03_Impresion-Termica](03_Impresion-Termica.md) §7.
⚠️ `GpsDisabled`, `NoSignal`, `Failure` no tienen pantalla de overlay: sólo texto en el VM (`…/MainPageViewModel.cs:83-97`).

### Callers fuera del VM
- Botón «Prueba» llama al coordinador desde code-behind (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Pages/MainPage.xaml.cs:21-25`).
- `TestService` (registrado, `MauiProgram.cs:35`) sólo delega en `GpsService` y nadie lo consume (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/TestService.cs:3-10`); es el caso de estudio de `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/servicio.md:1-129` (5 alternativas para que un servicio dispare el overlay; recomienda la 1).

⚠️ `Utilities/end2end/com.ejemplos.devices.gps.yaml` toca «Geo Posicionar», «Llamar», «Leer QR» (`:20`, `:24`, `:30`): textos que no existen en `Ejemplo_Maui_GPS` (corresponden a la híbrida). Ver [09](09_CI-CD-y-Build.md).

## 7. Prosa relacionada

| Enlace | Qué aporta |
|---|---|
| [pieces/gps/README.md](../../Ejemplos_Maui_Devices-docs/docs/pieces/gps/README.md) | Ficha de la pieza, snippets canónicos y observaciones |
| [ADR-0007](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0007-secretos-fuera-del-repo.md) | Decisión plantilla + secret de CI para claves |
| [ADR-0009](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0009-ciclo-vida-overlay-gps-y-cancelacion-navegacion.md) | Cierre del overlay GPS y cultura invariante (aplicado en la híbrida) |
| [ADR-0002](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0002-servicio-tipado-overlay-mvvm.md) | Patrón servicio → resultado tipado → overlay |
| [07-operations/build-and-run.md](../../Ejemplos_Maui_Devices-docs/docs/07-operations/build-and-run.md) | Paso manual de `ApiKeys.cs` en el runbook |
| [01-architecture/07-overlays-dispositivos.md](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/07-overlays-dispositivos.md) | Las dos generaciones del overlay |

## 8. Fuentes

| Ruta | Contenido |
|---|---|
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/*.cs` | Permisos, GPS, coordinador, geocoding, resultados |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/ApiKeys.cs.template` | Plantilla de la key |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/*.cs` | VM principal, overlay, comandos propios |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Controls/GpsStatusOverlayView.xaml`, `Pages/MainPage.xaml(.cs)` | UI |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Platforms/{Android/AndroidManifest.xml,iOS/Info.plist}` | Permisos declarados |
| `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/{Readme,secret,servicio}.md` | Enlace a docs MS; manejo de secretos; alternativas de servicio |
| `.gitignore:421` | Exclusión de `ApiKeys.cs` |
