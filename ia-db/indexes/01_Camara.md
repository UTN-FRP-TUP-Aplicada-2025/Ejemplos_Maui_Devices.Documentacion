# 01 — Cámara y captura de fotos

> **Propósito:** cómo los ejemplos capturan una foto y la devuelven a la pantalla anterior, con las tres técnicas de transferencia entre páginas y la normalización de la imagen.
> **Fuente primaria:** `Ejemplos_Devices/Camera/` (5 proyectos MAUI + 1 carpeta de documentación).
> **Índices relacionados:** [00_MASTER-INDEX](00_MASTER-INDEX.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (la híbrida reutiliza estas páginas) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) (4 de los 5 tienen workflow iOS).

---

## 1. Los cinco proyectos y qué enseña cada uno

El dominio es una **progresión didáctica**: cada proyecto agrega exactamente una idea sobre el anterior.

| # | Proyecto | `ApplicationId` | Enseña |
|---|----------|-----------------|--------|
| 1 | `Ejemplo_Photo_MediaPicker` | `com.ejemplos.devices.mediapicker` | El diálogo **nativo** del sistema (`MediaPicker.Default`), sin UI propia |
| 2 | `Ejemplo_Photo_MiMediaPicker_Task` | `com.ejemplos.devices.mimediapicker.task` | Visor **propio** con `CameraView`; el resultado vuelve por `TaskCompletionSource` |
| 3 | `Ejemplo_Photo_MiMediaPicker_Callback` | `com.ejemplos.photo.mimediapicker.callback` | Mismo visor; el resultado vuelve por **callback + archivo temporal** |
| 4 | `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion` | Igual que (3) + **normalización EXIF/escalado** por servicio inyectado |
| 5 | `Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion.miselfie` | Igual que (4) pero **cámara frontal** y máscara ovalada de encuadre |

Documentación del dominio: `Camera/Ejemplo_Docs_Photo/Readme.md` (permisos y arranque de ambas técnicas) y `Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` (guía conceptual de 12 secciones que compara las seis técnicas de transferencia y cómo lo resuelven WhatsApp/Instagram/Telegram/Signal).

## 2. Stack por proyecto

| Proyecto | Paquetes propios del dominio |
|----------|------------------------------|
| (1) MediaPicker | ninguno — solo `Microsoft.Maui.Controls` (`$(MauiVersion)`) |
| (2) Task, (3) Callback | `CommunityToolkit.Maui.Camera` 6.0.0 · `CommunityToolkit.Maui.Core` 14.0.0 · MAUI 10.0.30 |
| (4) y (5) Normalizacion | lo anterior + `MetadataExtractor` 2.9.0 · `SkiaSharp` 3.119.1 · MAUI 10.0.31 |

Todos: `net10.0-android` (+ `net10.0-ios` solo al compilar en macOS), Android mínimo API 25, iOS 15.0, `Nullable` habilitado. Fuente: los cinco `.csproj`.

## 3. Técnica 1 — Diálogo nativo (`MediaPicker`)

No hay visor propio: se delega en la app de cámara del sistema.

```
MainPage.OnAbrirCamaraClicked → CamaraService.TomarFotoAsync() → MediaPicker.Default.CapturePhotoAsync()
                                                              → copia a MemoryStream → ImageSource.FromStream
```

- `Services/CameraService.cs:6-21` — `TomarFotoAsync()`; `:23-35` — `ElegirDeGaleriaAsync()` (galería, `PickPhotosAsync`). Ambos envuelven todo en `MainThread.InvokeOnMainThreadAsync` y **copian a `MemoryStream` antes de salir**, porque el archivo temporal puede limpiarse apenas termina el intent de cámara.
- Contrato en `Services/ICamaraService.cs:7-11`; registro `AddSingleton<ICamaraService, CamaraService>()` en `MauiProgram.cs:29`.
- `Pages/MainPage.xaml.cs:28-58` conserva **comentado** el camino anterior con las tres variantes (A `FromFile`, B `MemoryStream`, y el código original con bug), que es el material de estudio de `MEDIAPICKER_FIX.md`.

