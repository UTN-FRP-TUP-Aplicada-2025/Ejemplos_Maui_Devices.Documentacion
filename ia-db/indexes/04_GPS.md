# 04 — GPS y geolocalización

> **Propósito:** el patrón «servicio con resultado tipado + coordinador dueño del overlay» aplicado a la ubicación, y la geocodificación inversa con Google.
> **Fuente primaria:** `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/` (1 proyecto, ~700 líneas) y `GPS/Ejemplo_Docs_GPS/` (3 documentos).
> **Índices relacionados:** [00_MASTER-INDEX](00_MASTER-INDEX.md) · [06_Telefonia](06_Telefonia.md) (mismo patrón, otro dispositivo) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (la híbrida lo reimplementa con `IUiDispatcher` y catálogo de errores).

---

## 1. El proyecto

`Ejemplo_Maui_GPS` · `com.ejemplos.devices.gps` · `net10.0-android` (+ iOS en macOS) · sin paquetes fuera de MAUI. Es el **ejemplo de referencia del patrón de captura de dispositivo**: los ejemplos de teléfono ([06](06_Telefonia.md)) y los overlays de la híbrida ([08](08_App-Hibrida-Integrada.md)) son variantes suyas.

## 2. Arquitectura en cuatro capas

```
MainPage.xaml  ── binding ──▶  MainPageViewModel  ──▶  GpsCoordinator (singleton)
                                     │                      │  dueño del overlay y del CTS
GpsStatusOverlayView ◀── binding ──  GpsOverlayViewModel ◀───┘
                                                            ▼
                                                       GpsService  ──▶ LocationPermissionService
                                                            ▼                    ▼
                                                     GpsResult (tipado)   LocationPermissionResult
```

| Capa | Archivo | Responsabilidad |
|------|---------|-----------------|
| Permisos | `Services/LocationPermissionService.cs` (51 l.) | `CheckAsync` / `RequestAsync` / `OpenAppSettings`; normaliza a 4 casos |
| Dispositivo | `Services/GpsService.cs` (69 l.) | Compone permiso + lectura y devuelve `GpsResult`; **nunca lanza** |
| Coordinación | `Services/GpsCoordinator.cs` (71 l.) | Punto único de entrada; dueño del overlay y de la cancelación |
| Presentación | `ViewModels/MainPageViewModel.cs` (116 l.) | Solo texto y comandos |

Registro (`MauiProgram.cs:32-42`): `LocationPermissionService`, `GpsService`, `GpsCoordinator`, `TestService`, `GoogleMapService` como **singleton**; `MainPageViewModel` y `MainPage` como **transient**.

## 3. Los dos tipos de resultado

`GpsResult` (`Services/GpsResult.cs`) — 8 casos, uno por situación que la UI debe distinguir:

| Caso | Se produce cuando |
|------|-------------------|
| `Success(Location)` | Hay coordenada |
| `PermissionDenied(bool CanRetry)` | Denegado; `CanRetry` viene de `ShouldShowRationale` |
| `PermissionRestricted` | Política del dispositivo (MDM, control parental) |
| `GpsDisabled` | `FeatureNotEnabledException` |
| `NotSupported` | `FeatureNotSupportedException` |
| `NoSignal` | La API devolvió `null` |
| `Cancelled` | `OperationCanceledException` |
| `Failure(string)` | Cualquier otra excepción |

`LocationPermissionResult` (enum): `Granted` · `DeniedCanRetry` · `Denied` · `Restricted`. El comentario del archivo explica la asimetría: `ShouldShowRationale` **solo existe en Android** (`true` = denegado sin «no volver a preguntar»); en iOS siempre cae en `Denied`, que fuerza el camino de ajustes.

## 4. La lectura de la ubicación

`GpsService.ObtenerUbicacionAsync` (`:33-68`) usa una **estrategia de dos pasos** para no hacer esperar al usuario:

```csharp
var location = await Geolocation.GetLastKnownLocationAsync();
if (!(location != null && (DateTimeOffset.Now - location.Timestamp) < TimeSpan.FromMinutes(1)))
{
    var req = new GeolocationRequest(GeolocationAccuracy.Medium, TimeSpan.FromSeconds(10));
    location = await Geolocation.GetLocationAsync(req, ct);
}
```

Última conocida si tiene **menos de 1 minuto**; si no, lectura fresca con precisión `Medium` y 10 s de tope. El camino directo (`GeolocationAccuracy.Best` + `DefaultTimeout` de 15 s) quedó **comentado** encima (`:36-41`), junto con la constante `DefaultTimeout` que ya no se usa (`:8`).

## 5. El coordinador

`GpsCoordinator` (`:5-9` del comentario) es el «punto único de entrada para capturar GPS desde cualquier parte de la app»: página, comando, deep link o navegación por URL hacen `await _coord.CapturarAsync()` y el overlay aparece y desaparece solo.

