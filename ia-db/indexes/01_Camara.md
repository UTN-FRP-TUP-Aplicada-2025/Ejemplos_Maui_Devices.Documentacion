# 01 — Cámara

> **Propósito**: destilar el dominio «Cámara» del repositorio didáctico `Ejemplos_Maui_Devices`: los cinco proyectos MAUI de foto/selfie y la documentación de `Ejemplo_Docs_Photo`. Cubre `MediaPicker` nativo, la página propia de captura con `CameraView` (CommunityToolkit) y sus dos formas de devolver el resultado (callback y `Task`), la normalización EXIF con SkiaSharp y la variante selfie.
> **Fuente primaria**: `Ejemplos_Devices/Camera/**` (commit `285f6fb`). Contexto: `Ejemplos_Devices/Ejemplos_Devices.slnx` y `.github/workflows/cd-ios-camera.*.yml`.
> **Índices hermanos**: la reutilización de estas páginas dentro de la app integradora (`LibApp/Devices/Camera/`) se documenta en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md); la visión general y el catálogo de índices, en [00_MASTER-INDEX](00_MASTER-INDEX.md).

---

## §1 Alcance y alias de rutas

Todas las rutas son relativas a la raíz del repo de código. Para abreviar se usan estos alias:

| Alias | Ruta |
|---|---|
| `CAM/` | `Ejemplos_Devices/Camera/` |
| `MP/` | `Ejemplos_Devices/Camera/Ejemplo_Photo_MediaPicker/` |
| `CB/` | `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Callback/` |
| `CBN/` | `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/` |
| `TK/` | `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Task/` |
| `SLF/` | `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/` |
| `DOC/` | `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/` |
| `WF/` | `.github/workflows/` |

Fuera de alcance: `Platforms/*/Resources`, imágenes, fuentes y `bin/obj`. Las plantillas MAUI estándar (`App.xaml`, `Resources/Styles/*.xaml`, `Program.cs`, `AppDelegate.cs`, `MainApplication.cs`) no se detallan porque no aportan al dominio.

---

## §2 Mapa de proyectos

| # | Proyecto | Técnica | Archivos clave | ¿En `.slnx`? | ¿Workflow CI? |
|---|---|---|---|---|---|
| 1 | `Ejemplo_Photo_MediaPicker` | Diálogo nativo `MediaPicker.CapturePhotoAsync` envuelto en un servicio (`ICamaraService`) | `MP/Services/CameraService.cs`, `MP/Pages/MainPage.xaml.cs`, `MP/MEDIAPICKER_FIX.md` | Sí | **No** |
| 2 | `Ejemplo_Photo_MiMediaPicker_Callback` | Página propia con `CameraView` creada por código; devuelve un **path** por `Action<string?>` vía `ShellNavigationQueryParameters` | `CB/Pages/MyMediaPickerPage.xaml.cs`, `CB/Pages/MainPage.xaml.cs` | Sí | Sí (`WF/cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback.yml`) |
| 3 | `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` | Igual que #2 + servicio DI `IImageService` que corrige EXIF, redimensiona y recomprime (SkiaSharp + MetadataExtractor) | `CBN/Services/ImageDeviceAutoRotateService.cs`, `CBN/Pages/MainPage.xaml.cs`, `CBN/README.md` | Sí | Sí (`WF/cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.yml`) |
| 4 | `Ejemplo_Photo_MiMediaPicker_Task` | Página propia con `CameraView` declarada en XAML; devuelve una `Image` por `TaskCompletionSource<Image>` con `Navigation.PushAsync/PopAsync` | `TK/Pages/MyMediaPickerPage.xaml.cs`, `TK/Pages/MainPage.xaml.cs` | Sí | Sí (`WF/cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Task.yml`) |
| 5 | `Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | Variante selfie de #3: cámara frontal, máscara oval (`GraphicsView` + `IDrawable`), normalizador instanciado con `new` | `SLF/Pages/MyMediaSelfiePickerPage.xaml.cs`, `SLF/Utilities/SelfieMaskDrawable.cs`, `SLF/Utilities/ImageDeviceAutoRotate.cs` | Sí | Sí (`WF/cd-ios-camera.Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion.yml`) |
| — | `Ejemplo_Docs_Photo` | Sólo documentación (no es proyecto) | `DOC/Readme.md`, `DOC/Transferencia_Foto_Entre_Pantallas.md` | Sólo `Readme.md` como `<File>` | No aplica |

- Los cinco `.csproj` figuran en la carpeta `/Ejemplos/Camera/` de `Ejemplos_Devices/Ejemplos_Devices.slnx` con `<Deploy />`. `DOC/Readme.md` figura en `/Ejemplos/Camera/Docs/`; `DOC/Transferencia_Foto_Entre_Pantallas.md` **no** figura.
- Los cuatro YAML de cámara están listados en la carpeta `/github.workflow/` del `.slnx`.
- El README raíz enlaza el dominio como «📄 Camara → `DOC/Readme.md`» y lo marca «En construcción ...» (`README.md:14-15`).
- La decisión «un proyecto mínimo por técnica» está reconstruida en el repo hermano de documentación (ADR-0001, `Ejemplos_Maui_Devices-docs/docs/04-decisions/0001-un-proyecto-por-tecnica.md`).

---

## §3 Árbol de archivos relevantes

```
Ejemplos_Devices/Camera/
├── Ejemplo_Docs_Photo/
│   ├── Readme.md                          (ISO-8859-1; MediaPicker + CommunityToolkit.Camera)
│   └── Transferencia_Foto_Entre_Pantallas.md  (guía conceptual, 762 líneas)
├── Ejemplo_Photo_MediaPicker/
│   ├── MEDIAPICKER_FIX.md                 (bug ObjectDisposedException + opciones A/B)
│   ├── MauiProgram.cs                     (AddServices → ICamaraService)
│   ├── Pages/MainPage.xaml(.cs)
│   └── Services/ICamaraService.cs, CameraService.cs (clase CamaraService)
├── Ejemplo_Photo_MiMediaPicker_Callback/
│   ├── AppShell.xaml.cs                   (Routing.RegisterRoute MyMediaPickerPage)
│   └── Pages/MainPage.xaml(.cs), MyMediaPickerPage.xaml(.cs)
├── Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/
│   ├── README.md                          (guía de uso rápido)
│   ├── Pages/…                            (MyMediaPickerPage idéntica a CB)
│   └── Services/IImageService.cs, ImageDeviceAutoRotateService.cs  (namespace …Utilities)
├── Ejemplo_Photo_MiMediaPicker_Task/
│   ├── Pages/MainPage.xaml(.cs), MyMediaPickerPage.xaml(.cs)  (MyMediaPickerPage.xaml.cs en Windows-1252; MainPage.xaml.cs en UTF-8 con BOM)
│   └── Platforms/Android/Resources/values/styles.xml          (MainTheme, sin uso visible)
└── Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/
    ├── Pages/MainPage.xaml(.cs), MyMediaSelfiePickerPage.xaml(.cs)
    └── Utilities/IImageDevice.cs, ImageDeviceAutoRotate.cs, SelfieMaskDrawable.cs