### 3.1 El bug de `ImageSource.FromStream` (documento `MEDIAPICKER_FIX.md`)

`ImageSource.FromStream` es **lazy**: no lee el stream al asignarlo, le pasa el delegate al motor de imágenes (Glide en Android), que lo consume después en otro hilo. Si el stream se abrió con `using`, ya está cerrado para entonces → `ObjectDisposedException` («Cannot access a closed file») visible en logcat vía `GlideExecutor`. Se manifiesta en Debug y no siempre en Release, pero el código es incorrecto en ambos. Fuente: `Ejemplo_Photo_MediaPicker/MEDIAPICKER_FIX.md`.

| Opción | Cómo | Cuándo conviene |
|--------|------|-----------------|
| A — `ImageSource.FromFile(photo.FullPath)` | Glide abre y cierra su propio `FileStream`; cachea por ruta y hace sampling | Caso simple; es la recomendada por el documento |
| B — copiar a `MemoryStream` | Leer todos los bytes con el stream abierto y devolver un `MemoryStream` nuevo por invocación | Si hay que transformar los bytes antes de mostrar |
| ✗ original | `using` + `FromStream(() => sourceStream)` | Nunca — es el bug |

⚠️ **Trampa de compilación:** `Pages/MainPage.xaml.cs:1` tiene `using Android.Media;` en un archivo **compartido** (no bajo `Platforms/`). El proyecto compila porque el TFM habitual es `net10.0-android`; con el TFM de iOS activo ese `using` no resuelve.

## 4. Técnica 2 — Visor propio con `CameraView`

Los proyectos (2)–(5) reemplazan el diálogo nativo por una página propia (`MyMediaPickerPage`). El motivo lo da `Ejemplo_Docs_Photo/Readme.md`: el diálogo nativo dio problemas en algunas versiones de Android, así que se recreó uno minimalista.

Anatomía de la página (referencia: `Ejemplo_Photo_MiMediaPicker_Callback/Pages/MyMediaPickerPage.xaml.cs`, 409 líneas):

| Bloque | Líneas | Qué resuelve |
|--------|--------|--------------|
| Permisos | `:86-148` | `CheckStatusAsync` → `RequestAsync` → `Restricted` → denegado; decide visor u overlay |
| Overlay | `:152-193` | `MostrarVisorCamara()` / `MostrarOverlayPermiso()`; ambos entran por `MainThread.BeginInvokeOnMainThread` |
| Reintento | `:197-205` | `BtnPedirPermiso` re-evalúa; `BtnGoToSettings` abre `AppInfo.ShowSettingsUI()` |
| Selección de cámara | `:216-237` | `GetAvailableCameras` → prefiere `CameraPosition.Rear` |
| Captura | `:239-274` | Guardia `_isCapturingImage`, `CancellationTokenSource`, `CaptureImage` en el UI thread |
| Resultado | `:276-321` | Ver §5 |
| Flash | `:336-360` | Ciclo `Off → On → Auto` y el ícono `flash_off/on/auto` por `FlashIcon` (fuente `MaterialIconsOutlined`) |
| Orientación | `:362-409` | Rearma `RowDefinitions`/`ColumnDefinitions` del `Grid` `DynamicLayout` en `BatchBegin/BatchCommit` según `DisplayOrientation` |

Guardias de entorno declaradas en el propio código:
- `:89-98` (`#if IOS`) — el **simulador de iOS no soporta `CameraView`**: si `DeviceInfo.Current.DeviceType == Virtual`, muestra overlay y no intenta abrir la cámara.
- `:257-264` — `InvalidCastException` al capturar se interpreta como «el dispositivo no soporta la cámara integrada».
- `:68-72` / `OnDisappearing` — en Android restablece `RequestedOrientation = Unspecified` al salir.

## 5. El eje del dominio: cómo vuelve la foto a la pantalla anterior

Es la decisión que separa los proyectos (2), (3) y (4)/(5).

