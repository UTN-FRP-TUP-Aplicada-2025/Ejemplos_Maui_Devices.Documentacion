# 01 — Cámara

> **Propósito**: cómo se toma una foto y cómo vuelve a la pantalla anterior en los cinco proyectos de `Ejemplos_Devices/Camera/`. Cubre `MediaPicker` nativo, la página propia con `CameraView` (por *callback* y por `Task`), la normalización EXIF/rotación con SkiaSharp, la selfie con cámara frontal y máscara, y los permisos. También mapea la carpeta `Ejemplo_Docs_Photo`.
> **Fuente primaria**: `Ejemplos_Devices/Camera/**` (commit `285f6fb`). Las rutas son relativas a la raíz del repo de código `Ejemplos_Maui_Devices`.
> **Índices relacionados**: [00_MASTER-INDEX](00_MASTER-INDEX.md) · [02_QR](02_QR.md) (otro uso de la cámara) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (copia de estas páginas en `LibApp/Devices/Camera` e `Images`) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) (workflows `cd-ios-camera.*` y `scripts/Ejemplo_Photo_MediaPicker_launch.bat`) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md)

---

## 1. Mapa del dominio

```
Ejemplos_Devices/Camera/
├── Ejemplo_Docs_Photo/                              ← prosa, sin código (§10)
│   ├── Readme.md                                    (ISO-8859-1; enlazado desde README.md:14)
│   └── Transferencia_Foto_Entre_Pantallas.md        (guía conceptual, 762 líneas)
├── Ejemplo_Photo_MediaPicker/                       ← A. diálogo nativo (MediaPicker)
│   ├── Services/{ICamaraService.cs, CameraService.cs}
│   └── MEDIAPICKER_FIX.md                           (bug del Stream cerrado + opciones A/B)
├── Ejemplo_Photo_MiMediaPicker_Callback/            ← B. página propia + Action<string?>
├── Ejemplo_Photo_MiMediaPicker_Task/                ← C. página propia + TaskCompletionSource<Image>
├── Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/ ← D. = B + EXIF/resize (servicio por DI)
│   ├── Services/{IImageService.cs, ImageDeviceAutoRotateService.cs}
│   └── README.md                                    (guía de uso rápido)
└── Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/ ← E. = D + cámara frontal + máscara
    └── Utilities/{IImageDevice.cs, ImageDeviceAutoRotate.cs, SelfieMaskDrawable.cs}
```

Los cinco proyectos están en la solución, carpeta `/Ejemplos/Camera/` (`Ejemplos_Devices/Ejemplos_Devices.slnx:36-52`); de `Ejemplo_Docs_Photo` sólo entra `Readme.md` (`:53-55`). Cada proyecto es una app MAUI independiente (Shell con una `MainPage`), con su propia copia de la página de captura: la duplicación es deliberada (cada ejemplo se lee solo).

## 2. Comparativa de los cinco proyectos

| | A. `MediaPicker` | B. `MiMediaPicker_Callback` | C. `MiMediaPicker_Task` | D. `..._Callback_Normalizacion` | E. `MiMediaSelfie_..._Normalizacion` |
|---|---|---|---|---|---|
| Captura | `MediaPicker.Default.CapturePhotoAsync()` (UI del SO) | `CameraView` (CommunityToolkit) en `MyMediaPickerPage` | `CameraView` en `MyMediaPickerPage` | igual que B | `CameraView` en `MyMediaSelfiePickerPage` |
| `CameraView` creado | — | en código, tras el permiso | en XAML, siempre | en código | en código |
| Cámara elegida | la del SO | trasera ?? primera | sólo trasera, sin alternativa | trasera ?? primera | **frontal** ?? primera |
| Retorno a `MainPage` | `Task<Stream?>` de un servicio | `Action<string?>` por `ShellNavigationQueryParameters` | `TaskCompletionSource<Image>` + `PushAsync` | `Action<string?>` (lambda `async`) | `Action<string?>` (lambda `async`) |
| Qué viaja | `MemoryStream` | ruta a JPEG en `CacheDirectory` | `Image` con `FromStream(() => e.Media)` | ruta → procesada → ruta | ruta → procesada → ruta |
| Normalización EXIF | no | no | no | `IImageService` (DI singleton) | `new ImageDeviceAutoRotate{…}` (sin DI) |
| Flash | — | sí (Off→On→Auto) | sí | sí | oculto (`IsVisible = false`) |
| NuGet extra | — | Camera 6.0.0, Core 14.0.0 | igual que B | B + `MetadataExtractor` 2.9.0, `SkiaSharp` 3.119.1 | igual que D |
| `Microsoft.Maui.Controls` | `$(MauiVersion)` | 10.0.30 | 10.0.30 | 10.0.31 | 10.0.31 |
| `ApplicationId` | `com.ejemplos.devices.mediapicker` | `com.ejemplos.photo.mimediapicker.callback` | `com.ejemplos.devices.mimediapicker.task` | `com.ejemplos.devices.imagen.callback.normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion.miselfie` |
| Workflow iOS | no hay | `cd-ios-camera.<proyecto>.yml` | sí | sí | sí |