```

---

## §4 Escalera didáctica (cómo se relacionan las variantes)

```
MediaPicker (diálogo nativo del SO)                    ← #1
   │  problema: diálogo nativo inestable en algunos Android (DOC/Readme.md:7-9)
   ▼
Página propia con CameraView (CommunityToolkit.Maui.Camera)
   ├── devuelve por Task (TaskCompletionSource<Image>)  ← #4  (el Stream cruza páginas)
   └── devuelve por callback (Action<string?> + path)   ← #2  (el archivo cruza páginas)
          ▼
       + normalización EXIF/resize/JPEG (servicio DI)   ← #3
          ▼
       + selfie: cámara frontal + máscara oval          ← #5  (normalizador con new)
```

| Eje de variación | #1 MP | #2 CB | #3 CBN | #4 TK | #5 SLF |
|---|---|---|---|---|---|
| Captura | `MediaPicker.Default.CapturePhotoAsync()` | `CameraView.CaptureImage()` | ídem #2 | ídem #2 | ídem #2 |
| `CameraView` | — | creada por código en `CameraContainer` | ídem #2 | declarada en XAML (`x:Name="Camera"`) | ídem #2 |
| Navegación | sin navegación | `Shell.GoToAsync(nameof(...), params)` / `GoToAsync("..")` | ídem #2 | `Navigation.PushAsync` / `PopAsync` | ídem #2 |
| Retorno | `Task<Stream?>` del servicio | `Action<string?>` con path en `CacheDirectory` | ídem #2 | `TaskCompletionSource<Image>` | ídem #2 |
| Post-proceso | — | — | `IImageService` por DI | — | `new ImageDeviceAutoRotate{…}` |
| Cámara elegida | la del SO | trasera, o la primera | ídem #2 | trasera (sin alternativa) | **frontal**, o la primera |
| Flash | — | botón visible (Off→On→Auto) | ídem #2 | botón visible | **oculto** |
| Guarda de simulador iOS | — | sí (`#if IOS` + `DeviceType.Virtual`) | sí | **no** | sí |

---

## §5 Flujo por variante

### §5.1 `Ejemplo_Photo_MediaPicker` (#1)

1. `MP/MauiProgram.cs:27-33` registra `AddSingleton<ICamaraService, CamaraService>()`.
2. `MP/Pages/MainPage.xaml.cs:17-25`: el botón «Tomar Foto» llama `_camaraService.TomarFotoAsync()` y asigna `ImageSource.FromStream(() => stream)`.
3. `MP/Services/CameraService.cs:6-21`: todo el cuerpo corre dentro de `MainThread.InvokeOnMainThreadAsync`; si `IsCaptureSupported`, captura, copia `OpenReadAsync()` a un `MemoryStream`, lo rebobina (`Position = 0`) y lo devuelve.
4. `ElegirDeGaleriaAsync()` (`CameraService.cs:23-35`) usa `MediaPicker.Default.PickPhotosAsync()` y toma el primero; **ninguna UI lo invoca**.
5. `MP/Pages/MainPage.xaml.cs:28-58` conserva comentada la versión previa con las opciones A (`FromFile(photo.FullPath)`) y B (`MemoryStream` por bytes), y el «código original (buggy)», en línea con `MP/MEDIAPICKER_FIX.md`.

### §5.2 `Ejemplo_Photo_MiMediaPicker_Callback` (#2)

```
MainPage                                   MyMediaPickerPage
────────                                   ─────────────────
OnAbrirCamaraClicked (MainPage.xaml.cs:11)
  arma Action<string?> resultadoCallback
  GoToAsync(nameof(MyMediaPickerPage),  ──► [QueryProperty OnPhotoCallback] (:7,:14)
     {"OnPhotoCallback", callback})          OnAppearing → EvaluarYMostrarEstadoPermisoAsync (:36-44, :86-148)
                                             MostrarVisorCamara → new CameraView (:152-176)
                                             OnTomarFotoClicked → CaptureImage(token) (:239-274)
                                             OnMediaCaptured (:276-321):
                                               1) e.Media → File.Create(CacheDirectory/photo_<guid>.jpg)
                                               2) InvokeOnMainThreadAsync:
  callback(path) ◄──────────────────────────        OnPhotoCallback?.Invoke(tempPath)
    File.ReadAllBytes(path)                         GoToAsync("..")
    finally File.Delete(path)
    Dispatcher.Dispatch → FromStream(() => new MemoryStream(bytes))
```

- Cancelación: «Volver» invoca `OnPhotoCallback?.Invoke(null)` y `GoToAsync("..")` (`CB/Pages/MyMediaPickerPage.xaml.cs:207-212`); el consumidor ignora `null`/vacío o un path inexistente (`CB/Pages/MainPage.xaml.cs:19-21`).
- Si `e.Media` falla al copiarse, borra el archivo parcial y pasa `null` (`:294-303`).
- `OnDisappearing` (`:46-73`) desuscribe eventos, cancela el token, quita la `CameraView` del contenedor y, en Android, restablece `RequestedOrientation = Unspecified`.
- Es la implementación concreta que recomienda `DOC/Transferencia_Foto_Entre_Pantallas.md` §3.3 y §11.