| | (2) `_Task` | (3) `_Callback` y (4)/(5) |
|---|---|---|
| Navegación | `Navigation.PushAsync` / `PopAsync` | `Shell.Current.GoToAsync(nameof(...))` / `".."` con ruta registrada en `AppShell.xaml.cs` |
| Canal de vuelta | `TaskCompletionSource<Image>` público (`MyMediaPickerPage.xaml.cs:11`) que la llamadora **await**ea | `Action<string?>` pasado como `ShellNavigationQueryParameters` y recibido por `[QueryProperty]` |
| Qué viaja | un `Image` ya construido sobre `e.Media` (`:227-238`) | la **ruta de un archivo temporal** en `FileSystem.CacheDirectory` |
| `CameraView` | declarado en XAML (`x:Name="Camera"`) | creado en código y montado en `CameraContainer.Content` |
| Cancelación | `OnDisappearing` hace `TrySetCanceled()` si la tarea no se completó (`:347-350`) | el callback se invoca con `null` en `OnVolverClicked` (`:207-212`) |
| Liberación | `Camera.Handler.DisconnectHandler()` en `OnDisappearing` | desuscribe eventos y pone `CameraContainer.Content = null` |

**El patrón de (3): materializar antes de cruzar.** `OnMediaCaptured` (`:276-321`) copia `e.Media` a `photo_{guid}.jpg` en `CacheDirectory` **fuera del UI thread**, y recién entonces salta al UI thread para invocar el callback y navegar atrás. El comentario del código lo explica: así el stream del toolkit «nace y muere en su hilo natural y nunca cruza páginas». La página consumidora es la responsable de borrar el archivo (`MainPage.xaml.cs:36-44`).

**El riesgo de (2):** `ResultadoTask.TrySetResult(new Image { Source = ImageSource.FromStream(() => e.Media) })` entrega un `ImageSource` **lazy sobre el stream del toolkit**, que la página de captura está a punto de destruir — es la misma familia de problemas que describe `MEDIAPICKER_FIX.md` §El problema. Es material didáctico, no el patrón recomendado.

## 6. Normalización de la imagen (proyectos 4 y 5)

Servicio: `ImageDeviceAutoRotateService` (proyecto 4, `Services/ImageDeviceAutoRotateService.cs`) e `ImageDeviceAutoRotate` (proyecto 5, `Utilities/ImageDeviceAutoRotate.cs`) — **idénticos salvo nombre de tipo, interfaz y namespace**.

| Parámetro | Default | Efecto |
|-----------|---------|--------|
| `MaxWidthHeight` | 1000 | Techo del lado mayor; si se supera, escala por ratio |
| `CompressionQuality` | 75 | Calidad del JPEG de salida |
| `CustomPhotoSize` | 50 | Porcentaje de reducción base antes del techo |

Pipeline (`:13-65`): leer EXIF con `MetadataExtractor` (`ExifIfd0Directory` → `TagOrientation`) → `SKBitmap.Decode` → `AplicarOrientation` (los 8 valores EXIF, `:91-136`) → `Resize` con `SKSamplingOptions(Linear, Nearest)` → `Encode(Jpeg, CompressionQuality)`.

Dos sobrecargas: `ProcesarPhotoAsync(Stream)` → `byte[]`, y `ProcesarPhotoAsync(string inputPath, string? outputPath = null)` → path, que escribe `photo_norm_{guid}.jpg` en `CacheDirectory` si no se le da destino (`:73-89`).

⚠️ **Los volteos EXIF no se aplican.** La sobrecarga `Rotate(bitmap, angle, ex, ey)` (`:161-183`) llama a `canvas.Scale(ex, ey)` **después** de `canvas.DrawBitmap`, así que el espejado de los casos EXIF 2, 4, 5 y 7 no tiene efecto sobre el bitmap resultante. Los casos de rotación pura (3, 6, 8) sí funcionan.

Cableado en el proyecto (4): `IImageService` inyectado en `MainPage` por constructor y registrado como `AddSingleton<IImageService, ImageDeviceAutoRotateService>()` (`MauiProgram.cs:38`). En el (5) **no hay DI**: `MainPage.xaml.cs:26-31` hace `new ImageDeviceAutoRotate() { … }` con los parámetros literales.

