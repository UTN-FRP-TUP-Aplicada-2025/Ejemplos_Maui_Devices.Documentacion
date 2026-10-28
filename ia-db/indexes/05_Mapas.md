# 05 — Mapas

> **Propósito:** mapa navegable del dominio «Mapas» del repositorio `Ejemplos_Maui_Devices`: el proyecto `Ejemplo_Maui_Mapas` (control `Microsoft.Maui.Controls.Maps.Map`, pines, centrado por región, cambio de tipo de mapa, punto azul del usuario) y la configuración de la clave de Google Maps en Android.
> **Fuente primaria:** `Ejemplos_Devices/Maps/**` (commit `285f6fb`). Contexto secundario: `Ejemplos_Devices/Ejemplos_Devices.slnx`, `README.md`, `.github/workflows/`.
> Rutas relativas a la raíz del repo de código. La lectura de `Geolocation`, los permisos de ubicación con overlay y la geocodificación HTTP se documentan en [04_GPS](04_GPS.md) (este proyecto **no** usa `Geolocation`). La app híbrida se documenta en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## §1 Mapa del dominio

| Proyecto | Ruta | Técnica | Archivos clave | En `.slnx` | Workflow CI |
|---|---|---|---|---|---|
| `Ejemplo_Maui_Mapas` | `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/` | Control `Map` (NuGet `Microsoft.Maui.Controls.Maps`) en code-behind: `MapSpan`, `Pin`, `MapType`, `IsShowingUser` | `Pages/MainPage.xaml`, `Pages/MainPage.xaml.cs`, `MauiProgram.cs`, `Platforms/Android/AndroidManifest.xml`, `Platforms/iOS/Info.plist` | Sí (`Ejemplos_Devices.slnx:74`) | **No** (no hay `cd-ios-*maps*`/`*mapas*` en `.github/workflows/`) |
| `Ejemplo_Docs_Maps` (no es proyecto) | `Ejemplos_Devices/Maps/Ejemplo_Docs_Maps/` | Prosa | `Readme.md` (**vacío**, 0 líneas) | Sí, como archivo (`Ejemplos_Devices.slnx:79`) | — |

Árbol versionado (sin detallar recursos binarios):

```
Ejemplos_Devices/Maps/
├── Ejemplo_Docs_Maps/
│   └── Readme.md                ← vacío
└── Ejemplo_Maui_Mapas/
    ├── Ejemplo_Maui_Mapas.csproj
    ├── Readme.md                ← enlace a la doc oficial + nombre del NuGet
    ├── MauiProgram.cs           ← .UseMauiMaps()
    ├── App.xaml(.cs) · AppShell.xaml(.cs)
    ├── Pages/MainPage.xaml(.cs) ← toda la lógica (code-behind, sin VM ni DI)
    ├── Platforms/Android/ AndroidManifest.xml · MainActivity.cs · MainApplication.cs · Resources/values/colors.xml
    ├── Platforms/iOS/ Info.plist · AppDelegate.cs · Program.cs · Resources/PrivacyInfo.xcprivacy
    ├── Platforms/Windows/ App.xaml(.cs) · Package.appxmanifest · app.manifest
    ├── Properties/launchSettings.json ← perfil «Windows Machine»
    └── Resources/ Fonts/OpenSans-* · Images/dotnet_bot.png · AppIcon · Splash · Raw/AboutAssets.txt · Styles/
```

Es el único proyecto de los dominios GPS/Mapas/Telefonía que trae carpeta `Platforms/Windows/` y TFM de Windows.

---

## §2 Stack y configuración del proyecto