### §5.3 `Ejemplo_Photo_MiMediaPicker_Task` (#4)

1. `TK/Pages/MainPage.xaml.cs:15-19`: `new MyMediaPickerPage()`, `Navigation.PushAsync(destinoPage)` y `await destinoPage.ResultadoTask.Task`.
2. `TK/Pages/MyMediaPickerPage.xaml.cs:11`: `public TaskCompletionSource<Image> ResultadoTask`.
3. `OnMediaCaptured` (`:227-245`): crea `new Image { Source = ImageSource.FromStream(() => e.Media) }`, `TrySetResult(image)` y `PopAsync()`.
4. `MainPage` copia `imagen.Source` a `ImgPhoto.Source` (`MainPage.xaml.cs:21-24`).
5. Al salir sin foto, `OnDisappearing` hace `TrySetCanceled()` (`:346-350`) para «habilitar el botón tomar foto de la pantalla principal» (comentario `:346`).
6. «Volver» cancela el token, restablece la orientación (Android) y hace `PopAsync()` (`:165-180`); en iOS sólo queda un comentario (`:175-177`).

### §5.4 `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` (#3)

- `MyMediaPickerPage.xaml.cs` es **idéntica** (salvo el namespace) a la de #2; la XAML difiere sólo en `Opacity="1"` del overlay.
- `CBN/MauiProgram.cs:35-41` registra `AddSingleton<IImageService, ImageDeviceAutoRotateService>()`; `MainPage` lo recibe por constructor (`CBN/Pages/MainPage.xaml.cs:9-13`).
- El callback es un lambda `async` (`:21-55`): `ProcesarPhotoAsync(path)` → `File.ReadAllBytes(outPath)` → `Dispatcher.Dispatch(...)`; en el `finally` borra **ambos** archivos, el original y el normalizado (`:43-54`).
- Pipeline detallado en §9.

### §5.5 `Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` (#5)

- Mismo esqueleto que #2/#3 con `MyMediaSelfiePickerPage` (ruta registrada en `SLF/AppShell.xaml.cs:11`).
- Selección de cámara **frontal** con alternativa a la primera disponible (`SLF/Pages/MyMediaSelfiePickerPage.xaml.cs:231`), disparada en `CameraView.Loaded` (`:170`, `:219-222`) y en `OnNavigatedTo` (`:88-95`).
- Flash oculto: `BtnFlashButton.IsVisible = false` (`:176`).
- La máscara se redibuja al cambiar el tema: suscripción a `Application.Current.RequestedThemeChanged` → `MaskOverlay.Invalidate()` (`:43-52`, `:62-63`).
- El normalizador no se inyecta: `new ImageDeviceAutoRotate { MaxWidthHeight = 1000, CompressionQuality = 75, CustomPhotoSize = 50 }` (`SLF/Pages/MainPage.xaml.cs:26-31`). `IImageDevice` no se registra en `SLF/MauiProgram.cs`.
- Detalle de la máscara en §10.

---

## §6 Cómo vuelve el resultado a la página anterior

| Variante | Mecanismo | Qué cruza la frontera | Quién limpia | Cancelación |
|---|---|---|---|---|
| #1 MP | sin navegación; `await` del servicio | `MemoryStream` (único) | GC | `foto is null` → `null` → `Source = null` |
| #2 CB | `ShellNavigationQueryParameters` + `[QueryProperty]` con `Action<string?>` | `string` path en `FileSystem.CacheDirectory` | consumidor (`File.Delete` en `finally`) | `Invoke(null)` desde «Volver» |
| #3 CBN | ídem #2 | path original → path `photo_norm_<guid>.jpg` | consumidor borra ambos | ídem #2 |
| #4 TK | `TaskCompletionSource<Image>` + `PushAsync/PopAsync` | `Image` cuyo `Source` envuelve el `Stream` del toolkit (`e.Media`) | nadie explícito | `TrySetCanceled()` en `OnDisappearing` |
| #5 SLF | ídem #2 | ídem #3 | ídem #3 | ídem #2 |