## 7. La variante selfie (proyecto 5)

Diferencias verificadas contra el proyecto (4) — todo lo demás es idéntico:

| Cambio | Dónde |
|--------|-------|
| Prefiere `CameraPosition.**Front**` en vez de `Rear` | `MyMediaSelfiePickerPage.xaml.cs` (`SeleccionarCamaraAsync`) |
| Botón de flash oculto (`BtnFlashButton.IsVisible = false`) | idem, en `MostrarVisorCamara()` |
| Máscara ovalada de encuadre sobre el visor | `Utilities/SelfieMaskDrawable.cs` + `<GraphicsView x:Name="MaskOverlay">` en `MyMediaSelfiePickerPage.xaml:26` |
| Se suscribe a `Application.Current.RequestedThemeChanged` para redibujar la máscara | idem, `OnRequestedThemeChanged → MaskOverlay.Invalidate()` |
| Selecciona la cámara también desde `_cameraView.Loaded` | idem, `OnCameraViewLoaded` |

`SelfieMaskDrawable` (`:16-48`) dibuja un rectángulo con un óvalo recortado por `WindingMode.EvenOdd`; si `OverlayColor`/`BorderColor` quedan en `null` elige blanco o negro según `Application.Current.RequestedTheme`. Proporciones: `WidthFraction` 0.75 y `HeightToWidthRatio` 1.35, con tope de 0.9 de la altura visible.

⚠️ **Namespace cruzado.** En el proyecto (5), `MyMediaSelfiePickerPage` declara `namespace Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.Pages` — el del proyecto (4) — tanto en el `.xaml` (`x:Class`) como en el `.xaml.cs`. Es un resabio de copia: compila, pero obliga a `MainPage.xaml.cs:1` a importar el namespace del otro proyecto. Solo `MainPage` usa el namespace propio.

## 8. Observaciones

- **Mojibake en literales.** Los archivos de los proyectos (2)–(5) tienen los textos de UI con acentos corruptos (`C�mara`, `orientaci�n`): están guardados en una codificación distinta de UTF-8. Es visible en los mensajes de overlay y en los comentarios.
- **`Ejemplo_Photo_MiMediaPicker_Callback` y `..._Callback_Normalizacion` comparten `MyMediaPickerPage.xaml.cs` byte a byte** (salvo el namespace): toda la diferencia entre ambos está en `MainPage` y en el servicio de normalización.
- **Carpeta vs. namespace en el proyecto (4):** `IImageService.cs` e `ImageDeviceAutoRotateService.cs` viven en `Services/` pero declaran `namespace ….Utilities` (en el (5) la carpeta sí se llama `Utilities/`).
- El proyecto (1) inyecta `CamaraService` **concreto** en el constructor de `MainPage` (`:11`) mientras el contenedor registra la **interfaz**; la página se instancia desde el `DataTemplate` del `ShellContent`, no desde el contenedor.

## 9. Fuentes

| Ruta | Contenido |
|------|-----------|
| `Camera/Ejemplo_Docs_Photo/Readme.md` | Permisos de Android para MediaPicker y CameraView; arranque de `UseMauiCommunityToolkitCore` |
| `Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` | Guía conceptual: 6 técnicas de transferencia, apps reales, anti-patterns, tabla comparativa |
| `Camera/Ejemplo_Photo_MediaPicker/MEDIAPICKER_FIX.md` | El bug de `FromStream`, opciones A/B, y cómo guardan las fotos WhatsApp/Instagram |
| `Camera/*/Pages/MyMediaPickerPage.xaml{,.cs}` | El visor propio (permisos, captura, flash, orientación) |
| `Camera/*/Pages/MainPage.xaml.cs` | La pantalla consumidora y el canal de vuelta de la foto |
| `Camera/*/{Services,Utilities}/` | Servicio de cámara (1) y normalización EXIF (4)/(5); máscara selfie (5) |