| Aspecto | Valor | Fuente |
|---|---|---|
| TFM | `net10.0-android`; `+net10.0-ios` en macOS; `+net10.0-windows10.0.19041.0` en Windows | `Ejemplo_Maui_Mapas.csproj:4-6` |
| `ApplicationId` | `com.ejemplos.devices.mapas` | `Ejemplo_Maui_Mapas.csproj:17` |
| `ApplicationTitle` | `Mapas` | `Ejemplo_Maui_Mapas.csproj:15` |
| Mínimos | iOS 15.0 · Android 25 · Windows 10.0.17763.0 | `Ejemplo_Maui_Mapas.csproj:24-27` |
| Android | `targetSdkVersion=36`; RIDs arm/arm64/x86/x64; páginas de 16 KB; APK | `Platforms/Android/AndroidManifest.xml:26`, `Ejemplo_Maui_Mapas.csproj:32-49` |
| iOS Release | intérprete + `SdkOnly`; AOT solo `ios-arm64` | `Ejemplo_Maui_Mapas.csproj:53-63` |
| NuGets | `Microsoft.Maui.Controls` `$(MauiVersion)` · `Microsoft.Extensions.Logging.Debug` 10.0.2 · **`Microsoft.Maui.Controls.Maps` 10.0.40** (versión fija, no `$(MauiVersion)`) | `Ejemplo_Maui_Mapas.csproj:77-79` |
| Arranque | `.UseMauiApp<App>().UseMauiMaps()` — sin registro DI propio | `MauiProgram.cs:10-17` |
| Shell | una sola `ShellContent` → `Pages.MainPage` | `AppShell.xaml:10-13` |
| Windows | `WindowsPackageType=None`; `Package.appxmanifest` y `app.manifest` de plantilla (placeholders `$placeholder$`) | `Ejemplo_Maui_Mapas.csproj:22`, `Platforms/Windows/Package.appxmanifest:9-16` |

`Readme.md` del proyecto solo dice el enlace a la doc del control `Map` y el nombre del NuGet (`Ejemplo_Maui_Mapas/Readme.md:2-5`).

---

## §3 Arquitectura y flujo

No hay capa de servicios ni ViewModel: todo vive en `MainPage.xaml.cs`. Es el ejemplo de «code-behind directo» del repo para este control.

```
App → AppShell → MainPage
                   ├─ XAML: 3 botones + <maps:Map x:Name="MyMap">
                   └─ code-behind
                        OnAppearing ─┬─ PedirPermisosAsync()  → IsShowingUser = true | alerta
                                     ├─ CentrarMapa(-31.749788, -60.520532)
                                     └─ AgregarPins()          → 2 Pin + MarkerClicked
```

### §3.1 Secuencia al aparecer la página

| Paso | Qué ocurre | Fuente |
|---|---|---|
| 1 | `async void OnAppearing()` | `Pages/MainPage.xaml.cs:15-22` |
| 2 | `Permissions.CheckStatusAsync<LocationWhenInUse>()`; si no está concedido, `RequestAsync` | `Pages/MainPage.xaml.cs:26-29` |
| 3 | Concedido → `MyMap.IsShowingUser = true` (punto azul); denegado → `DisplayAlertAsync("Permiso denegado", …)` | `Pages/MainPage.xaml.cs:31-34` |
| 4 | Centra con `MapSpan.FromCenterAndRadius(Location, Distance.FromKilometers(2))` + `MoveToRegion` | `Pages/MainPage.xaml.cs:20`, `:45-50` |
| 5 | Agrega dos `Pin` (`PinType.Place`) y suscribe `MarkerClicked` | `Pages/MainPage.xaml.cs:52-77` |

### §3.2 Interacciones

