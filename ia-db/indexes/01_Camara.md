# 01 — Cámara

> **Propósito:** mapa de los cinco ejemplos de captura de foto: qué agrega cada uno al anterior, cómo vuelve la foto a la pantalla que la pidió (callback vs `Task`), la normalización EXIF, la variante selfie y los permisos por plataforma.
> **Fuentes primarias:** `Ejemplos_Devices/Camera/` (5 proyectos + `Ejemplo_Docs_Photo/`), `Ejemplos_Devices/Ejemplos_Devices.slnx:36-55`. HEAD indexado `cc80ce1`.
> CI/CD de cámara (workflows `cd-ios-camera.*`): ver [09_CI-CD-y-Build](09_CI-CD-y-Build.md). Uso de cámara/selfie dentro de la app híbrida: [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## 1. Inventario

| # | Proyecto | ApplicationId | Paquetes NuGet (además de `Microsoft.Extensions.Logging.Debug` 10.0.2) | En `.slnx` |
|---|---|---|---|---|
| 1 | `Ejemplo_Photo_MediaPicker` | `com.ejemplos.devices.mediapicker` (`…/Ejemplo_Photo_MediaPicker.csproj:18`) | `Microsoft.Maui.Controls` `$(MauiVersion)` (`:44`) | sí (`Ejemplos_Devices.slnx:37`) |
| 2 | `Ejemplo_Photo_MiMediaPicker_Callback` | `com.ejemplos.photo.mimediapicker.callback` (`:18`) | `CommunityToolkit.Maui.Camera` 6.0.0, `CommunityToolkit.Maui.Core` 14.0.0, `Microsoft.Maui.Controls` 10.0.30 (`…_Callback.csproj:76-79`) | sí (`.slnx:40`) |
| 3 | `Ejemplo_Photo_MiMediaPicker_Task` | `com.ejemplos.devices.mimediapicker.task` (`:18`) | Camera 6.0.0, Core 14.0.0, Controls 10.0.30 (`…_Task.csproj:80-83`) | sí (`.slnx:46`) |
| 4 | `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion` (`:18`) | Camera 6.0.0, Core 14.0.0, `MetadataExtractor` 2.9.0, `SkiaSharp` 3.119.1, Controls 10.0.31 (`…_Normalizacion.csproj:76-82`) | sí (`.slnx:43`) |
| 5 | `Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion.miselfie` (`:19`) | igual que #4 (`…Selfie_Callback_Normalizacion.csproj:87-93`) | sí (`.slnx:49`) |
| — | `Ejemplo_Docs_Photo` (sólo prosa) | — | — | sí, como archivo (`.slnx:53-55`) |

Común a los cinco: TFM `net10.0-android` siempre y `net10.0-ios` sólo si se compila en macOS (`…/Ejemplo_Photo_MediaPicker.csproj:4-5`, idéntico en los demás); mínimo iOS 15.0 / Android 25.0 (`:25-26`). `$(MauiVersion)` del ej. 1 no está fijado en el repo (no hay `Directory.Build.props`): lo pone el workload instalado (versión efectiva no verificada).

## 2. Escalera didáctica: qué agrega cada ejemplo

| # | Captura | Retorno a la pantalla | Qué viaja | Qué enseña / diferencia con el anterior |
|---|---|---|---|---|
| 1 | Diálogo **nativo** `MediaPicker.Default.CapturePhotoAsync()` (`Ejemplo_Photo_MediaPicker/Services/CameraService.cs:11`) | `await` directo del servicio (`Pages/MainPage.xaml.cs:20`) | `MemoryStream` copiado antes de salir (`CameraService.cs:14-20`) | Servicio + interfaz `ICamaraService` registrada en DI (`MauiProgram.cs:27-33`); bug del stream cerrado documentado en `MEDIAPICKER_FIX.md` |
| 2 | Pantalla **propia** con `CameraView` del Toolkit creada por código (`Ejemplo_Photo_MiMediaPicker_Callback/Pages/MyMediaPickerPage.xaml.cs:152-176`) | **Callback** `Action<string?>` pasado por `ShellNavigationQueryParameters` (`Pages/MainPage.xaml.cs:17-60`) | `string` con ruta a un JPG temporal en `CacheDirectory` (`MyMediaPickerPage.xaml.cs:281-303`) | Reemplaza el diálogo nativo (problemático en algunos Android, `Ejemplo_Docs_Photo/Readme.md:7-9`); overlay de permisos, flash, orientación |
| 3 | Pantalla propia con `toolkit:CameraView` declarado en XAML (`Ejemplo_Photo_MiMediaPicker_Task/Pages/MyMediaPickerPage.xaml:23`) | **`TaskCompletionSource<Image>`** + `Navigation.PushAsync` (`Pages/MainPage.xaml.cs:15-19`) | un `Image` cuyo `Source` envuelve `e.Media` (`MyMediaPickerPage.xaml.cs:235-236`) | Contraste deliberado: retorno `await`-able, pero el stream cruza páginas (ver §3 ⚠️) |
| 4 | Igual que #2 (archivo idéntico salvo namespace) | Callback, ahora `async` | ruta temporal → **normalizada** por `IImageService` (`Pages/MainPage.xaml.cs:21-34`) | Normalización EXIF + resize + JPEG con SkiaSharp (§4); limpieza de ambos temporales en `finally` (`MainPage.xaml.cs:43-55`) |
| 5 | Igual que #4 pero cámara **frontal** y máscara oval (`MyMediaSelfiePickerPage.xaml.cs:231`, `Utilities/SelfieMaskDrawable.cs`) | Callback | ruta temporal → normalizada con instancia directa, sin DI (`Pages/MainPage.xaml.cs:26-31`) | Selfie (§5); selección de cámara en `Loaded` y redibujo de máscara al cambiar tema |

```
#1 MediaPicker nativo ──► #2 CameraView propio + callback(path) ──► #4 + normalización EXIF ──► #5 + selfie/máscara
                                   └──► #3 CameraView propio + Task (contraste, stream directo)
```

## 3. Flujo de captura y retorno

### 3.1 Callback (ej. 2, 4, 5)

```
MainPage.OnAbrirCamaraClicked
  ├─ arma Action<string?> resultadoCallback
  └─ Shell.Current.GoToAsync("MyMediaPickerPage", {OnPhotoCallback: callback})
        MyMediaPickerPage  [QueryProperty OnPhotoCallback]
          OnAppearing → EvaluarYMostrarEstadoPermisoAsync → MostrarVisorCamara (crea CameraView)
          OnTomarFotoClicked → CameraView.CaptureImage(ct)
          OnMediaCaptured(e.Media)
            1) copia e.Media → CacheDirectory/photo_<guid>.jpg   (fuera del UI thread)
            2) MainThread: OnPhotoCallback(path); GoToAsync("..")
          OnVolverClicked → OnPhotoCallback(null); GoToAsync("..")
  callback(path): lee bytes → borra temporal → Dispatcher.Dispatch(ImgPhoto.Source = FromStream(new MemoryStream(bytes)))
```

| Paso | Cita (ej. 2) |
|---|---|
| Ruta registrada en Shell | `Ejemplo_Photo_MiMediaPicker_Callback/AppShell.xaml.cs:11` |
| Recepción del callback | `Pages/MyMediaPickerPage.xaml.cs:7,14` |
| Materializar a archivo, limpiar si falla | `Pages/MyMediaPickerPage.xaml.cs:276-304` |
| Invocar callback y volver | `Pages/MyMediaPickerPage.xaml.cs:309-320` |
| Cancelación = callback(null) | `Pages/MyMediaPickerPage.xaml.cs:207-212` |
| Consumidor: leer, borrar, mostrar con `MemoryStream` nuevo por evaluación | `Pages/MainPage.xaml.cs:17-53` |
| Botón deshabilitado durante la operación | `Pages/MainPage.xaml.cs:13,66-69` |

Razón del diseño (el stream del Toolkit «nace y muere en su hilo y nunca cruza páginas»): comentario en `Pages/MyMediaPickerPage.xaml.cs:278-280`.

### 3.2 Task (ej. 3)

| Paso | Cita |
|---|---|
| `new MyMediaPickerPage()` + `Navigation.PushAsync` + `await ResultadoTask.Task` | `Ejemplo_Photo_MiMediaPicker_Task/Pages/MainPage.xaml.cs:15-19` |
| `TaskCompletionSource<Image> ResultadoTask` | `Pages/MyMediaPickerPage.xaml.cs:11` |
| `TrySetResult(new Image{Source=FromStream(() => e.Media)})` + `PopAsync` | `Pages/MyMediaPickerPage.xaml.cs:231-238` |
| «Back» sin foto → `TrySetCanceled()` en `OnDisappearing` | `Pages/MyMediaPickerPage.xaml.cs:347-349` |
| Libera el handler de la cámara | `Pages/MyMediaPickerPage.xaml.cs:343` |

- ⚠️ `ImageSource.FromStream(() => e.Media)` reutiliza el stream del Toolkit en otra página: es exactamente la carga diferida que `MEDIAPICKER_FIX.md:17-28` describe como origen de `ObjectDisposedException`. La prosa de la pieza lo trata como anti-patrón deliberado de contraste.
- ⚠️ `TrySetCanceled()` hace que el `await` de `MainPage` lance `TaskCanceledException`, que cae en un `catch` vacío (`Pages/MainPage.xaml.cs:30-32`): el aviso «Cancelado» (`:27`) sólo se mostraría con un `Image` nulo, que el productor nunca entrega.
- La ruta Shell se registra (`AppShell.xaml.cs:11`) pero la navegación usa `Navigation.PushAsync`, no `GoToAsync`.

### 3.3 Ej. 1: servicio sobre `MediaPicker`

- `TomarFotoAsync` / `ElegirDeGaleriaAsync` en el hilo principal, copian a `MemoryStream` porque el temporal «puede limpiarse apenas termine el camera intent» (`Services/CameraService.cs:6-35`). La UI sólo usa `TomarFotoAsync` (`Pages/MainPage.xaml.cs:17-25`); `ElegirDeGaleriaAsync` no tiene botón.
- `MEDIAPICKER_FIX.md` compara Opción A `FromFile(photo.FullPath)` vs Opción B `MemoryStream` (`:34-103`) y cómo las apps persisten en `MediaStore` (`:107-142`). El código viejo queda comentado en `Pages/MainPage.xaml.cs:28-58`.

## 4. Normalización EXIF (ej. 4 y 5)

Servicio `ImageDeviceAutoRotateService : IImageService` (ej. 4, singleton en `MauiProgram.cs:35-41`) / `ImageDeviceAutoRotate : IImageDevice` (ej. 5, idéntico salvo nombres, sin DI).

| Etapa | Cita (ej. 4, `Services/ImageDeviceAutoRotateService.cs`) |
|---|---|
| Parámetros: `MaxWidthHeight`=1000, `CompressionQuality`=75, `CustomPhotoSize`=50 % | `:9-11` |
| Leer tag `Orientation` de `ExifIfd0Directory` (MetadataExtractor) | `:22-28` |
| `SKBitmap.Decode` + `AplicarOrientation` | `:34-36` |
| Escala: 50 % por defecto, o encaje en 1000 px si algún lado lo supera | `:39-48` |
| `Resize` + `Encode(Jpeg, 75)` → `byte[]` | `:52-58` |
| Sobrecarga `path → path` (escribe `photo_norm_<guid>.jpg` en `CacheDirectory`) | `:73-89` |
| Tabla de orientaciones EXIF 1–8 → rotación/espejo | `:91-136` |

Contrato: `Services/IImageService.cs:11` (`Stream → byte[]?`) y `:19` (`path → path`). Prosa del patrón de sobrecargas: `Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` §6 (`:377-507`).

- ⚠️ En `Rotate(bitmap, angle, ex, ey)` el `canvas.Scale(ex, ey)` se aplica **después** de `DrawBitmap` (`Services/ImageDeviceAutoRotateService.cs:178-180`; igual en ej. 5 `Utilities/ImageDeviceAutoRotate.cs:180`), así que los casos con espejo (2, 4, 5, 7) no espejan: sólo rotan.
- ⚠️ Carpeta `Services/` pero namespace `…_Normalizacion.Utilities` (`Services/IImageService.cs:1`, `Services/ImageDeviceAutoRotateService.cs:5`).
- El callback del consumidor es una lambda `async` asignada a `Action<string?>` (async void) (`Pages/MainPage.xaml.cs:21`).

## 5. Selfie (ej. 5)

| Aspecto | Cita (`Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/`) |
|---|---|
| Cámara frontal (fallback a la primera disponible) | `Pages/MyMediaSelfiePickerPage.xaml.cs:231` |
| Selección al `Loaded` del `CameraView` (los ej. 2/4 sólo la eligen en `OnNavigatedTo` si la vista ya existe) | `Pages/MyMediaSelfiePickerPage.xaml.cs:170,219-222` |
| Máscara: `GraphicsView` `InputTransparent` sobre el contenedor de cámara | `Pages/MyMediaSelfiePickerPage.xaml:24-27` |
| Óvalo recortado: rectángulo + elipse con `WindingMode.EvenOdd` | `Utilities/SelfieMaskDrawable.cs:37-41` |
| Color según tema (blanco Light / negro Dark) e invalidación al cambiar tema | `Utilities/SelfieMaskDrawable.cs:22-24`; `Pages/MyMediaSelfiePickerPage.xaml.cs:43-51` |

- La foto no se espeja (no hay código de mirror). La normalización es la misma que el ej. 4.
- ⚠️ `MyMediaSelfiePickerPage` vive en el namespace del ej. 4 (`Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.Pages`: `Pages/MyMediaSelfiePickerPage.xaml:5`, `.xaml.cs:5`) y `MainPage` lo importa con un `using` de ese nombre (`Pages/MainPage.xaml.cs:1`). Compila porque es el mismo ensamblado; es resto de copia.

## 6. Pantalla propia de cámara (ej. 2–5): permisos y overlay

Lógica en `EvaluarYMostrarEstadoPermisoAsync` (ej. 2: `Pages/MyMediaPickerPage.xaml.cs:86-148`):

```
[iOS simulador] → overlay "no disponible" (sin reintento)            :89-98
CheckStatus(Camera)==Granted → IsCaptureSupported ? visor : overlay   :100-111
RequestAsync(Camera)==Granted → idem                                  :113-124
Restricted → overlay sin reintento                                    :126-132
[Android] ShouldShowRationale → "Pedir permiso" | si no "Abrir configuración" (AppInfo.ShowSettingsUI)  :134-147, :202-205
```

- El overlay es un `Grid` en XAML (`PermissionDeniedOverlay`), no un `ContentView`+VM: `Pages/MyMediaPickerPage.xaml:25-43`. No sigue el patrón servicio→resultado tipado→overlay MVVM del teléfono ([06_Telefonia](06_Telefonia.md) §3); la prosa de arquitectura lo explica como límite del patrón (la cámara hospeda contenido).
- Ej. 3 difiere: acepta `Limited`, pide el permiso siempre que no sea `Granted`/`Restricted`, sin guarda de simulador ni `IsCaptureSupported` (`Ejemplo_Photo_MiMediaPicker_Task/Pages/MyMediaPickerPage.xaml.cs:64,92-107`).
- Flash Off→On→Auto y layout por orientación (filas/columnas reasignadas): `Ejemplo_Photo_MiMediaPicker_Callback/Pages/MyMediaPickerPage.xaml.cs:336-409`.
- Registro del Toolkit: `.UseMauiCommunityToolkitCore().UseMauiCommunityToolkitCamera()` (`Ejemplo_Photo_MiMediaPicker_Callback/MauiProgram.cs:15-16`).
- ⚠️ Los textos de overlay de los ej. 2, 4 y 5 contienen el carácter de reemplazo U+FFFD (p. ej. `"C�mara no disponible"`, `Ejemplo_Photo_MiMediaPicker_Callback/Pages/MyMediaPickerPage.xaml.cs:93`): 19 apariciones en ej. 2 y 4, 10 en ej. 5. El `.xaml.cs` del ej. 3 está en codificación no UTF-8.

## 7. Permisos por plataforma

### 7.1 Android (`Platforms/Android/AndroidManifest.xml`)

| Ej. | `CAMERA` | `RECORD_AUDIO` | Otros | `<queries>` `IMAGE_CAPTURE` | `uses-feature camera` |
|---|---|---|---|---|---|
| 1 | `:14` | `:15` | `WRITE_EXTERNAL_STORAGE` maxSdk 32 `:16` | no | `required="true"` + autofocus `:19-20` |
| 2 | `:13` | — | — | **no** | `required="false"` `:14` |
| 3 | `:13` | `:14` | — | `:16-20` | — |
| 4 | `:13` | `:14` | — | `:16-20` | — |
| 5 | `:13` | `:14` | — | `:16-20` | — |

Todos declaran `ACCESS_NETWORK_STATE`/`INTERNET` (plantilla) y `minSdk 25 / targetSdk 36`.

- ⚠️ Ej. 2 consulta `MediaPicker.Default.IsCaptureSupported` (`Pages/MyMediaPickerPage.xaml.cs:104,117`) sin declarar el intent `IMAGE_CAPTURE` en `<queries>` (posible falso negativo en Android 11+; registrado como B-02 en el GAP-REPORT).
- `Ejemplo_Docs_Photo/Readme.md:17-31` lista además `READ_MEDIA_*` para elegir de galería; ningún manifest de los ejemplos los declara.

### 7.2 iOS (`Platforms/iOS/Info.plist`)

| Ej. | Claves | Cita |
|---|---|---|
| 1 | `NSCameraUsageDescription`, `NSMicrophoneUsageDescription` | `:59-63` |
| 2 | ídem | `:59-63` |
| 3 | `NSCameraUsageDescription`, `NSMicrophoneUsageDescription` (texto: «does not require…»), `NSPhotoLibraryAddUsageDescription` | `:59-66` |
| 4 | `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSPhotoLibraryAddUsageDescription` | `:59-64` |
| 5 | ídem ej. 4 | `:59-64` |

- ⚠️ Ej. 1 pide galería por `PickPhotosAsync` (`Services/CameraService.cs:26`) sin `NSPhotoLibraryUsageDescription`; no se dispara porque la UI no lo usa.

## 8. Avisos de lectura (⚠️) adicionales

- ⚠️ Ej. 1: `MainPage` pide `CamaraService` **concreto** por constructor (`Pages/MainPage.xaml.cs:9-11`) pero DI registra `ICamaraService` (`MauiProgram.cs:29`) y `MainPage` no está registrada; además `using Android.Media;` sin `#if ANDROID` (`Pages/MainPage.xaml.cs:1`) rompe el TFM iOS. Resolución en ejecución no verificada.
- ⚠️ Ej. 4: `MainPage(IImageService)` (`Pages/MainPage.xaml.cs:9-13`) tampoco está registrada en DI; que Shell la construya vía `DataTemplate` no está verificado.
- ⚠️ `Ejemplo_Docs_Photo/Readme.md:4-9` dice «Hay dos ejemplos»; hay cinco. El archivo está en ISO-8859.
- Duplicación entre ej. 2/4/5 (misma página de cámara) es deliberada (un proyecto por técnica, ADR-0001).

## 9. Fuentes

| Ruta | Contiene |
|---|---|
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MediaPicker/Services/CameraService.cs` | Servicio sobre `MediaPicker` con copia a memoria |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MediaPicker/MEDIAPICKER_FIX.md` | Bug del stream cerrado (Glide), opciones A/B, `MediaStore` |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Callback/Pages/MyMediaPickerPage.xaml(.cs)` | Página de cámara propia: permisos, overlay, flash, orientación, productor callback |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Task/Pages/*` | Variante `TaskCompletionSource` |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/Services/*` | Normalización EXIF + resize + JPEG |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/README.md` | Guía rápida: NuGet, registro DI, uso en página |
| `Ejemplos_Devices/Camera/Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/Utilities/SelfieMaskDrawable.cs` | Máscara oval |
| `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/Readme.md` | Introducción, permisos de `MediaPicker`, instalación del Toolkit |
| `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` | Técnicas de transferencia (Stream/byte[]/path/singleton/messenger), EXIF, `Dispatcher`/`MainThread`, base64 vs multipart, anti-patrones, recomendación (§1–§12) |

## 10. Prosa relacionada

- [Pieza camera](../../Ejemplos_Maui_Devices-docs/docs/pieces/camera/README.md) — estructura, cómo ejecutar, permisos, snippets canónicos por ejemplo y preguntas guía.
- [ADR-0002 servicio tipado + overlay MVVM](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0002-servicio-tipado-overlay-mvvm.md) — patrón del que la cámara queda parcialmente afuera.
- [07-overlays-dispositivos](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/07-overlays-dispositivos.md) — límites del patrón overlay (§7) y deriva de cámara/QR (§8.7).
- [GLOSSARY](../../Ejemplos_Maui_Devices-docs/GLOSSARY.md) — «Normalización EXIF», «Transferencia por callback / por Task».
- [GAP-REPORT](../../Ejemplos_Maui_Devices-docs/GAP-REPORT.md) — B-02 y B-03 (cámara).