Base común a los cinco `.csproj`: `net10.0-android` (más `net10.0-ios` sólo si se compila en macOS), `SupportedOSPlatformVersion` Android 25 / iOS 15, `AndroidLdFlags` `-Wl,-z,max-page-size=16384` y cuatro RID Android (p. ej. `Ejemplo_Photo_MiMediaPicker_Callback/Ejemplo_Photo_MiMediaPicker_Callback.csproj:4-5,25-26,31-34`). B a E agregan APK en Debug/Release y AOT con intérprete en iOS Release (`…Callback.csproj:36-62`). E es el único con `<MauiXamlInflator>SourceGen</MauiXamlInflator>` (`Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion.csproj:15`).

Evolución didáctica: **A** (el SO hace todo) → **B** (página propia y retorno por callback) ↔ **C** (la misma página con retorno awaitable) → **D** (B más el pipeline de procesamiento) → **E** (D adaptado a selfie).

`Pages/MyMediaPickerPage.xaml.cs` de B y de D son idénticos salvo la línea 5 (namespace): la diferencia entre ambos está en `MainPage` y en el servicio de §8.

## 3. A — `Ejemplo_Photo_MediaPicker` (diálogo nativo)

```
MainPage.OnAbrirCamaraClicked ──► CamaraService.TomarFotoAsync()
                                    └─ MainThread.InvokeOnMainThreadAsync(
                                         IsCaptureSupported? → CapturePhotoAsync()
                                         → OpenReadAsync → CopyToAsync(MemoryStream) → Position=0)
ImgPhoto.Source = ImageSource.FromStream(() => stream)
```

| Pieza | Ubicación |
|---|---|
| Registro DI `ICamaraService → CamaraService` (singleton) | `Ejemplo_Photo_MediaPicker/MauiProgram.cs:27-33` |
| Contrato `TomarFotoAsync` / `ElegirDeGaleriaAsync` | `Ejemplo_Photo_MediaPicker/Services/ICamaraService.cs:7-11` |
| Captura y copia a memoria («el archivo temporal puede limpiarse») | `Ejemplo_Photo_MediaPicker/Services/CameraService.cs:6-21` |
| Galería con `PickPhotosAsync()` (sin usar desde la UI) | `Ejemplo_Photo_MediaPicker/Services/CameraService.cs:23-35` |
| Handler activo | `Ejemplo_Photo_MediaPicker/Pages/MainPage.xaml.cs:17-25` |
| Versión anterior comentada, con opción A (`FromFile`), opción B (`MemoryStream`) y el código con el bug | `Ejemplo_Photo_MediaPicker/Pages/MainPage.xaml.cs:28-58` |

- ⚠️ El constructor pide la clase concreta `MainPage(CamaraService camaraService)` (`Pages/MainPage.xaml.cs:11`), pero el contenedor registra la interfaz `ICamaraService` (`MauiProgram.cs:29`) y `MainPage` no está registrada. La página se instancia desde `ShellContent ContentTemplate` (`AppShell.xaml:10`). Cómo resuelve Shell esa dependencia: **no verificado** en ejecución.
- ⚠️ `using Android.Media;` sin `#if ANDROID` (`Pages/MainPage.xaml.cs:1`). Es el único proyecto de la carpeta sin workflow iOS; el efecto sobre un build iOS está **no verificado**.
- ⚠️ `FromStream(() => stream)` devuelve siempre **el mismo** `MemoryStream` (`Pages/MainPage.xaml.cs:22-24`). La guía advierte que un stream sólo se lee una vez si MAUI reevalúa el `ImageSource` (`Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md:129-130`). `MEDIAPICKER_FIX.md` propone `() => new MemoryStream(imageBytes)` (`:99`).
- El archivo se llama `CameraService.cs`, pero la clase es `CamaraService`.
- No pide permisos explícitamente: delega en `MediaPicker`. Manifest y plist en §7.