- Es **dueño del `GpsOverlayViewModel`**, que construye pasándole los callbacks (`onRetry: CapturarAsync`, `onOpenSettings: _permissions.OpenAppSettings`). Así los botones del overlay no quedan atados a una página concreta.
- Es **dueño de la cancelación**: cada llamada cancela y reemplaza el `CancellationTokenSource` anterior; con un token externo usa `CreateLinkedTokenSource`.
- Todo cambio de overlay entra por `MainThread.InvokeOnMainThreadAsync`.
- `Aplicar(result)` solo decide **overlay**: muestra permiso denegado o restringido, y en cualquier otro caso oculta. El texto lo decide cada caller.

## 6. El overlay

`ViewModels/GpsOverlayViewModel.cs` (109 l.) + `Controls/GpsStatusOverlayView.xaml`. Dos estados observables (`IsBusy`, `IsDenied`) que se combinan en `IsVisible`, más `Title`, `Message`, `CanRetry`/`MustOpenSettings` y tres comandos (`PedirPermisoCommand`, `AbrirAjustesCommand`, `CerrarOverlayCommand`). Recibe los callbacks por constructor «para mantener el `ContentView` autónomo: sus bindings no necesitan cruzar al VM padre».

El VM principal expone `MostrandoContenido => !Overlay.IsVisible` y se suscribe a `Overlay.PropertyChanged` para reemitirlo (`MainPageViewModel.cs:41-45`).

## 7. Geocodificación inversa: `GoogleMapService`

`Services/GoogleMapService.cs` (60 l.). Llama a `https://maps.googleapis.com/maps/api/geocode/json?latlng=…&key=…&language=es` con `GetFromJsonAsync` y devuelve `results[0].formatted_address`. **Nunca lanza**: todos los caminos de error devuelven un `string` descriptivo.

- Formatea lat/lng con `CultureInfo.InvariantCulture` (si no, en configuraciones regionales con coma decimal la URL sale inválida).
- La clave sale de `ApiKeys.GoogleMaps`, una constante de `Services/ApiKeys.cs`, archivo **ignorado por git** (`.gitignore:421`: `**/Services/ApiKeys.cs`). En el repo solo está `Services/ApiKeys.cs.template`.

⚠️ **El proyecto no compila recién clonado**: hay que copiar `ApiKeys.cs.template` a `ApiKeys.cs` y poner la clave. Es una consecuencia deliberada de la opción 3 de `secret.md`.

## 8. Comandos de la pantalla

| Comando | Qué hace |
|---------|----------|
| `ObtenerUbicacionCommand` | `CapturarAsync` y vuelca el texto de `ActualizarTexto` |
| `CancelarCommand` | `_coord.Cancelar()` |
| `MostrarEnMapaCommand` | Captura y abre `https://maps.google.com/?q=lat,lng` con `Browser.Default.OpenAsync` |
| `ObtenerDominicilioCommand` | Captura y reemplaza el texto por el domicilio geocodificado |

`ActualizarTexto` (`:83-97`) es un `switch` exhaustivo sobre los 8 casos de `GpsResult`: es el ejemplo canónico de por qué el resultado es tipado.

⚠️ **Bug verificable:** el setter de `Domicilio` (`MainPageViewModel.cs:25`) escribe en `_coordenadas`, no en `_domicilio` — copia y pega. La propiedad `Domicilio` nunca cambia de valor; en la práctica no se nota porque `ObtenerDominicilioAsync` escribe el domicilio directamente en `Coordenadas` (`:108`).

## 9. Documentación del dominio (`Ejemplo_Docs_GPS/`)

| Archivo | Contenido |
|---------|-----------|
| `Readme.md` | Una línea: el link a la doc de `Geolocation` de Microsoft |
| `secret.md` | **Cómo manejar API keys sin exponerlas**: archivo ignorado, User Secrets de .NET (`dotnet user-secrets set "GoogleMaps:ApiKey" …`), y ⭐ la opción elegida — clase estática + `.template` con cero fricción; incluye pros y contras de cada una |
| `servicio.md` | **Cómo hacer que un servicio use el GPS y el overlay reaccione**: descarta inyectar el VM en el servicio (invierte la dependencia natural, genera ciclos de DI, acopla a una pantalla) y ordena las alternativas de menos a más acoplada — (1) devolver `GpsResult` y que el VM lo aplique ⭐, (2) `IProgress<GpsStatus>`, … |

`TestService` (`Services/TestService.cs`, 11 líneas) es exactamente la opción 1 de `servicio.md` implementada: envuelve `GpsService` y no conoce la UI. Está registrado en DI pero **no lo usa nadie**: es el ejemplo del documento hecho código.

## 10. Fuentes

| Ruta | Contenido |
|------|-----------|
| `GPS/Ejemplo_Maui_GPS/Services/` | Permisos, lectura, coordinación, geocodificación, `ApiKeys.cs.template` |
| `GPS/Ejemplo_Maui_GPS/ViewModels/` | VM de la página, VM del overlay, `AsyncRelayCommand`, `ViewModelBase` |
| `GPS/Ejemplo_Maui_GPS/Controls/GpsStatusOverlayView.xaml` | El overlay reutilizable |
| `GPS/Ejemplo_Docs_GPS/secret.md` | Estrategias de manejo de claves |
| `GPS/Ejemplo_Docs_GPS/servicio.md` | Cómo acoplar servicios al overlay sin invertir dependencias |