Claves del patrón callback (#2, #3, #5):

- `ShellNavigationQueryParameters` (a diferencia de un diccionario común) permite pasar un objeto no serializable —un delegado— sin que quede en la URI (`CB/Pages/MainPage.xaml.cs:55-60`).
- El stream del toolkit se materializa a archivo **antes** de saltar al hilo de UI y nunca sale de la página productora (comentario `CB/Pages/MyMediaPickerPage.xaml.cs:278-280`).
- El consumidor captura `bytes`, no `path`, en el lambda de `Dispatcher.Dispatch`, por eso puede borrar el archivo antes de pintar (razonamiento en `DOC/Transferencia_Foto_Entre_Pantallas.md` §7.5).

La comparación callback vs. `Task` como lección en sí misma la recoge ADR-0002 del repo hermano (`Ejemplos_Maui_Devices-docs/docs/04-decisions/0002-servicio-tipado-overlay-mvvm.md:42`).

---

## §7 APIs y servicios clave

| API | Dónde | Uso |
|---|---|---|
| `MediaPicker.Default.IsCaptureSupported` / `CapturePhotoAsync()` / `PickPhotosAsync()` | `MP/Services/CameraService.cs:9-11, 26` | Captura y galería nativas |
| `MediaPicker.Default.IsCaptureSupported` | `CB/…/MyMediaPickerPage.xaml.cs:104, 117`; `SLF/…:117, 128` | Se reutiliza como detector de cámara antes de mostrar la `CameraView` |
| `CommunityToolkit.Maui.Views.CameraView` | `CB/…:160-168`; `TK/Pages/MyMediaPickerPage.xaml:23-27` | Vista previa y captura |
| `CameraView.GetAvailableCameras(ct)` / `SelectedCamera` / `CameraPosition.Rear\|Front` | `CB/…:222-227`; `TK/…:187-190`; `SLF/…:230-234` | Elegir cámara |
| `CameraView.CaptureImage(ct)` + eventos `MediaCaptured` / `MediaCaptureFailed` | `CB/…:253, 276, 323` | Disparo asíncrono; el resultado llega por evento (`e.Media`) |
| `CameraView.CameraFlashMode` (`CameraFlashMode.Off/On/Auto`) | `CB/…:336-360` | Ciclo del flash; ícono vía `FlashIcon` enlazado (`BindingContext = this`) |
| `CameraView.IsAvailable` | `CB/…:311` | Si es `false` muestra «El dispositivo no está activo» |
| `Permissions.CheckStatusAsync/RequestAsync<Permissions.Camera>()` | `CB/…:100, 113`; `TK/…:62, 92` | Flujo de permisos (§8) |
| `Permissions.ShouldShowRationale<Permissions.Camera>()` | `CB/…:136-138` (`#if ANDROID`) | Decide «Pedir permiso» vs «Abrir configuración» |
| `AppInfo.ShowSettingsUI()` | `CB/…:202-205` | Abre los ajustes de la app |
| `DeviceDisplay.MainDisplayInfoChanged` | `CB/…:40, 52, 362-366` | Reacomoda la grilla en apaisado (§11.4) |
| `FileSystem.CacheDirectory` | `CB/…:285-287`; `CBN/Services/ImageDeviceAutoRotateService.cs:85` | Archivos temporales `photo_*.jpg` / `photo_norm_*.jpg` |
| `MainThread.InvokeOnMainThreadAsync` / `BeginInvokeOnMainThread`, `Dispatcher.Dispatch` | `CB/…:154, 180, 309`; `CB/Pages/MainPage.xaml.cs:49` | Saltos al hilo de UI |
| `Routing.RegisterRoute` + `Shell.Current.GoToAsync` | `CB/AppShell.xaml.cs:11`; `CB/…:211, 314` | Navegación Shell |
| `MetadataExtractor.ImageMetadataReader` | `CBN/Services/ImageDeviceAutoRotateService.cs:23-28` | Leer `ExifIfd0Directory.TagOrientation` |
| `SkiaSharp` (`SKBitmap.Decode`, `Resize`, `SKCanvas`, `SKImage.Encode`) | `CBN/Services/ImageDeviceAutoRotateService.cs:34-58, 139-183` | Rotar, escalar y recomprimir JPEG |
| `Microsoft.Maui.Graphics.IDrawable`, `PathF`, `WindingMode.EvenOdd` | `SLF/Utilities/SelfieMaskDrawable.cs:5-49` | Máscara oval |

---

## §8 Flujo de permisos (variantes con `CameraView`)

```
OnAppearing
 └─ [#if IOS && DeviceType.Virtual] → overlay «Cámara no disponible» (sin reintento)   (#2,#3,#5)
 └─ CheckStatusAsync<Camera>
     ├─ Granted ──► IsCaptureSupported ? MostrarVisorCamara : overlay «sin cámara»
     └─ otro ─► RequestAsync<Camera>
                ├─ Granted ──► ídem
                ├─ Restricted ──► overlay «Acceso restringido» (sin reintento)
                └─ Denied ──► Android: ShouldShowRationale
                               ├─ true  → overlay + botón «Pedir permiso»
                               └─ false → overlay + botón «Abrir configuración»
```

- Fuente: `CB/Pages/MyMediaPickerPage.xaml.cs:86-148`; idéntica (mismas líneas) en `CBN/Pages/MyMediaPickerPage.xaml.cs`, y la misma lógica en `SLF/Pages/MyMediaSelfiePickerPage.xaml.cs:99-150`.
- El botón «Volver» vive **sólo dentro del overlay** de permisos (`CB/Pages/MyMediaPickerPage.xaml:40`): con el visor activo no hay «Volver» propio; salir con el back del sistema/barra de navegación no invoca el callback (en #2/#3/#5 el consumidor simplemente no recibe nada; en #4 `OnDisappearing` cancela el TCS).
- #4 TK separa el flujo en `EvaluarYMostrarEstadoPermisoAsync` + `PedirPermisoYActualizarVistaAsync`, acepta `Granted` **o `Limited`** y explica en un comentario que una instalación nueva de Android devuelve `Denied` y no `Unknown` (`TK/Pages/MyMediaPickerPage.xaml.cs:60-119`, comentario `:80-83`).
- El overlay (`PermissionDeniedOverlay`) comparte la celda de la vista previa; muestra el glifo `no_photography` de `MaterialIconsOutlined` y los botones «Pedir permiso», «Abrir configuración» y «Volver» (`CB/Pages/MyMediaPickerPage.xaml:25-43`).

---

## §9 Normalización EXIF (#3 y #5)

**Pipeline** (`CBN/Services/ImageDeviceAutoRotateService.cs`; copia textual en `SLF/Utilities/ImageDeviceAutoRotate.cs`):

| Paso | Líneas | Qué hace |
|---|---|---|
| 1 | `:15` | `await Task.Yield()` |
| 2 | `:22-28` | `ImageMetadataReader.ReadMetadata(stream)` → `ExifIfd0Directory.TagOrientation` |
| 3 | `:33-36` | `SKBitmap.Decode(stream)` y `AplicarOrientation(bitmap, orientation)` |
| 4 | `:39-48` | Tamaño destino: `CustomPhotoSize`% del original; si un lado supera `MaxWidthHeight`, escala proporcional para encajar en `MaxWidthHeight` |
| 5 | `:52-58` | `Resize(..., SKFilterMode.Linear, SKMipmapMode.Nearest)` → `Encode(Jpeg, CompressionQuality)` → `byte[]` |
| path→path | `:73-89` | Sobrecarga que reutiliza la de `Stream` y escribe `CacheDirectory/photo_norm_<guid>.jpg` |

**Parámetros por defecto**: `MaxWidthHeight = 1000`, `CompressionQuality = 75`, `CustomPhotoSize = 50` (`:9-11`). #5 los repite explícitamente al instanciar (`SLF/Pages/MainPage.xaml.cs:26-31`).

**Tabla de orientaciones** (`AplicarOrientation`, `:91-136`):

| EXIF | Comentario en el código | Llamada |
|---|---|---|
| 1 | normal | sin cambio |
| 2 | voltear horizontalmente | `Rotate(bmp, 0, -1, 1)` |
| 3 | girar 180° | `Rotate(bmp, 180)` |
| 4 | voltear verticalmente | `Rotate(bmp, 0, 1, -1)` |
| 5 | voltear vertical + rotar 90° derecha | `Rotate(bmp, 90, 1, -1)` |
| 6 | girar 90° derecha | `Rotate(bmp, 90)` |
| 7 | girar 90° izquierda + voltear vertical | `Rotate(bmp, -90, -1, 1)` |
| 8 | girar 90° izquierda | `Rotate(bmp, -90)` |
| otro / sin EXIF | — | sin cambio |

- `Rotate` se atribuye a `https://github.com/djdd87/SynoAI` (`:138`).
- ⚠️ Los volteos (2, 4, 5, 7) no se aplican: ver §15, T-7.
- La firma doble `Stream → byte[]` + `path → path` es la que propone `DOC/Transferencia_Foto_Entre_Pantallas.md` §6.1; el código usa `using var` en lugar del bloque `using (…) { }` del documento, que es equivalente.
- Política de limpieza: el consumidor borra original y normalizado en su `finally` (`CBN/Pages/MainPage.xaml.cs:43-54`), lo que corresponde a la «política 2» del documento (§6.3), aunque éste recomienda la 1 para el ejemplo didáctico.

---

## §10 Selfie: máscara y cámara frontal (#5)

- `SelfieMaskDrawable` (`SLF/Utilities/SelfieMaskDrawable.cs`) pinta un rectángulo completo con un óvalo recortado (`PathF.AppendRectangle` + `AppendEllipse`, `FillPath(..., WindingMode.EvenOdd)`, `:37-41`).
- Proporción: ancho `WidthFraction = 0.75` del área, alto = ancho × `HeightToWidthRatio = 1.35`, con tope en 90 % del alto (`:13-32`).
- Color: si `OverlayColor` es `null`, blanco en tema claro y negro en oscuro (`:22-24`); borde opcional (`BorderThickness = 0` por defecto, `:10, 43-48`).
- XAML: `<util:SelfieMaskDrawable x:Key="SelfieMask"/>` en recursos y `GraphicsView` con `InputTransparent="True"` superpuesta a `CameraContainer` dentro de una `Grid` anidada (`SLF/Pages/MyMediaSelfiePickerPage.xaml:9-11, 24-27`).
- Fondo de página y botones con `AppThemeBinding` (`:7, 17, 52`).
- `Resources/Images/mascara.png` está versionada y excluida de `None` en el `.csproj` (`SLF/….csproj:81-83`), pero ningún `.cs`/`.xaml` la referencia (la máscara se dibuja por código).
- El `.csproj` activa `<MauiXamlInflator>SourceGen</MauiXamlInflator>` (`SLF/Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion.csproj:15`), único en el dominio.

---

## §11 Permisos y configuración por plataforma

### §11.1 Android (`Platforms/Android/AndroidManifest.xml`)

| Entrada | MP | CB | CBN | TK | SLF |
|---|---|---|---|---|---|
| `ACCESS_NETWORK_STATE`, `INTERNET` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `CAMERA` | ✓ | ✓ | ✓ | ✓ | ✓ |
| `RECORD_AUDIO` | ✓ | — | ✓ | ✓ | ✓ |
| `WRITE_EXTERNAL_STORAGE maxSdkVersion=32` | ✓ | — | — | — | — |
| `uses-feature camera` (+ `autofocus`) | `required=true` (ambos) | `camera required=false` | — | — | — |
| `<queries>` `IMAGE_CAPTURE` | — | — | ✓ | ✓ | ✓ |
| `uses-sdk min/target` | 25 / 36 | 25 / 36 | 25 / 36 | 25 / 36 | 25 / 36 |

`MainActivity` es la de plantilla en los cinco (`Theme = "@style/Maui.SplashTheme"`, `SingleTop`, `ConfigurationChanges` con `Orientation`/`ScreenSize`).

### §11.2 iOS (`Platforms/iOS/Info.plist`)

| Clave | MP | CB | CBN | TK | SLF |
|---|---|---|---|---|---|
| `NSCameraUsageDescription` | ES | ES | «Require to use camera» | ES (adjuntar fotos a formularios) | «Require to use camera» |
| `NSMicrophoneUsageDescription` | ES | ES | — | «This app does not require access to the microphone.» | — |
| `NSPhotoLibraryUsageDescription` | — | — | ✓ | — | ✓ |
| `NSPhotoLibraryAddUsageDescription` | — | — | ✓ | ✓ (texto sobre «claims») | ✓ |
| `MinimumOSVersion` | 15.0 | 15.0 | 15.0 | 15.0 | 15.0 |

Los cinco `Entitlements.plist` y `Entitlements.Development.plist` son idénticos entre sí: `aps-environment=development` y `get-task-allow=true`. El workflow firma con `Entitlements.Development.plist`.

### §11.3 `.csproj` comunes

- `TargetFrameworks`: `net10.0-android`; `net10.0-ios` sólo si se compila en macOS (`IsOSPlatform('osx')`). Sin target Windows (aunque `Properties/launchSettings.json` trae el perfil «Windows Machine» de plantilla).
- `SupportedOSPlatformVersion`: iOS 15.0, Android 25.0. `AndroidLdFlags=-Wl,-z,max-page-size=16384` y `RuntimeIdentifiers` para las cuatro ABI de Android.
- #2–#5 agregan: APK (`AndroidPackageFormat=apk`, `AndroidCreatePackagePerAbi=False`), y para iOS Release `UseInterpreter`, `PublishTrimmed`, `MtouchLink=SdkOnly`, AOT sólo en `ios-arm64`.

| Proyecto | `ApplicationId` / `CFBundleIdentifier` | `ApplicationTitle` |
|---|---|---|
| MP | `com.ejemplos.devices.mediapicker` | Mi MediaPicker |
| CB | `com.ejemplos.photo.mimediapicker.callback` | Mi MediaPicker Callback |
| CBN | `com.ejemplos.devices.imagen.callback.normalizacion` | Callback Normalizacion |
| TK | `com.ejemplos.devices.mimediapicker.task` | Mi MediaPicker Task |
| SLF | `com.ejemplos.devices.imagen.callback.normalizacion.miselfie` | Selfie Callback Normalizacion |

### §11.4 Orientación

`UpdateLayoutOrientation` reescribe filas y columnas de `DynamicLayout`: en vertical, flash arriba, vista previa al centro y disparador abajo; en apaisado, disparador a la izquierda, vista previa al centro y flash a la derecha (`CB/Pages/MyMediaPickerPage.xaml.cs:368-409`). Al salir, en Android se restablece `ScreenOrientation.Unspecified`.

---

## §12 Paquetes NuGet y versiones

| Paquete | MP | CB | CBN | TK | SLF |
|---|---|---|---|---|---|
| `Microsoft.Maui.Controls` | `$(MauiVersion)` | 10.0.30 | 10.0.31 | 10.0.30 | 10.0.31 |
| `Microsoft.Extensions.Logging.Debug` | 10.0.2 | 10.0.2 | 10.0.2 | 10.0.2 | 10.0.2 |
| `CommunityToolkit.Maui.Camera` | — | 6.0.0 | 6.0.0 | 6.0.0 | 6.0.0 |
| `CommunityToolkit.Maui.Core` | — | 14.0.0 | 14.0.0 | 14.0.0 | 14.0.0 |
| `MetadataExtractor` | — | — | 2.9.0 | — | 2.9.0 |
| `SkiaSharp` | — | — | 3.119.1 | — | 3.119.1 |

- Inicialización: `.UseMauiCommunityToolkitCore().UseMauiCommunityToolkitCamera()` (`CB/MauiProgram.cs:15-16`; ídem CBN, TK, SLF).
- Fuentes: `MaterialIconsOutlined-Regular.otf` (glifos `photo_camera`, `flash_*`, `no_photography`) en #2–#5; #4 registra además tres Font Awesome 6 (`TK/MauiProgram.cs:20-22`).
- Las versiones divergentes entre proyectos son una consecuencia aceptada de ADR-0001 (repo hermano).

---

## §13 Solución, scripts y CI

- **Script local Android**: `Ejemplos_Devices/scripts/Ejemplo_Photo_MediaPicker_launch.bat` hace `dotnet build ... -f net10.0-android -t:Run` sobre `MP`, previa verificación de `adb devices` (`:6-7, 30-37`). Es el único script del dominio. ADR-0006 (repo hermano) reconstruye la decisión «Android local, iOS en CI».
- **Workflows** (`WF/cd-ios-camera.<Proyecto>.yml`), cuatro casi idénticos:

| Parámetro | Valor |
|---|---|
| Disparador | sólo `workflow_call:`; el `push` a `main` con filtro de paths está comentado (`:5-15`) |
| Runner | `macos-15` |
| Xcode | 26.0, descargado de Google Drive con `gdown` (ID distinto en el de selfie: `Xcode_26_Universal_1.xip`) |
| .NET | SDK `10.0.300`, workloads `ios maui maui-ios` `10.0.100` |
| Build | `dotnet build -c Release -f net10.0-ios -p:RuntimeIdentifier=iossimulator-arm64 -p:EnableCodeSigning=false …` |
| Firma | ad hoc (`codesign --sign "-"`) con `Entitlements.Development.plist` |
| Evidencia | `.app.zip` y GIF por `Utilities/simular.sh` en el simulador «iPhone 17 Pro Max» (`continue-on-error: true`) |

- `MP` no tiene workflow.

---

## §14 Documentación del dominio

| Documento | Contenido | Observación |
|---|---|---|
| `DOC/Readme.md` | Dos enfoques (MediaPicker y CommunityToolkit.Camera), permisos Android de la doc de Microsoft, `dotnet add package`, snippet de `MauiProgram` | ISO-8859-1; dice «dos ejemplos» |
| `DOC/Transferencia_Foto_Entre_Pantallas.md` | Guía conceptual: `Stream` vs `byte[]` vs path vs singleton vs `WeakReferenceMessenger` vs `FileProvider`; apps reales; downsampling/EXIF; pipeline path→path; `Dispatcher`/`MainThread`; multipart vs base64; antipatrones; recomendación §11 | Cita `Ejemplo_Photo_MiMediaPicker_Callback` como implementación (§3.3, §11). Las cifras de §4 se declaran aproximadas (disclaimer `:267-271`) |
| `MP/MEDIAPICKER_FIX.md` | Causa raíz del crash en Debug (`ObjectDisposedException` en `GlideExecutor`), opción A `FromFile`, opción B `MemoryStream`, galería/`MediaStore` | Recomienda A (`:142`) |
| `CBN/README.md` | Guía de uso rápido: NuGet, `AddServices`, callback, manifiestos | Snippet idéntico a `CBN/Pages/MainPage.xaml.cs` |

---

## §15 Trampas ⚠️ y discrepancias verificadas

| ID | Trampa / discrepancia | Evidencia |
|---|---|---|
| T-1 | ⚠️ `MP` inyecta la clase **concreta** `CamaraService` en el constructor de `MainPage`, pero el contenedor sólo registra `ICamaraService`; la página se crea desde `DataTemplate` del Shell. Probable fallo al resolver `MainPage` (no verificado en ejecución). | `MP/Pages/MainPage.xaml.cs:9-11` vs `MP/MauiProgram.cs:29`; `MP/AppShell.xaml:10` |
| T-2 | ⚠️ `using Android.Media;` sin `#if ANDROID` en código compartido: no compila para `net10.0-ios` (no verificado con build; el proyecto no tiene workflow iOS). | `MP/Pages/MainPage.xaml.cs:1` |
| T-3 | ⚠️ `MP` asigna `FromStream(() => stream)` devolviendo **siempre la misma** instancia de `MemoryStream`. `MEDIAPICKER_FIX.md` (opción B) y `Transferencia` §3.1 piden un stream nuevo por invocación; si MAUI reevalúa el `ImageSource`, el stream ya está consumido. | `MP/Pages/MainPage.xaml.cs:22-24` vs `MP/MEDIAPICKER_FIX.md:73, 99` |
| T-4 | ⚠️ `MEDIAPICKER_FIX.md` presenta la opción A como solución vigente, pero el código activo usa el servicio con `MemoryStream`; la opción A quedó comentada. | `MP/MEDIAPICKER_FIX.md:142` vs `MP/Pages/MainPage.xaml.cs:17-38` |
| T-5 | `CamaraService` envuelve **toda** la captura en `InvokeOnMainThreadAsync`, cosa que `Transferencia` §7.6 desaconseja («saltar al UI thread justo a tiempo»). Archivo `CameraService.cs` ≠ clase `CamaraService`. `ElegirDeGaleriaAsync` no tiene UI y el manifiesto no declara `READ_MEDIA_IMAGES` (que sí lista `DOC/Readme.md:19-22`). | `MP/Services/CameraService.cs:6-7, 23`; `MP/Platforms/Android/AndroidManifest.xml:10-20` |
| T-6 | ⚠️ En #4 TK el `Stream` del toolkit (`e.Media`) cruza páginas dentro de un `ImageSource.FromStream` diferido: es exactamente el antipatrón §3.1/§9 de `Transferencia`. | `TK/Pages/MyMediaPickerPage.xaml.cs:235` |
| T-7 | ⚠️ En los volteos EXIF (2, 4, 5, 7) `canvas.Scale(ex, ey)` se llama **después** de `DrawBitmap`, así que no tiene efecto: la imagen se rota pero no se espeja, contra lo que dicen los comentarios. Afecta a #3 y #5. | `CBN/Services/ImageDeviceAutoRotateService.cs:178-180`; `SLF/Utilities/ImageDeviceAutoRotate.cs:178-180` |
| T-8 | ⚠️ `TK/MainPage` traga toda excepción con `catch (Exception ex) { }`: al volver sin foto, `TrySetCanceled()` produce `TaskCanceledException` y la rama «Cancelado / No se recibió ningún dato» es inalcanzable (el TCS nunca recibe `null`). | `TK/Pages/MainPage.xaml.cs:21-32` vs `TK/Pages/MyMediaPickerPage.xaml.cs:346-350` |
| T-9 | ⚠️ Mojibake: literales de UI con el carácter U+FFFD guardado en el archivo («C�mara», «est�»…) en 19 líneas de CB/CBN y 10 de SLF; TK está codificado en Windows-1252 sin BOM (su visualización depende de cómo lo decodifique el compilador, no verificado). Algunas cadenas de SLF están bien («Cámara no disponible», `:120, 131`) y otras no en la misma línea. | `CB/Pages/MyMediaPickerPage.xaml.cs:93-94, 107-108, 120-121, 129, 142-143, 145-146, 229-230, 261-262, 318` (+ comentarios `:150, 214, 334`); ídem `CBN/`; `SLF/Pages/MyMediaSelfiePickerPage.xaml.cs:106-107, 120, 137, 147-148, 236, 318` (+ comentarios `:152, 217`); `TK/Pages/MyMediaPickerPage.xaml.cs` |
| T-10 | ⚠️ `MyMediaSelfiePickerPage` vive en el namespace **de otro proyecto** (`Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.Pages`), arrastrado de la copia; compila porque es el mismo ensamblado. El `CFBundleName` de SLF también conserva el nombre de CBN. | `SLF/Pages/MyMediaSelfiePickerPage.xaml.cs:5`; `SLF/Pages/MyMediaSelfiePickerPage.xaml:5`; `SLF/AppShell.xaml.cs:1`; `SLF/Platforms/iOS/Info.plist:41` |
| T-11 | ⚠️ En SLF, `UpdateLayoutOrientation` reposiciona `CameraContainer`, pero éste ya no es hijo directo de `DynamicLayout` (está dentro de la `Grid` anidada con la máscara). En apaisado la `Grid` anidada queda en `Row=1/Col=0` y la vista previa no se mueve a la columna central (no verificado en ejecución). | `SLF/Pages/MyMediaSelfiePickerPage.xaml.cs:386, 398` vs `SLF/Pages/MyMediaSelfiePickerPage.xaml:24-27` |
| T-12 | Trampa de lectura: en CBN los archivos están en `Services/` pero declaran el namespace `…Utilities` (`MainPage` hace `using …Utilities`). | `CBN/Services/IImageService.cs:1`; `CBN/Services/ImageDeviceAutoRotateService.cs:5`; `CBN/MauiProgram.cs:3` |
| T-13 | ⚠️ `CBN/README.md` lista como dependencias sólo los dos paquetes del toolkit; el proyecto también necesita `MetadataExtractor` 2.9.0 y `SkiaSharp` 3.119.1. | `CBN/README.md:7-10` vs `CBN/….csproj:79-80` |
| T-14 | ⚠️ `DOC/Readme.md` habla de «dos ejemplos» (hay cinco) y su snippet de `MauiProgram` sólo llama `UseMauiCommunityToolkitCore()`; los proyectos con `CameraView` también llaman `UseMauiCommunityToolkitCamera()`. | `DOC/Readme.md:4-9, 48-74` vs `CB/MauiProgram.cs:15-16` |
| T-15 | `CustomPhotoSize = 50` reduce a la mitad **toda** imagen cuyos lados no superen 1000 px; las mayores se ajustan a 1000 px e ignoran el 50 %. El `SKBitmap` rotado intermedio no se libera. | `CBN/Services/ImageDeviceAutoRotateService.cs:36-48` |
| T-16 | En CBN/SLF el callback es un lambda `async` asignado a `Action<string?>` (equivale a `async void`): la productora navega con `GoToAsync("..")` sin esperar la normalización, y las excepciones sólo van a `Debug.WriteLine`. Si `ProcesarPhotoAsync` devuelve `null`, `File.ReadAllBytes("")` lanza y cae en ese `catch`. | `CBN/Pages/MainPage.xaml.cs:21, 28-30, 39-42`; `CB/Pages/MyMediaPickerPage.xaml.cs:313-314` |
| T-17 | TK: `OnDisappearing` llama `base.OnDisappearing()` dos veces; la ruta `MyMediaPickerPage` se registra en `AppShell` pero no se usa (se navega con `PushAsync`); `SeleccionarCamaraAsync` no tiene alternativa si falta la trasera; no hay guarda de simulador iOS. | `TK/Pages/MyMediaPickerPage.xaml.cs:334, 363, 188`; `TK/AppShell.xaml.cs:11` |
| T-18 | Restos de plantilla o copia: `<MauiXaml Update>` y `<None Update>` duplicados para la misma XAML (CBN, TK); `styles.xml` con `MainTheme` que `MainActivity` no usa (TK); `<queries> IMAGE_CAPTURE` y `aps-environment` (push) en proyectos que no los usan; el comentario del YAML de TK dice «proyecto GPS»; el script `Ejemplo_Photo_MediaPicker_launch.bat` imprime el encabezado «Ejemplo_Maui_GPS» aunque compila `MP`. | `CBN/….csproj:86-96`; `TK/….csproj:87-97`; `TK/Platforms/Android/Resources/values/styles.xml:3`; `WF/cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Task.yml:33`; `Ejemplos_Devices/scripts/Ejemplo_Photo_MediaPicker_launch.bat:6, 23` |
| T-19 | Los cuatro workflows sólo declaran `workflow_call` y ningún YAML del repo los invoca ni tiene `workflow_dispatch`: no se disparan solos desde este repo (lectura de los YAML; no verificado en GitHub). | `WF/cd-ios-camera.*.yml:3-15` |
| T-20 | Trampa de lectura: en #2/#3 la selección de la cámara trasera sólo ocurre en `OnNavigatedTo` y bajo la guarda `_cameraView != null`; la `CameraView` se crea más tarde (tras el `await` de permisos y dentro de `BeginInvokeOnMainThread`), así que en la primera apertura puede quedar la cámara por defecto del toolkit, y el camino «Pedir permiso» → `MostrarVisorCamara` tampoco la selecciona (no verificado en ejecución). #5 lo cubre suscribiendo `CameraView.Loaded`; #4 la llama además tras `RequestAsync` concedido. | `CB/Pages/MyMediaPickerPage.xaml.cs:75-82, 152-168`; `SLF/Pages/MyMediaSelfiePickerPage.xaml.cs:170, 219-222`; `TK/Pages/MyMediaPickerPage.xaml.cs:94-97` |
| T-21 | En #2/#3/#5, si `CameraView.IsAvailable` es `false` al recibir la foto, se muestra la alerta pero no se invoca el callback ni se borra el `photo_<guid>.jpg` ya escrito en `CacheDirectory`. | `CB/Pages/MyMediaPickerPage.xaml.cs:283-319` |

---

## §16 Preguntas frecuentes

| Pregunta | Sección |
|---|---|
| ¿Qué proyecto uso para el diálogo nativo de cámara? | §2, §5.1 |
| ¿Por qué existe una página de cámara propia si ya está `MediaPicker`? | §4, §14 |
| ¿Cómo vuelve la foto a la página anterior en cada variante? | §6 |
| ¿Por qué se pasa un path y no el `Stream`? | §6, §14, §15 T-6 |
| ¿Cómo se pasa un delegado por navegación Shell? | §5.2, §6 |
| ¿Cuál es la diferencia entre la variante callback y la variante `Task`? | §4, §5.3, §6 |
| ¿Qué pasa si el usuario niega el permiso de cámara? | §8 |
| ¿Cómo se corrige la rotación EXIF y qué parámetros usa? | §9 |
| ¿Se espejan bien las orientaciones EXIF 2/4/5/7? | §15 T-7 |
| ¿Cómo se dibuja la máscara de la selfie y cómo se elige la cámara frontal? | §5.5, §10 |
| ¿Qué permisos y claves de `Info.plist` necesita cada ejemplo? | §11.1, §11.2 |
| ¿Qué versiones de NuGet usa cada proyecto? | §12 |
| ¿Qué ejemplos compila el CI de iOS y cómo? | §13 |
| ¿Cómo lanzo el ejemplo MediaPicker en un Android físico? | §13 |
| ¿Por qué el ejemplo MediaPicker crasheaba en Debug? | §14, §15 T-3/T-4 |
| ¿Por qué veo «C�mara» en la UI? | §15 T-9 |
| ¿Por qué a veces abre con otra cámara que la trasera? | §15 T-20 |
| ¿Dónde se reutilizan estas páginas en la app híbrida? | [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) |

---

## §17 Fuentes

| Fuente | Aporta |
|---|---|
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | Pertenencia a la solución (§2, §13) |
| `README.md:14-15` | Enlace al dominio, «En construcción» |
| `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/Readme.md` | Enfoques y permisos documentados (§14, T-14) |
| `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` | Marco conceptual de transferencia, pipeline e hilos (§6, §9, §14) |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MediaPicker/{MauiProgram.cs, Pages/MainPage.xaml(.cs), Services/*.cs, MEDIAPICKER_FIX.md, *.csproj, Platforms/*}` | #1 (§5.1, T-1 a T-5) |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Callback/{MauiProgram.cs, AppShell.xaml(.cs), Pages/*, *.csproj, Platforms/*}` | #2 (§5.2, §7, §8, §11.4) |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/{Services/*, Pages/*, README.md, *.csproj, Platforms/*}` | #3 (§5.4, §9, T-7, T-12, T-13) |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Task/{Pages/*, MauiProgram.cs, *.csproj, Platforms/*}` | #4 (§5.3, T-6, T-8, T-17) |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/{Pages/*, Utilities/*, AppShell.xaml.cs, *.csproj, Platforms/*}` | #5 (§5.5, §10, T-10, T-11) |
| `Ejemplos_Devices/scripts/Ejemplo_Photo_MediaPicker_launch.bat` | Lanzamiento local en Android (§13) |
| `.github/workflows/cd-ios-camera.*.yml` (4) | CI iOS (§13, T-19) |
| Repo hermano `Ejemplos_Maui_Devices.Documentacion`: ADR-0001, ADR-0002, ADR-0006 (`Ejemplos_Maui_Devices-docs/docs/04-decisions/`) | Contexto de decisiones (sólo lectura) |