### 3.1 `MEDIAPICKER_FIX.md` (diagnóstico de la cátedra)

| Sección | Contenido |
|---|---|
| `:3-30` | Causa raíz por logcat: `ObjectDisposedException` en `GlideInputStreamModelLoader`. `FromStream` es diferido y el `using` cierra el `FileStream` antes de que Glide lo lea. Se nota en Debug y no siempre en Release. |
| `:34-65` | **Opción A**: `ImageSource.FromFile(photo.FullPath)`. Glide maneja su propio stream. |
| `:69-103` | **Opción B**: copiar a `byte[]` y usar `FromStream(() => new MemoryStream(bytes))`. Riesgo de OOM. |
| `:107-142` | Cómo guardan en galería WhatsApp e Instagram (`MediaStore`). Cierra recomendando la opción A. |

⚠️ El documento recomienda la **opción A** (`:142`), pero el código activo usa una tercera variante: un servicio que devuelve `MemoryStream`, más cerca de la B y con el reuso del mismo stream señalado arriba. Las opciones A y B sólo están en comentarios.

## 4. B — `Ejemplo_Photo_MiMediaPicker_Callback` (página propia + callback)

Es la implementación de referencia de la guía (`Transferencia_Foto_Entre_Pantallas.md:718-739`).

```
MainPage                                        MyMediaPickerPage  [QueryProperty OnPhotoCallback]
 BtnPhoto.IsEnabled=false                        OnAppearing → EvaluarYMostrarEstadoPermisoAsync (§6)
 Action<string?> cb = path => {…}                MostrarVisorCamara → new CameraView (+MediaCaptured)
 GoToAsync(nameof(MyMediaPickerPage),            OnTomarFotoClicked → CaptureImage(token) en UI thread
   {"OnPhotoCallback", cb}) ───────────────────► OnMediaCaptured (hilo de la cámara):
                                                   e.Media → File.Create(Cache/photo_<guid>.jpg)
                                                 InvokeOnMainThreadAsync:
 cb(path):  ◄──────────────────────────────────    IsAvailable? cb(tempPath); GoToAsync("..")
   bytes = File.ReadAllBytes(path)               OnVolverClicked → cb(null); GoToAsync("..")
   finally File.Delete(path)
   Dispatcher.Dispatch(ImgPhoto.Source =
     FromStream(() => new MemoryStream(bytes)))
```

| Pieza | Ubicación (`Ejemplo_Photo_MiMediaPicker_Callback/…`) |
|---|---|
| Ruta registrada | `AppShell.xaml.cs:11` |
| Callback del consumidor: leer, borrar y mostrar desde memoria | `Pages/MainPage.xaml.cs:17-53` |
| Navegación con `ShellNavigationQueryParameters` | `Pages/MainPage.xaml.cs:55-60` |
| `[QueryProperty]` + `Action<string?>? OnPhotoCallback` | `Pages/MyMediaPickerPage.xaml.cs:7,14` |
| Limpieza en `OnDisappearing`: desuscribe, saca el `CameraView` y libera la orientación en Android | `Pages/MyMediaPickerPage.xaml.cs:46-73` |
| Reselecciona la cámara en `OnNavigatedTo` si ya hay permiso | `Pages/MyMediaPickerPage.xaml.cs:75-82` |
| `CameraView` creado en código dentro de `CameraContainer` | `Pages/MyMediaPickerPage.xaml.cs:152-176`, `Pages/MyMediaPickerPage.xaml:20-22` |
| Selección trasera ?? primera | `Pages/MyMediaPickerPage.xaml.cs:216-237` |
| Captura: guardia de reentrada, `CancellationTokenSource`, `InvalidCastException` como «no soportada» | `Pages/MyMediaPickerPage.xaml.cs:239-274` |
| Materializa a archivo y luego salta al UI thread | `Pages/MyMediaPickerPage.xaml.cs:276-321` |
| Flash Off→On→Auto con ícono bindeado (`FlashIcon`) | `Pages/MyMediaPickerPage.xaml.cs:16-28,336-360` |
| Layout rotable: reubica en el `Grid` según `DisplayOrientation` | `Pages/MyMediaPickerPage.xaml.cs:362-409` |