| Control | Handler | Efecto | Fuente |
|---|---|---|---|
| «🗺 Normal» | `OnNormalClicked` | `MyMap.MapType = MapType.Street` | `Pages/MainPage.xaml:11`, `Pages/MainPage.xaml.cs:87` |
| «🛰 Satélite» | `OnSateliteClicked` | `MyMap.MapType = MapType.Satellite` | `Pages/MainPage.xaml:12`, `Pages/MainPage.xaml.cs:89` |
| «📍 Mi lugar» | `OnGoToLocation` | Recentra en **coordenadas fijas** `(-31.749788, -60.520532)`, radio 2 km | `Pages/MainPage.xaml:13`, `Pages/MainPage.xaml.cs:91-97` |
| Tocar un pin | `OnPinClicked` | `DisplayAlertAsync("📍 Pin", Label\nAddress)`; `e.HideInfoWindow = false` (según el comentario, deja el tooltip nativo; se asigna tras el `await`, ver §7 #12) | `Pages/MainPage.xaml.cs:79-85` |

---

## §4 API del control `Map`

| Elemento | Uso en el ejemplo | Fuente |
|---|---|---|
| Namespace XAML | `xmlns:maps="http://schemas.microsoft.com/dotnet/2021/maui/maps"` | `Pages/MainPage.xaml:4` |
| `<maps:Map>` | `MapType="Street"`, `IsShowingUser="False"` (se activa tras el permiso), `IsScrollEnabled="True"`, `IsZoomEnabled="True"` | `Pages/MainPage.xaml:17-20` |
| `MapSpan` (constructor) | Variante comentada `new MapSpan(centro, 0.1, 0.1)` («grados de zoom») centrada en Buenos Aires | `Pages/MainPage.xaml.cs:37-43` |
| `MapSpan.FromCenterAndRadius` + `Distance.FromKilometers` | Variante activa (radio en km) | `Pages/MainPage.xaml.cs:48`, `:93-95` |
| `Map.MoveToRegion(MapSpan)` | Centrado | `Pages/MainPage.xaml.cs:49`, `:96` |
| `Pin { Label, Address, Location, Type }` | Dos pines `PinType.Place` | `Pages/MainPage.xaml.cs:56-69` |
| `Map.Pins.Add` | Alta de pines | `Pages/MainPage.xaml.cs:75` |
| `Pin.MarkerClicked` / `PinClickedEventArgs.HideInfoWindow` | Evento de toque | `Pages/MainPage.xaml.cs:74`, `:79-85` |
| `MapType.Street` / `MapType.Satellite` | Cambio de tipo | `Pages/MainPage.xaml.cs:87-89` |
| `using` | `Microsoft.Maui.Controls.Maps` (Pin, Map) y `Microsoft.Maui.Maps` (MapSpan, Distance, MapType) | `Pages/MainPage.xaml.cs:1-2` |

### §4.1 Datos de ejemplo

| Elemento | Coordenadas | Texto | Fuente |
|---|---|---|---|
| Centro inicial y «Mi lugar» | `-31.749788, -60.520532` (zona de Paraná, Entre Ríos — ubicación no verificada en el repo) | — | `Pages/MainPage.xaml.cs:20`, `:93` |
| Pin «Obelisco» | `-34.6037, -58.3816` | `Address = "Calle 25"` | `Pages/MainPage.xaml.cs:56-62` |
| Pin «Casa Rosada» | `-34.6083, -58.3712` | `Address = "Calle 33"` | `Pages/MainPage.xaml.cs:63-69` |
| Centro comentado («Buenos Aires») | `-34.6037, -58.3816` | — | `Pages/MainPage.xaml.cs:39-40` |

Las direcciones «Calle 25» y «Calle 33» son textos de relleno, no direcciones reales de esos lugares.

---

## §5 Clave de Google Maps y permisos por plataforma

### §5.1 Android (`Platforms/Android/AndroidManifest.xml`)

| Elemento | Valor | Línea |
|---|---|---|
| `meta-data com.google.android.geo.API_KEY` | **Clave literal versionada** (forma `AIza…`; no se reproduce aquí) | `:9-11` |
| `meta-data com.google.android.gms.version` | `@integer/google_play_services_version` | `:13-14` |
| `ACCESS_NETWORK_STATE`, `INTERNET` | declarados | `:17-18` |
| `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION` | declarados («opcionales si querés mostrar la ubicación del usuario») | `:22-24` |
| `uses-sdk` | min 25 · target 36 | `:26` |

El control `Map` en Android se apoya en Google Maps, que exige la clave en ese `meta-data` (documentación oficial enlazada en `Ejemplo_Maui_Mapas/Readme.md:2`; requisito no verificado más allá del manifiesto).

### §5.2 iOS (`Platforms/iOS/Info.plist`)

| Clave | Valor | Línea |
|---|---|---|
| `NSLocationWhenInUseUsageDescription` | «Necesitamos tu ubicación para mostrarte en el mapa.» | `:32-33` |
| `LSRequiresIPhoneOS`, `UIDeviceFamily` 1/2, `arm64` | plantilla | `:5-15` |
| `CFBundleIdentifier`, `MinimumOSVersion`, versiones | **ausentes** del plist (los toma del `.csproj`; a diferencia de GPS y Telefonía, que los repiten en el plist) | `Platforms/iOS/Info.plist:1-35` |
| Clave de mapas | no aplica: no se configura ninguna clave para iOS | — |

No hay `Entitlements.plist` en este proyecto (sí los hay en GPS, Dialer y DirectCall).

### §5.3 Windows

`Platforms/Windows/` es la plantilla estándar de MAUI (`App.xaml.cs:6-23`, `Package.appxmanifest`, `app.manifest`); `Properties/launchSettings.json:3-6` trae el perfil `"Windows Machine"` de la plantilla. No se configura token de mapas para Windows. Si el control `Map` funciona en Windows con este NuGet no está verificado en el repo.

---

## §6 Build, ejecución y CI

| Vía | Estado | Fuente |
|---|---|---|
| Solución | Proyecto incluido con `<Deploy />` | `Ejemplos_Devices/Ejemplos_Devices.slnx:73-76` |
| CI iOS | **No existe** workflow para Mapas | `.github/workflows/` (inventario: GPS, Phone×2, Camera, QR, Printer, Integrada) |
| Script `.bat` | No hay `Ejemplo_Maui_Mapas_launch.bat` (en `scripts/` solo GPS, MediaPicker y `GetEnviromentVersion`) | `Ejemplos_Devices/Ejemplos_Devices.slnx:197-199` |
| Clave en build | No hace falta `ApiKeys.cs`: la clave está en el manifiesto, así que el proyecto compila recién clonado | `Platforms/Android/AndroidManifest.xml:10-11` |

---

## §7 Trampas y discrepancias ⚠️

| # | Trampa / discrepancia | Evidencia |
|---|---|---|
| 1 | ⚠️ **API key de Google versionada en texto plano** en el manifiesto Android. Contradice la regla de la propia cátedra («**Nunca** poner keys reales en archivos versionados», opción 3 con `ApiKeys.cs` ignorado). Si se considera filtrada, el doc indica revocarla y generar otra, porque borrar el commit no alcanza. | `Platforms/Android/AndroidManifest.xml:10-11` vs `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/secret.md:114-115`, `:71-78` |
| 2 | ⚠️ **«📍 Mi lugar» no va a la ubicación del usuario**: recentra en coordenadas fijas (las mismas del centrado inicial). La posición real solo se ve como punto azul (`IsShowingUser`). | `Pages/MainPage.xaml:13`, `Pages/MainPage.xaml.cs:91-97` |
| 3 | ⚠️ **Los pines no se ven al iniciar**: están en Buenos Aires (~-34.6) y el mapa arranca centrado en `-31.75, -60.52` con radio 2 km; hay que desplazarse para encontrarlos. | `Pages/MainPage.xaml.cs:20`, `:60`, `:67` |
| 4 | `OnAppearing` vuelve a agregar los pines cada vez que la página aparece (no limpia `MyMap.Pins`). Con una sola página en el Shell rara vez se nota. | `Pages/MainPage.xaml.cs:15-22`, `:72-76` |
| 5 | `OnAppearing` y `OnPinClicked` son `async void` sin `try/catch`: una excepción del permiso o del alert no queda contenida. | `Pages/MainPage.xaml.cs:15`, `:79` |
| 6 | Los permisos de ubicación se piden sin distinguir Android «no volver a preguntar», restringido de iOS, etc.: solo «concedido o alerta». El manejo completo está en [04_GPS](04_GPS.md) §4. | `Pages/MainPage.xaml.cs:24-35` |
| 7 | `Microsoft.Maui.Controls.Maps` fijado en 10.0.40, mientras `Microsoft.Maui.Controls` sigue `$(MauiVersion)`: si el workload trae otra versión de MAUI, quedan desalineados. | `Ejemplo_Maui_Mapas.csproj:77-79` |
| 8 | `Ejemplo_Docs_Maps/Readme.md` está vacío, pero el README raíz lo enlaza como documentación de Maps («En construcción…»). | `Ejemplos_Devices/Maps/Ejemplo_Docs_Maps/Readme.md` (0 líneas), `README.md:23-24` |
| 9 | En `.slnx`, la carpeta de documentación de Printer referencia `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md`: el nombre «Docs_Maps» dentro de Printer no corresponde a este dominio (ruta no versionada en `Maps/`). | `Ejemplos_Devices/Ejemplos_Devices.slnx:111` |
| 10 | `MainPage.xaml` no define `Title`; el Shell muestra «Mapas» (título del Shell) / «Home» (título del `ShellContent`). | `Pages/MainPage.xaml:2-5`, `AppShell.xaml:8-11` |
| 11 | El proyecto declara TFM de Windows y carpeta `Platforms/Windows/`, cosa que ningún otro ejemplo de GPS/Telefonía hace; el soporte real del control en Windows no está verificado. | `Ejemplo_Maui_Mapas.csproj:6`, `Platforms/Windows/` |
| 12 | ⚠️ **`e.HideInfoWindow = false` se asigna después del `await`**: `OnPinClicked` es `async void`, así que el handler devuelve el control al disparador en el primer `await DisplayAlertAsync(...)` y la asignación llega cuando el evento ya se procesó. El comentario («muestra el tooltip nativo») sugiere que esa línea controla el tooltip; en la práctica no influye (además `false` es el valor por defecto de la propiedad — no verificado en el repo). Para que tenga efecto debería ir antes del `await`. | `Pages/MainPage.xaml.cs:79-85` |

---

## §8 Diferencias con variantes relacionadas

| Aspecto | `Ejemplo_Maui_Mapas` | `Ejemplo_Maui_GPS` ([04_GPS](04_GPS.md)) |
|---|---|---|
| Qué muestra | Mapa nativo embebido (`Map`) | Coordenadas en texto; abre Google Maps **web** por `Browser.OpenAsync` |
| Lectura de ubicación | No usa `Geolocation`; solo `IsShowingUser` | `Geolocation.GetLastKnownLocationAsync` / `GetLocationAsync` |
| Permisos | `CheckStatus`/`Request` inline + alerta | `LocationPermissionService` + overlay MVVM |
| Clave de Google | Literal en `AndroidManifest.xml` (Maps SDK Android) | `ApiKeys.cs` ignorado + `.template` (Geocoding HTTP) |
| Arquitectura | Code-behind | Servicio tipado + coordinador + overlay |
| NuGet extra | `Microsoft.Maui.Controls.Maps` 10.0.40 | ninguno |
| CI | ninguno | `cd-ios-gps.Ejemplo_GPS.yml` |

---

## §9 Preguntas frecuentes

| Pregunta | Sección |
|---|---|
| ¿Qué NuGet y qué llamada de arranque necesita el control `Map`? | §2, §4 |
| ¿Dónde se configura la clave de Google Maps? | §5.1, §7 #1 |
| ¿Por qué Mapas compila recién clonado y GPS no? | §6, §8 |
| ¿Cómo centro el mapa en una coordenada con un radio? | §3.1, §4 |
| ¿Cómo agrego pines y reacciono al toque? | §3.2, §4 |
| ¿Cómo cambio entre mapa normal y satelital? | §3.2 |
| ¿Por qué «Mi lugar» no me lleva a mi posición? | §7 #2 |
| ¿Por qué no veo los pines al abrir la app? | §7 #3 |
| ¿Qué permisos y claves de Info.plist usa? | §5 |
| ¿Hay CI para Mapas? | §6 |
| ¿En qué se diferencia del ejemplo GPS? | §8 |

---

## §10 Fuentes

| Archivo | Líneas | Rol |
|---|---|---|
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Ejemplo_Maui_Mapas.csproj` | 82 | TFM, ID, NuGets |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/MauiProgram.cs` | 25 | `UseMauiMaps()` |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Pages/MainPage.xaml` | 23 | control `Map` y botones |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Pages/MainPage.xaml.cs` | 98 | permisos, centrado, pines, tipos |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/AppShell.xaml` | 15 | navegación |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Platforms/Android/AndroidManifest.xml` | 27 | clave y permisos Android |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Platforms/iOS/Info.plist` | 35 | claves iOS |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Platforms/Windows/*` | 8 / 25 / 46 / 17 | plantilla Windows |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Properties/launchSettings.json` | 8 | perfil «Windows Machine» |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Readme.md` | 4 | enlace a doc + NuGet |
| `Ejemplos_Devices/Maps/Ejemplo_Docs_Maps/Readme.md` | 0 | vacío |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | 73-79, 111 | pertenencia a la solución |
| `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/secret.md` | 71-78, 114-115 | política de claves de la cátedra |
| `README.md` | 23-24 | enlace a docs de Maps |
