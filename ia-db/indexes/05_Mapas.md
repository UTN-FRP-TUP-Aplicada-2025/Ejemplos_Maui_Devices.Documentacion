# 05 — Mapas

> **Propósito:** el control `Map` de MAUI: permisos, centrado, pines y tipos de mapa; y la clave de Google Maps que este dominio necesita.
> **Fuente primaria:** `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/` (1 proyecto, ~100 líneas de código propio).
> **Índices relacionados:** [00_MASTER-INDEX](00_MASTER-INDEX.md) · [04_GPS](04_GPS.md) (usa la misma clave de Google, y abre el mapa por navegador en vez de por control).

---

## 1. El proyecto

`Ejemplo_Maui_Mapas` · `com.ejemplos.devices.mapas` · paquete propio: **`Microsoft.Maui.Controls.Maps` 10.0.40**.

Es el **único ejemplo del repositorio que declara también el TFM de Windows**: `net10.0-windows10.0.19041.0` (condicionado a compilar en Windows), además de Android e iOS. Por eso tiene `Platforms/Windows/App.xaml.cs`.

Arranque: `.UseMauiMaps()` en `MauiProgram.cs:12` — sin él, el control no resuelve su handler.

## 2. Lo que demuestra

Todo vive en `Pages/MainPage.xaml{,.cs}`. El XAML declara `<maps:Map x:Name="MyMap" MapType="Street">` y tres botones; el code-behind resuelve cuatro cosas:

| Tema | Método | Detalle |
|------|--------|---------|
| Permiso + punto azul | `PedirPermisosAsync()` (`:24-35`) | `LocationWhenInUse`; si se concede, `MyMap.IsShowingUser = true`; si no, un `DisplayAlert` |
| Centrar y hacer zoom | `CentrarMapa(lat, lon, radioKm = 2)` (`:45-50`) | `MapSpan.FromCenterAndRadius(ubicacion, Distance.FromKilometers(radioKm))` + `MoveToRegion` |
| Pines | `AgregarPins()` (`:52-77`) | `Pin { Label, Address, Location, Type = PinType.Place }`, suscripción a `MarkerClicked` y `MyMap.Pins.Add` |
| Tipo de mapa | `OnNormalClicked` / `OnSateliteClicked` (`:87-89`) | `MyMap.MapType = MapType.Street | MapType.Satellite` |

Secuencia de arranque (`OnAppearing`, `:15-22`): pedir permisos → centrar en **Paraná, Entre Ríos** (`-31.749788, -60.520532`) → agregar pines.

Los dos pines son de Buenos Aires (Obelisco y Casa Rosada) con direcciones de relleno («Calle 25», «Calle 33»), así que quedan **fuera del área centrada**: hay que alejar el zoom para verlos. El botón «📍 Mi lugar» vuelve a Paraná.

`OnPinClicked` muestra un `DisplayAlert` y deja `e.HideInfoWindow = false` para que además aparezca el tooltip nativo del mapa.

Una alternativa de centrado por `MapSpan(centro, 0.1, 0.1)` — zoom en **grados** en vez de kilómetros — quedó comentada (`:37-43`).

## 3. Configuración de Android

`Platforms/Android/AndroidManifest.xml` declara:

- Permisos `INTERNET`, `ACCESS_NETWORK_STATE`, `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`.
- `<meta-data android:name="com.google.android.geo.API_KEY" android:value="AIza…" />` y `com.google.android.gms.version`.

⚠️ **La clave de Google Maps está escrita en el manifiesto y versionada.** Es exactamente lo que `GPS/Ejemplo_Docs_GPS/secret.md` recomienda evitar ([índice 04 §7](04_GPS.md)): en el dominio GPS la clave se aisló en `Services/ApiKeys.cs` (ignorado por git) y acá quedó en claro dentro del `AndroidManifest.xml`. Si la clave se rota, hay que editar el manifiesto.

## 4. Observaciones

- **Los dos Readme del dominio están invertidos**: `Maps/Ejemplo_Docs_Maps/Readme.md` —la carpeta de documentación— está **vacío (0 bytes)**, y el contenido (link a la doc del control `Map` y el nombre del NuGet) vive en `Maps/Ejemplo_Maui_Mapas/Readme.md`.
- No hay servicios, ViewModels ni overlay: es el ejemplo más directo del repositorio, todo en code-behind.
- **No tiene workflow de CI** ([índice 09](09_CI-CD-y-Build.md)).

## 5. Fuentes

| Ruta | Contenido |
|------|-----------|
| `Maps/Ejemplo_Maui_Mapas/Pages/MainPage.xaml{,.cs}` | El control, los pines, el centrado y el cambio de tipo de mapa |
| `Maps/Ejemplo_Maui_Mapas/MauiProgram.cs` | `.UseMauiMaps()` |
| `Maps/Ejemplo_Maui_Mapas/Platforms/Android/AndroidManifest.xml` | Permisos y clave de Google Maps |
| `Maps/Ejemplo_Maui_Mapas/Readme.md` | Link a la doc de Microsoft y NuGet requerido |