Contrato del callback: `null`/vacío = cancelado (`MainPage.xaml.cs:19-21`, `MyMediaPickerPage.xaml.cs:209-210`). El consumidor es dueño del archivo temporal y lo borra en `finally` (`MainPage.xaml.cs:34-44`). Si hay excepción al copiar, el productor borra el archivo a medio escribir y pasa `null` (`MyMediaPickerPage.xaml.cs:294-303`).

⚠️ Si el `CameraView` no está `IsAvailable`, se muestra la alerta y **no** se invoca el callback ni se vuelve (`MyMediaPickerPage.xaml.cs:311-319`). Si ya se había escrito un temporal, ese archivo queda en caché.

## 5. C — `Ejemplo_Photo_MiMediaPicker_Task` (página propia + `TaskCompletionSource`)

| Pieza | Ubicación (`Ejemplo_Photo_MiMediaPicker_Task/…`) |
|---|---|
| `new MyMediaPickerPage()` + `Navigation.PushAsync` + `await destinoPage.ResultadoTask.Task` | `Pages/MainPage.xaml.cs:15-24` |
| `TaskCompletionSource<Image> ResultadoTask` | `Pages/MyMediaPickerPage.xaml.cs:11` |
| `CameraView` declarado en XAML (`x:Name="Camera"`, eventos en XAML) | `Pages/MyMediaPickerPage.xaml:23-27` |
| Permisos en dos pasos (`Evaluar…` / `PedirPermisoYActualizarVistaAsync`) que aceptan `Limited` | `Pages/MyMediaPickerPage.xaml.cs:60-119` |
| Sólo cámara trasera, sin alternativa | `Pages/MyMediaPickerPage.xaml.cs:183-196` |
| Resultado: `new Image { Source = FromStream(() => e.Media) }` → `TrySetResult` → `PopAsync` | `Pages/MyMediaPickerPage.xaml.cs:227-245` |
| Back del sistema: `TrySetCanceled()` si la tarea sigue pendiente, y `Handler.DisconnectHandler()` | `Pages/MyMediaPickerPage.xaml.cs:332-364` |

- ⚠️ Es justo la técnica que la guía desaconseja: «Pasar `Stream` directo… nunca para cruzar páginas» (`Transferencia_Foto_Entre_Pantallas.md:116-133` y tabla de anti-patrones `:692`). El `Stream` del toolkit viaja dentro de un `Image` hacia otra página. Sirve como contraejemplo.
- ⚠️ La alerta «Cancelado / No se recibió ningún dato» (`MainPage.xaml.cs:25-28`) nunca se muestra: el `TCS` sólo recibe un `Image` no nulo (`MyMediaPickerPage.xaml.cs:233-236`), y la cancelación lanza `TaskCanceledException`, que se traga el `catch` vacío (`MainPage.xaml.cs:30-32`).
- `OnVolverClicked` cancela la captura en curso, libera la orientación en Android y hace `PopAsync` (`:165-180`); no toca el `TCS`: la cancelación llega por `OnDisappearing`. El botón «Pedir permiso» del overlay llama directo a `PedirPermisoYActualizarVistaAsync` (`:155-158`).
- Detalles de borrador: `base.OnDisappearing()` se llama dos veces (`:334` y `:363`); la ruta se registra en `AppShell.xaml.cs:11` pero no se usa (se navega con `PushAsync`). El `.csproj` tiene un `None Update` que duplica el `MauiXaml Update` de la misma página (`Ejemplo_Photo_MiMediaPicker_Task.csproj:87-97`; D repite el mismo par en `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.csproj:86-97`) y carga tres fuentes Font Awesome que la cámara no usa (`MauiProgram.cs:20-22`).
- ⚠️ El archivo `Pages/MyMediaPickerPage.xaml.cs` está codificado en Windows-1252 (los otros cuatro proyectos usan UTF-8). Cómo lo decodifica el compilador: **no verificado**.

## 6. Permisos de cámara en tiempo de ejecución

Flujo de B, D y E (`Ejemplo_Photo_MiMediaPicker_Callback/Pages/MyMediaPickerPage.xaml.cs:86-148`):

```
#if IOS && DeviceType.Virtual → overlay «no disponible en el simulador» (sin reintento)   :89-98
CheckStatusAsync<Camera>
 ├─ Granted → IsCaptureSupported ? MostrarVisorCamara : overlay «sin cámara»           :100-111
 └─ RequestAsync<Camera>
     ├─ Granted → (ídem)                                                                 :113-124
     ├─ Restricted → overlay «política del dispositivo» (sin reintento)                 :126-132
     └─ Denied → #if ANDROID puedeReintentar = ShouldShowRationale<Camera>()           :134-138
                 overlay: reintento → «Pedir permiso» ; si no → «Abrir configuración»   :140-147
```

| Botón del overlay (`MyMediaPickerPage.xaml:25-43`) | Acción |
|---|---|
| Pedir permiso (visible si `puedeReintentar`) | vuelve a correr `EvaluarYMostrarEstadoPermisoAsync` (`.xaml.cs:197-200`) |
| Abrir configuración | `AppInfo.ShowSettingsUI()` (`.xaml.cs:202-205`) |
| Volver | `OnPhotoCallback(null)` + `GoToAsync("..")` (`.xaml.cs:207-212`) |

Diferencias: C no tiene guardia de simulador ni chequeo de `IsCaptureSupported`, y acepta `Limited` (`Ejemplo_Photo_MiMediaPicker_Task/Pages/MyMediaPickerPage.xaml.cs:64,94`). Sus comentarios explican que en Android una instalación nueva devuelve `Denied` y no `Unknown` (`:80-83`), y qué devuelve `ShouldShowRationale` (`:105-106`). A no pide permisos: lo resuelve `MediaPicker`.

⚠️ **Texto visible roto.** En B, D y E los literales de `MyMediaPickerPage.xaml.cs` / `MyMediaSelfiePickerPage.xaml.cs` tienen el carácter U+FFFD guardado en el archivo («C�mara no disponible», «est�», «Pod�s»): 19 líneas afectadas en B, 19 en D y 10 en E (27, 27 y 19 caracteres; algunas son comentarios, p. ej. `// C�MARA`). El usuario ve esos títulos y mensajes del overlay (p. ej. `Ejemplo_Photo_MiMediaPicker_Callback/Pages/MyMediaPickerPage.xaml.cs:93-94,107,129`). E corrigió algunos (`Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/Pages/MyMediaSelfiePickerPage.xaml.cs`): `:131` y `:265` del todo; `:120` y `:236` a medias (en una línea el título está bien y el mensaje roto, en la otra al revés). Siguen rotos `:106-107,137,147-148,318`. El XAML (`Text="Sin acceso a la cámara"`) sí está bien.

## 7. Manifiestos de plataforma

| Proyecto | AndroidManifest (permisos/features) | `<queries>` `IMAGE_CAPTURE` | Info.plist (iOS) |
|---|---|---|---|
| A | `CAMERA`, `RECORD_AUDIO`, `WRITE_EXTERNAL_STORAGE` (máx. SDK 32); `camera` y `camera.autofocus` **required=true** (`Platforms/Android/AndroidManifest.xml:13-20`) | no | `NSCameraUsageDescription`, `NSMicrophoneUsageDescription` (`Platforms/iOS/Info.plist:59-63`) |
| B | `CAMERA`; `camera` required=false (`:13-14`) | no | cámara + micrófono |
| C | `CAMERA`, `RECORD_AUDIO` | sí | cámara («adjuntar fotos a los formularios»), micrófono («does not require»), `NSPhotoLibraryAddUsageDescription` |
| D | `CAMERA`, `RECORD_AUDIO` | sí | `NSCameraUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSPhotoLibraryAddUsageDescription` (en inglés) |
| E | `CAMERA`, `RECORD_AUDIO` (`Platforms/Android/AndroidManifest.xml:13-14`) | sí (`:16-20`) | igual que D |

Todos declaran `INTERNET`, `ACCESS_NETWORK_STATE` y `uses-sdk min 25 / target 36`. Los `MainActivity` difieren sólo en formato y namespace: mismos atributos (`LaunchMode.SingleTop`, `ConfigurationChanges` con `Orientation`).
⚠️ Los `Entitlements.plist` y `Entitlements.Development.plist` de los cinco declaran `aps-environment=development` (notificaciones push), algo que la cámara no usa. Es plantilla compartida; la firma está en [09_CI-CD-y-Build](09_CI-CD-y-Build.md).
⚠️ En E, `CFBundleName` quedó con el nombre de D, `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` (`Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/Platforms/iOS/Info.plist:40-41`). `CFBundleIdentifier` sí es el propio.

## 8. Normalización EXIF/rotación/tamaño (D y E)

```
path (Cache/photo_<guid>.jpg)
  └─ ProcesarPhotoAsync(string inputPath, string? outputPath=null)        path→path
       └─ ProcesarPhotoAsync(Stream)                                       Stream→byte[]
            1. MetadataExtractor: ExifIfd0Directory → TagOrientation
            2. SKBitmap.Decode → AplicarOrientation(1..8)
            3. resize (MaxWidthHeight / CustomPhotoSize)
            4. SKImage.Encode(Jpeg, CompressionQuality)
       └─ File.WriteAllBytesAsync(Cache/photo_norm_<guid>.jpg) → outPath
```

| Pieza | D (`…MiMediaPicker_Callback_Normalizacion/Services/`) | E (`…MiMediaSelfie_Callback_Normalizacion/Utilities/`) |
|---|---|---|
| Interfaz | `IImageService.cs:3-20` | `IImageDevice.cs:3-20` (misma firma) |
| Implementación | `ImageDeviceAutoRotateService.cs` | `ImageDeviceAutoRotate.cs` (difiere sólo en el namespace y en los nombres de clase e interfaz) |
| Cómo se obtiene | DI singleton (`MauiProgram.cs:35-41`) inyectado en `MainPage(IImageService)` (`Pages/MainPage.xaml.cs:9`) | `new ImageDeviceAutoRotate { MaxWidthHeight=1000, CompressionQuality=75, CustomPhotoSize=50 }` (`Pages/MainPage.xaml.cs:26-31`) |

Detalle de `ImageDeviceAutoRotateService.cs` (los números de línea valen para los dos archivos):

| Líneas | Qué hace |
|---|---|
| `9-11` | Valores por defecto: `MaxWidthHeight=1000`, `CompressionQuality=75`, `CustomPhotoSize=50` (%) |
| `22-28` | Lee la orientación EXIF (IFD0) |
| `33-36` | Decodifica y aplica la orientación |
| `39-48` | Tamaño: si **ambos** lados ≤ `MaxWidthHeight`, escala a `CustomPhotoSize` %; si no, ajusta el lado mayor a `MaxWidthHeight` |
| `52-58` | Redimensiona (`SKFilterMode.Linear`) y codifica JPEG |
| `73-89` | Sobrecarga path→path; implementa la §6.1 de la guía (`Transferencia_Foto_Entre_Pantallas.md:389-428`) |
| `91-136` | `switch` sobre EXIF 1–8: 3 → 180°, 6 → 90°, 8 → −90°; 2/4/5/7 → volteos |
| `138-183` | `Rotate` (crédito: `github.com/djdd87/SynoAI`, `:138`) |

- ⚠️ **Los volteos no voltean.** `Rotate(bitmap, angle, ex, ey)` llama a `canvas.Scale(ex, ey)` **después** de `DrawBitmap` (`:178-180`), así que la escala no afecta lo ya dibujado. Las orientaciones 2 y 4 quedan iguales; la 5 y la 7 sólo rotan. Los comentarios de `:102-124` prometen el volteo.
- ⚠️ Una foto chica (≤ 1000 px por lado) sale **a la mitad** (50 %), no intacta (`:39-40`).
- ⚠️ El bitmap rotado que devuelve `AplicarOrientation` no se libera (sólo `originalBitmap` y `resizedBitmap` tienen `using`, `:34,52`).
- ⚠️ En D la carpeta es `Services/`, pero el namespace es `…Normalizacion.Utilities` (`IImageService.cs:1`, `ImageDeviceAutoRotateService.cs:5`).
- Ni D ni E espejan la selfie: la normalización es la misma para las dos cámaras.

Consumidor (D, `Pages/MainPage.xaml.cs:21-55`; E, `Pages/MainPage.xaml.cs:19-58`): la lambda del callback es `async` asignada a `Action<string?>`, o sea `async void`. Procesa, lee los bytes, hace `Dispatcher.Dispatch` y borra **los dos** temporales (original y `photo_norm_*`) en `finally` (D `:43-54`). Como `MyMediaPickerPage` invoca el callback y enseguida hace `GoToAsync("..")`, el procesamiento corre en paralelo con la vuelta. `File.ReadAllBytes(outPath ?? "")` lanza si el procesamiento devolvió `null`, y el `catch` lo registra con `Debug.WriteLine` (D `:30,39-42`).

`README.md` de D (`Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/README.md`) repite el registro DI (`:33-39`), el handler de `MainPage` (`:47-101`) y los fragmentos de manifest y plist (`:106-125`), que coinciden con el código. ⚠️ Su lista de NuGet (`:7-10`) trae sólo `CommunityToolkit.Maui.Camera` y `.Core`: omite `MetadataExtractor` y `SkiaSharp`, que el servicio necesita.

## 9. E — Selfie (cámara frontal + máscara)

Diferencias con D (`Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion/…`):

| Aspecto | Ubicación |
|---|---|
| Cámara frontal ?? primera | `Pages/MyMediaSelfiePickerPage.xaml.cs:231` |
| Selección al `Loaded` del `CameraView` (además de `OnNavigatedTo`) | `.xaml.cs:170,219-222,88-95` |
| Flash siempre oculto | `.xaml.cs:176` |
| Máscara: `GraphicsView` con `InputTransparent` sobre `CameraContainer`, en un `Grid` interno | `Pages/MyMediaSelfiePickerPage.xaml:9-11,24-27` |
| `SelfieMaskDrawable`: rectángulo + elipse con `WindingMode.EvenOdd`; óvalo de 75 % del ancho, relación 1.35, tope 90 % del alto; color según el tema (blanco/negro) | `Utilities/SelfieMaskDrawable.cs:5-50` |
| Redibuja al cambiar el tema (`RequestedThemeChanged` → `Invalidate`) | `.xaml.cs:43-52,62-63` |
| Fondo y botones con `AppThemeBinding` | `.xaml:7,17,19,52,54` |

- ⚠️ La página vive en el namespace de D: `x:Class="Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.Pages.MyMediaSelfiePickerPage"` (`.xaml:5`, `.xaml.cs:5`). Por eso `MainPage.xaml.cs:1` y `AppShell.xaml.cs:1` importan `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.Pages`, aunque el `RootNamespace` es el de la selfie.
- ⚠️ En apaisado, `UpdateLayoutOrientation` reubica `CameraContainer` (`.xaml.cs:386,398`), pero `CameraContainer` es hijo del `Grid` interno, no de `DynamicLayout`. El `Grid` interno conserva `Grid.Row="1"`. El resultado visual en apaisado está **no verificado** en dispositivo.
- `Resources/Images/mascara.png` está versionado pero se excluye con `<None Remove>` (`.csproj:81-83`) y no lo usa ningún `.cs` ni `.xaml`: la máscara se dibuja por código.

## 10. `Ejemplo_Docs_Photo` (prosa del dominio)

| Documento | Qué aporta | Notas |
|---|---|---|
| `Readme.md` | Introducción y motivo de la página propia: el diálogo nativo «dio problemas en algunas versiones de Android» (`:6-9`); enlaces a Microsoft Learn (MediaPicker, CameraView); permisos Android de MediaPicker; `dotnet add package`; `UseMauiCommunityToolkitCore()`; `CAMERA` | Codificado en ISO-8859-1. Es el destino del enlace «Camara» de `README.md:14` («En construcción…») |
| `Transferencia_Foto_Entre_Pantallas.md` | Guía conceptual, ver tabla siguiente | UTF-8; fuera de la `.slnx` |

Secciones de `Transferencia_Foto_Entre_Pantallas.md`:

| § (líneas) | Tema |
|---|---|
| §1–2 (`34-108`) | El problema: hilos, ciclo de vida, carga diferida, scoped storage, EXIF |
| §3 (`112-257`) | Seis técnicas: `Stream`, `byte[]`, ruta en caché, *singleton* de foto pendiente, `WeakReferenceMessenger`, `FileProvider` |
| §4 (`261-339`) | WhatsApp, Instagram, Telegram, Signal y X (cifras aproximadas, con su propio descargo en `:267-271`) |
| §5–6 (`343-505`) | Downsampling, JPEG, EXIF; pipeline path→path y limpieza de intermedios |
| §7 (`509-660`) | `Dispatcher` / `MainThread`: postear o esperar; por qué no hay carrera al capturar `bytes` |
| §8 (`664-684`) | Multipart en vez de base64 |
| §9–10 (`688-714`) | Antipatrones y tabla comparativa |
| §11 (`718-739`) | Recomendación: el patrón de B |
| §12 (`743-762`) | Referencias |

- ⚠️ `Readme.md:4-9` dice «Hay dos ejemplos» (MediaPicker y CommunityToolkit), pero la carpeta tiene cinco proyectos.
- ⚠️ El fragmento de `MauiProgram` del `Readme.md` (`:50-74`) sólo llama a `UseMauiCommunityToolkitCore()`. B a E también necesitan `.UseMauiCommunityToolkitCamera()` (p. ej. `Ejemplo_Photo_MiMediaPicker_Callback/MauiProgram.cs:15-16`).
- ⚠️ El `Readme.md` (`:17-31`) lista `READ_MEDIA_*` y `READ_EXTERNAL_STORAGE`, que **no** están en el manifest de A (§7). A tiene `ElegirDeGaleriaAsync`, pero no lo usa.
- La guía pide pasar la ruta y no el stream. B, D y E lo cumplen; A y C no (§3 y §5).

## 11. Patrones transversales del dominio

| Patrón | Dónde | Por qué (según la fuente) |
|---|---|---|
| Materializar el stream a archivo en el productor | B/D/E `OnMediaCaptured` | el stream nace y muere en el hilo de la cámara (`…Callback/Pages/MyMediaPickerPage.xaml.cs:278-280`) |
| Leer los bytes y borrar el archivo en `finally` | B/D/E `MainPage` | caché limpia; la imagen vive en memoria (`Transferencia…md:169-174`) |
| `FromStream(() => new MemoryStream(bytes))` | B/D/E | cada reevaluación recibe un stream nuevo (`…Callback/Pages/MainPage.xaml.cs:46-52`) |
| Saltar al UI thread sólo para tocar controles | todos | `MainThread.InvokeOnMainThreadAsync` / `BeginInvokeOnMainThread` / `Dispatcher.Dispatch` |
| Overlay de estado de permiso en vez de alerta | B–E | la página muestra la causa y la salida (pedir, ajustes, volver) |
| Deshabilitar el botón mientras dura la operación | `MainPage` B–E; `DynamicLayout.IsEnabled` en la captura | evita doble navegación o doble captura |
| Orientación reactiva | B–E `UpdateLayoutOrientation` + `MainDisplayInfoChanged` | barra de botones al costado en apaisado |

Errores al usuario: sólo `DisplayAlertAsync("Error", …)` al fallar la navegación (`…Callback/Pages/MainPage.xaml.cs:62-65`) y «Error del dispositivo» si la cámara no está disponible. El resto va a `Debug.WriteLine`.

## 12. Preguntas frecuentes y trampas

| Pregunta | Respuesta corta |
|---|---|
| ¿Qué proyecto copio para una app real? | B (o D si hace falta orientar o comprimir). Es el que recomienda la guía (§10). |
| ¿Por qué la app se reiniciaba en Debug con MediaPicker? | `using` + `FromStream` diferido → `ObjectDisposedException` en Glide (`MEDIAPICKER_FIX.md:9-28`). |
| ¿Dónde está la rotación EXIF? | `ImageDeviceAutoRotateService.cs:91-136` (D) / `ImageDeviceAutoRotate.cs` (E). Los volteos no funcionan (§8). |
| ¿Cómo se cambia a cámara frontal? | `CameraPosition.Front` en `SeleccionarCamaraAsync` (E `:231`). |
| ¿Por qué no aparece la cámara en el simulador de iOS? | Guardia explícita en B/D/E (`…Callback/Pages/MyMediaPickerPage.xaml.cs:89-98`); `CameraView` no funciona ahí. |
| ¿Quién borra los temporales? | El consumidor (`MainPage`), en `finally`. Si se cancela, no se crea ninguno. |
| ¿El callback corre en el UI thread? | Se invoca dentro de `InvokeOnMainThreadAsync`. En D/E la lambda `async` sigue después de `await ProcesarPhotoAsync`, y el código igual encola la asignación con `Dispatcher.Dispatch` (comentario en D `Pages/MainPage.xaml.cs:32-33`; §7.3 de la guía). |
| ¿Por qué el overlay muestra «C�mara»? | Literales guardados con U+FFFD (§6). |
| ¿Existe la versión integrada en la app híbrida? | Sí, en `Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/{Camera,Images}`. Ver [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md). |
| ¿Cómo se compila o publica para iOS? | Workflows `cd-ios-camera.<proyecto>.yml` (B–E). Ver [09_CI-CD-y-Build](09_CI-CD-y-Build.md). |
