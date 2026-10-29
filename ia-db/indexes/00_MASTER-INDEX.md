# 00 — Índice maestro

> **Propósito**: dar la visión del repositorio como sistema. Qué es, con qué stack, qué 24 proyectos tiene y qué demuestra cada uno, cómo está organizado, qué patrones se repiten entre dominios (servicio → resultado tipado → overlay; cómo se decide el mensaje que ve el usuario ante un error) y qué decisiones lo estructuran. No repite el detalle de cada dominio: lo linkea.
> **Fuente primaria**: los 25 `.csproj` versionados, `Ejemplos_Devices/Ejemplos_Devices.slnx`, `.github/workflows/*.yml` y los tipos de resultado, fallo y overlay de `GPS/`, `Phone/`, `Printer/Ejemplo_MotorDSL_Dialog/` e `Integrada/`. Commit `285f6fb`; las rutas son relativas a la raíz del repo de código `Ejemplos_Maui_Devices`, y las que empiezan por un dominio (`GPS/…`, `Integrada/…`) cuelgan de `Ejemplos_Devices/`.
> **Índices relacionados**: [01_Camara](01_Camara.md) · [02_QR](02_QR.md) · [03_Impresion-Termica](03_Impresion-Termica.md) · [04_GPS](04_GPS.md) · [05_Mapas](05_Mapas.md) · [06_Telefonia](06_Telefonia.md) · [07_Red-Conectividad](07_Red-Conectividad.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md) · entrada: [README](../README.md)

---

## 1. Qué es

`Ejemplos_Maui_Devices` es material **didáctico** de la cátedra, no un producto ni una librería. Es una colección de ejemplos .NET MAUI, mínimos y autocontenidos, que muestran cómo una app móvil usa una capacidad del dispositivo: cámara, QR, impresión térmica Bluetooth, GPS, mapas, teléfono y red. Todo eso converge en una app híbrida (`Ejemplo_Maui_Hibrida`) donde la pantalla la pone una web Blazor remota y el lado nativo ejecuta las capacidades.

Rasgos constantes, deliberados (no son defectos):

| Rasgo | Cómo se ve | Detalle |
|---|---|---|
| Cada ejemplo se copia por separado | Sin proyecto de librería compartida; los tipos se repiten entre ejemplos y en la híbrida | §7, [10 §11](10_Documentacion-Transversal.md) (ADR-0001) |
| La variación es la lección | Matrices: 5 formas de capturar una foto, 4 librerías de QR × 2 variantes, 3 generaciones de impresión | [01 §2](01_Camara.md), [02 §1](02_QR.md), [03 §1](03_Impresion-Termica.md) |
| Se deja el rastro | Caminos anteriores comentados junto al nuevo; el porqué está en comentarios, CHANGELOG y docs | [10 §4](10_Documentacion-Transversal.md) |
| Hay proyectos «en construcción» | `Ejemplo_Maui_Connectivity` es un esqueleto; varios `Ejemplo_Docs_*` están vacíos | [07 §5](07_Red-Conectividad.md), [10 §9](10_Documentacion-Transversal.md) |

---

## 2. Stack (verificado en los `.csproj`)

| Capa | Valor | Fuente |
|---|---|---|
| Lenguaje / UI | C# · .NET MAUI (XAML) · Blazor Interactive Server (backend) | `.csproj` de cada proyecto |
| TFM de las 22 apps MAUI | `net10.0-android`; se suma `net10.0-ios` **sólo si el host es macOS** (`IsOSPlatform('osx')`) | p. ej. `GPS/Ejemplo_Maui_GPS/Ejemplo_Maui_GPS.csproj:4-5` |
| TFM extra | `net10.0-windows10.0.19041.0` sólo en `Ejemplo_Maui_Mapas` y sólo si el host es Windows | [05 §3](05_Mapas.md) |
| TFM de web y tests | `net10.0` (`Microsoft.NET.Sdk.Web` / xUnit con `UseMaui`) | [08 §1](08_App-Hibrida-Integrada.md) |
| Mínimos | iOS 15.0 · Android 25 (21 en `Ejemplo_Maui_DirectCall`) | [09 §13](09_CI-CD-y-Build.md) |
| `Microsoft.Maui.Controls` | `$(MauiVersion)` en 13 apps; no se fija en el repo (no hay `Directory.Build.props` ni `global.json`), así que el valor efectivo **no está verificado**. Fijado en: 10.0.30 (Callback, Task), 10.0.31 (las dos de normalización), 10.0.60 (Dialer, MotorDSL_Dialog), 10.0.80 (BSN ×2, que dejan comentada la línea `$(MauiVersion)`, e Híbrida) ([02 §3](02_QR.md)) | `.csproj` |
| SDK en CI | .NET SDK 10.0.300 · workloads 10.0.100 · Xcode 26.0 | [09 §5](09_CI-CD-y-Build.md) |
| Dispositivos de referencia | Android físico: Motorola Moto g42 (textos del flujo Maestro verificados en él, `CHANGELOG.md:140`; caso de estudio TLS, `Ejemplos_Devices/Docs/Certificados-SSL/README.md:6`) · CI: simulador iPhone 17 Pro Max | [09 §2](09_CI-CD-y-Build.md), [10 §5](10_Documentacion-Transversal.md) |

**Paquetes por necesidad**

| Necesidad | Paquete @ versión | Dónde |
|---|---|---|
| Cámara con UI propia | `CommunityToolkit.Maui.Camera` 6.0.0 (Cámara) / 6.1.0 (Híbrida) | [01](01_Camara.md), [08 §3.3](08_App-Hibrida-Integrada.md) |
| Normalizar imagen | `MetadataExtractor` 2.9.0 + `SkiaSharp` 3.119.1 (Cámara) / 3.119.2 (Híbrida) | [01 §8](01_Camara.md) |
| QR | `BarcodeScanner.Mobile.Maui` 9.0.1 · `BarcodeScanning.Native.Maui` 3.0.4 · `CameraScanner.Maui` 1.8.31 · `CameraMaui` 1.4.5 | [02 §2](02_QR.md) |
| QR en simulador x64 | `AdamE.Google.iOS.GoogleUtilities` 8.1.0.3 (condicional por RID) | [09 §7](09_CI-CD-y-Build.md) |
| ESC/POS a mano | `ESCPOS_NET` 2.2.1 | [03 §3](03_Impresion-Termica.md) |
| Impresión por documento | 7 × `MotorDsl.*` 1.0.12 (`Ejemplo_MotorDSL`) / 1.0.13 (`_Dialog`, Híbrida, Tests) | [03 §4-§5](03_Impresion-Termica.md) |
| Mapas | `Microsoft.Maui.Controls.Maps` 10.0.40 | [05 §3](05_Mapas.md) |
| MVVM / Toolkit | `CommunityToolkit.Mvvm` 8.4.2 · `CommunityToolkit.Maui(.Core)` 14.0.0–14.2.0 | Híbrida, `MotorDSL_Dialog`, Tests |
| Backend | `Microsoft.AspNetCore.OpenApi` 10.0.8 · `Scalar.AspNetCore` 2.14.14 | [08 §10](08_App-Hibrida-Integrada.md) |
| Tests | `xunit` 2.9.2 · `xunit.runner.visualstudio` 2.8.2 · `Microsoft.NET.Test.Sdk` 17.11.1 | [08 §12](08_App-Hibrida-Integrada.md) |

GPS, Mapas (salvo `.Maps`), Teléfono y Red no usan NuGet de dominio: sólo MAUI Essentials. ⚠️ `Ejemplo_ThermalPrinter` referencia `BarcodeScanner.Mobile.Maui` 9.0.1 y no lo usa en el código ([03 §2](03_Impresion-Termica.md)).

---

## 3. Catálogo de los 24 proyectos

`git ls-files '*.csproj'` devuelve 25. El 25.º, `Printer/Ejemplo_Docs_Printer/Prompts_Generacion_Apk_ejemplo/ficheros_claude.ia/Thermalprintermaui.csproj` (`net8.0`, `ESCPOS_NET` 4.5.0), es material de documentación y no forma parte de la solución ([03 §8](03_Impresion-Termica.md)). «Workflow» quiere decir que existe un `.github/workflows/cd-ios-*.yml` que compila el proyecto. Sólo el de la híbrida corre solo; los otros 17 son `workflow_call` sin llamador ([09 §2.1](09_CI-CD-y-Build.md)).

| Dominio | Proyecto | `ApplicationId` | Qué demuestra | Workflow | `.slnx` |
|---|---|---|---|:-:|:-:|
| Cámara [01](01_Camara.md) | `Ejemplo_Photo_MediaPicker` | `com.ejemplos.devices.mediapicker` | Diálogo nativo `MediaPicker`, devuelve un `MemoryStream` | no | sí |
| | `Ejemplo_Photo_MiMediaPicker_Callback` | `com.ejemplos.photo.mimediapicker.callback` | Página propia con `CameraView`; la foto vuelve por *callback* con la ruta en caché (el de referencia) | sí | sí |
| | `Ejemplo_Photo_MiMediaPicker_Task` | `com.ejemplos.devices.mimediapicker.task` | Misma página, retorno por `TaskCompletionSource<Image>` (contraejemplo) | sí | sí |
| | `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion` | Callback + normalización EXIF/tamaño por DI | sí | sí |
| | `Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion.miselfie` | Cámara frontal + máscara de selfie | sí | sí |
| QR [02](02_QR.md) | `BSM.LectorQR` | `com.ejemplos.devices.qr.barcodescanner_mobile_maui.simple` | BarcodeScanner.Mobile (ML Kit), lectura con alerta | sí (x64) | sí |
| | `BSM.LectorQR_Dialog` | `…qr.barcodescanner_mobile_maui.dialog` | Página lectora como diálogo con resultado (`TaskCompletionSource`) | sí (x64) | sí |
| | `BSN.LectorQR` | `…qr.barcodescanner_native_maui.simple` | BarcodeScanning.Native (la que usa la híbrida) | sí | sí |
| | `BSN.LectorQR_Dialog` | `…qr.barcodescanner_native_maui.dialog` | Ídem, variante diálogo | sí | sí |
| | `CS.LectorQR` | `…qr.camerascanner_maui.simple` | CameraScanner.Maui | sí | sí |
| | `CS.LectorQR_Dialog` | `…qr.camerascanner.maui.dialog` ⚠️ | Ídem, variante diálogo | sí | sí |
| | `ZN.LectorQR` | `…qr.zxing_net_maui.simple` | `CameraMaui` (Camera.MAUI con ZXing; ⚠️ no es ZXing.Net.Maui) | sí | sí |
| | `ZN.LectorQR_Dialog` | `…qr.zxing_net_maui.dialog` | Ídem, variante diálogo | sí | sí |
| Impresión [03](03_Impresion-Termica.md) | `Ejemplo_ThermalPrinter` | `com.ejemplos.devices.ThermalPrinter` | ESC/POS a mano con un socket RFCOMM propio, code-behind | no | sí |
| | `Ejemplo_MotorDSL` | `com.ejemplos.devices.MotorDSL` | Documento JSON → motor DSL → bytes | sí | sí |
| | `Ejemplo_MotorDSL_Dialog` | `com.ejemplos.devices.MotorDSL.Dialog` | Flujo con overlay, resultados tipados y catálogo `PRN-*` | no | sí |
| GPS [04](04_GPS.md) | `Ejemplo_Maui_GPS` | `com.ejemplos.devices.gps` | Permiso → lectura → `GpsResult` → coordinador, geocodificación inversa por Google | sí | sí |
| Mapas [05](05_Mapas.md) | `Ejemplo_Maui_Mapas` | `com.ejemplos.devices.mapas` | Control `Map`: centrado, pines, tipo de mapa | no | sí |
| Teléfono [06](06_Telefonia.md) | `Ejemplo_Maui_Dialer` | `com.ejemplos.phone.dialer` | Abrir el marcador (sin permisos) | sí | sí |
| | `Ejemplo_Maui_DirectCall` | `com.ejemplos.phone.directcall` | Llamada directa con `CALL_PHONE`, `CallResult` y overlay | sí | sí |
| Red [07](07_Red-Conectividad.md) | `Ejemplo_Maui_Connectivity` | `com.ejemplos.red.connectivity` | Esqueleto: `ConnectivityChanged` a `Console`; la UI está vacía | no | sí |
| Integrada [08](08_App-Hibrida-Integrada.md) | `Ejemplo_Maui_Hibrida` | `com.ejemplos.devices.integrada.hibrida` | `WebView` + puente de comandos por URL + 4 overlays | sí (`push`) | sí |
| | `Ejemplo_ws_Blazor` | — (web, `net10.0`) | Backend Blazor + Web API, desplegado en `https://aplicada.somee.com` | no | sí |
| | `Ejemplo_Maui_Hibrida.Tests` | — (xUnit, `net10.0`) | 78 tests (67 `[Fact]` + 11 `[Theory]`) de VMs, puente e invariantes | no | **no** |

`…` = `com.ejemplos.devices.`. Totales: **17 con workflow / 7 sin** ([09 §3](09_CI-CD-y-Build.md)); **23 en el `.slnx` / 1 fuera** ([09 §12](09_CI-CD-y-Build.md)). Un workflow 18.º, `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml`, quedó huérfano: apunta a un proyecto que se renombró a `ZN.LectorQR_Dialog` ([09 §16](09_CI-CD-y-Build.md)). ⚠️ Los `PACKAGE_NAME` de tres workflows QR no coinciden con el bundle id real ([09 §13.1](09_CI-CD-y-Build.md)).

---

## 4. Estructura del repositorio

```
Ejemplos_Maui_Devices/                       (990 archivos versionados en 285f6fb)
├── README.md            portada; enlaza 6 dominios (no Teléfono ni Integrada)  → 10 §3
├── CHANGELOG.md         9 entradas, 2026-07-13 → 2026-07-23                     → 10 §4
├── .gitignore           plantilla VisualStudio + **/Services/ApiKeys.cs          → 09 §14
├── vs.bat               `code .`
├── .github/workflows/   18 cd-ios-*.yml + Readme.md + Pipelinea-Version.md + Analisis/  → 09
├── Utilities/           simular.sh · simular_ui.sh · end2end/*.yaml (Maestro) · download_sectigo.ps1 → 09 §8-§11
└── Ejemplos_Devices/
    ├── Ejemplos_Devices.slnx         solución XML: 23 proyectos + carpetas de archivos sueltos → 09 §12
    ├── Camera/    5 proyectos + Ejemplo_Docs_Photo       → 01
    ├── QR/        8 proyectos + Ejemplo_Docs_QR          → 02
    ├── Printer/   3 proyectos + Ejemplo_Docs_Printer     → 03
    ├── GPS/       1 proyecto  + Ejemplo_Docs_GPS         → 04
    ├── Maps/      1 proyecto  + Ejemplo_Docs_Maps        → 05
    ├── Phone/     2 proyectos (sin carpeta de docs)      → 06
    ├── Red/       1 proyecto  + Ejemplo_Docs_Red         → 07
    ├── Integrada/ 3 proyectos + Ejemplo_Docs_Integrada   → 08
    ├── Docs/      Certificados-SSL · otros · qr-nuget · web-hibrida (18 archivos) → 10
    └── scripts/   3 .bat de build+deploy Android (Windows) → 09 §11
```

| Conteo (`git ls-files`, sin `wwwroot/lib` ni `Platforms/*/Resources`) | |
|---|---|
| `.cs` / `.xaml` / `.razor` | 322 / 129 / 12 |
| `.md` | 62 |
| `.yml` / `.sh` / `.bat` | 18 / 2 / 4 |
| Carpetas `Ejemplo_Docs_*` | 7 |

---

## 5. Patrón transversal: servicio → resultado tipado → overlay

El repo resuelve igual cada capacidad que puede fallar: el servicio **no lanza**, devuelve un `abstract record` con un `sealed record` por situación que la UI tiene que distinguir; un coordinador o un VM de overlay hace `switch` y decide la pantalla.

```
 Página / comando / URL ──► Coordinador u OverlayVM (singleton, dueño del CTS y del hilo UI)
                               │ ShowBusy(...)
                               ▼
                           I*Service.XxxAsync()  ──► Permisos → API del SO / librería
                               │ devuelve XxxResult (sin excepciones)
                               ▼
                 switch (result) ─► Success → Hide()
                               └─► resto  → pantalla de error + botonera
```

### 5.1 Resultados tipados que existen

| Tipo | Variantes | Fuente |
|---|---|---|
| `GpsResult` | 8: `Success`, `PermissionDenied(CanRetry)`, `PermissionRestricted`, `GpsDisabled`, `NotSupported`, `NoSignal`, `Cancelled`, `Failure` | `GPS/Ejemplo_Maui_GPS/Services/GpsResult.cs:5-15` (copia en la Híbrida) |
| `CallResult` (DirectCall) | 7: …`PermissionDenied`, `PermissionDeniedPermanent`… | `Phone/Ejemplo_Maui_DirectCall/Services/CallResult.cs:6-28` |
| `CallResult` (Híbrida) | 7: funde `PermissionDenied`/`PermissionDeniedPermanent` en `PermissionDenied(CanRetry)`, agrega `InvalidNumber` y `Success` lleva `CallMode` | `Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/Phone/Models/CallResult.cs:5` (la Híbrida como un todo: [06 §8](06_Telefonia.md)) |
| `DiscoverResult` / `PrintResult` | 5 (`Found`, `Empty`, `BluetoothOff`, `PermissionRevoked`, `NotSupported`) / 2 | `Printer/Ejemplo_MotorDSL_Dialog/Models/` → [03 §5.2](03_Impresion-Termica.md) |
| `DocumentResult` | 3 (`Ok`, `NetworkError`, `InvalidContract`), sólo en la Híbrida | → [08 §7.1](08_App-Hibrida-Integrada.md) |
| `NetworkResult` | 6 (`Online`, `Offline`, `Timeout`, `DnsFailure`, `HttpFailure`, `RequestFailure`), sólo en la Híbrida | → [07 §8](07_Red-Conectividad.md), [08 §8.2](08_App-Hibrida-Integrada.md) |
| `ApiCallResult` | 6 (`Success`, `HttpError`, `NetworkError`, `Timeout`, `Cancelled`, `Blocked`) | → [08 §7.2](08_App-Hibrida-Integrada.md) |

Los permisos se normalizan a cuatro casos, `Granted`, `DeniedCanRetry`, `Denied` y `Restricted`, en `LocationPermissionResult`, `BluetoothPermissionResult` y `CallPermissionResult` (p. ej. `GPS/Ejemplo_Maui_GPS/Services/LocationPermissionResult.cs:7-10`). `DeniedCanRetry` sale de `ShouldShowRationale`, que es de Android → [04 §4.1](04_GPS.md), [06 §5.2](06_Telefonia.md), [03 §6.2](03_Impresion-Termica.md).

### 5.2 Dos generaciones del overlay

| | Temprana: `Ejemplo_Maui_GPS`, `Ejemplo_Maui_DirectCall` | Madura: `Ejemplo_MotorDSL_Dialog` y los 4 overlays de la Híbrida |
|---|---|---|
| Quién decide | `GpsCoordinator` / `CallCoordinator` singletons (`GPS/…/MauiProgram.cs:34`, `Phone/Ejemplo_Maui_DirectCall/MauiProgram.cs:38`) | `XxxOverlayViewModel : StatusOverlayViewModel` |
| Estados | Métodos ad hoc (`ShowPermissionDenied`, `ShowRestricted`, `ShowFailure`…) | `OverlayMode { None, Busy, Error }` (`Integrada/…/LibApp/Devices/Common/ViewModels/StatusOverlayViewModel.cs:11`) |
| Botonera | Fija en XAML, con callbacks pasados por constructor | Dinámica: `OverlayAction(Text, Command, Style)` |
| Detalle | [04 §5-§6](04_GPS.md), [06 §5.4-§5.5](06_Telefonia.md) | [03 §5.4](03_Impresion-Termica.md), [08 §8](08_App-Hibrida-Integrada.md) |

### 5.3 De los ejemplos a la híbrida

Los ejemplos aislados son los ensayos; `Ejemplo_Maui_Hibrida` reúne una versión endurecida de cada uno bajo `Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/`. Lo que **sólo** existe en la híbrida es el puente de comandos por URL (`LibApp/UrlCommands/`), el bridge del `WebView` (`LibApp/CustomWebView/`) y la suite de tests con invariantes ejecutables.

| Dominio | Qué llega a la híbrida | Carpeta | Detalle |
|---|---|---|---|
| Cámara | `MyMediaPickerPage` y `MyMediaSelfiePickerPage` + normalización EXIF/tamaño (`MetadataExtractor` + `SkiaSharp`) | `Camera/`, `Images/` | [01 §8-§9](01_Camara.md), [08 §6.3](08_App-Hibrida-Integrada.md) |
| QR | `QRLectorPage` con `BarcodeScanning.Native.Maui` 3.0.4 (la recomendada por `Docs/qr-nuget`) | `QRLector/` | [02 §12](02_QR.md), [08 §7.3](08_App-Hibrida-Integrada.md) |
| Impresión | El flujo de `Ejemplo_MotorDSL_Dialog` (catálogo `PRN-*` idéntico) + `IPrinterService` y el caso «documento por red» (`DocumentResult`) | `MotorDSL/` | [03 §9](03_Impresion-Termica.md), [08 §7.1](08_App-Hibrida-Integrada.md) |
| GPS | Servicio con lectura fresca y permisos adentro, overlay de 2.ª generación y catálogo `GPS-*` | `GPS/` | [04 §10](04_GPS.md) |
| Teléfono | `ICallService` + overlay de 2.ª generación y catálogo `TEL-*` | `Phone/` | [06 §8](06_Telefonia.md) |
| Red | `NetworkService` con sonda HTTP activa y overlay reactivo (el ejemplo aislado es un esqueleto) | `Networks/` | [07 §8](07_Red-Conectividad.md), [08 §8.2](08_App-Hibrida-Integrada.md) |

Mapas no tiene contraparte en la híbrida.

---

## 6. Cómo se decide el mensaje al usuario ante un error

Hay tres modelos en el repo, uno por generación. El tercero es la regla vigente.

| Modelo | Dónde | Qué ve el usuario | Fuente |
|---|---|---|---|
| **1. Excepción a texto** | `Ejemplo_ThermalPrinter`, `Ejemplo_MotorDSL` | `ex.Message` encadenado (TP); tipo, mensaje, inner y stack para diagnóstico (MD) | [03 §7.5](03_Impresion-Termica.md) |
| **2. Switch del coordinador** | `Ejemplo_Maui_GPS`, `Ejemplo_Maui_DirectCall` | GPS: el overlay se muestra sólo para permiso o restricción (`GpsCoordinator.cs:56-70`) y el resto va a un label con `ActualizarTexto`, que en `Failure` muestra `$"Error: {f.Message}"` (`GPS/Ejemplo_Maui_GPS/ViewModels/MainPageViewModel.cs:83-97`). DirectCall: `Overlay.ShowFailure(f.Message)` (`Phone/Ejemplo_Maui_DirectCall/Services/CallCoordinator.cs:94-120`) | [04 §5, §7.3](04_GPS.md), [06 §5.4](06_Telefonia.md) |
| **3. Catálogo con código estable** | `Ejemplo_MotorDSL_Dialog` (`PRN-*`) e Híbrida (`GPS-*`, `TEL-*`, `PRN-*`) | `Title` + `DisplayMessage` = `"{UserMessage}\n\n{Code}"`; el `TechnicalMessage` va al log | ver abajo |

**Cadena del modelo 3** (ejemplo GPS de la Híbrida; los demás son espejo):

```
GpsResult ──GpsErrorCatalog.Describir(result)──► GpsFailure(Code, Title, UserMessage, TechnicalMessage)
   (GpsErrorCatalog.cs:82-92)                         │
                                                      ▼
GpsOverlayViewModel.Mostrar(fallo): switch (fallo.Code) → botonera   (GpsOverlayViewModel.cs:61-99)
                                    ShowError(icono, fallo.Title, fallo.DisplayMessage, acciones)
```

Rutas bajo `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/GPS/`.

| Regla | Texto / efecto | Fuente |
|---|---|---|
| Los códigos son contrato con soporte | Se agregan, no se renombran | `GPS/Models/GpsErrorCatalog.cs:7-9` · `Printer/Ejemplo_MotorDSL_Dialog/Models/PrinterErrorCatalog.cs:9-11` |
| Un código por **acción distinta del usuario** | Dos causas que se resuelven con el mismo gesto comparten código | `PrinterErrorCatalog.cs:13-14` · `GpsErrorCatalog.cs:11` |
| El código va separado y al final | Para dictárselo a soporte sin que compita con la acción | `GPS/Models/GpsFailure.cs:20-24` |
| Sólo `UserMessage` llega al usuario | `TechnicalMessage` conserva el original para el log | `GpsFailure.cs:4-5` |
| Excepciones de envío: no se reclasifican | `PrinterErrorCatalog.Describe` usa `PrintError.FromException` de la librería y mapea `Hardware`/`Connection`/`Timeout`/resto | `PrinterErrorCatalog.cs:107-128` → [03 §7.3](03_Impresion-Termica.md) |
| La botonera sale del código | Un botón primario y «Cerrar» secundario si no es la única salida | `GpsOverlayViewModel.cs:65-96` · [03 §7.4](03_Impresion-Termica.md) |
| Red no tiene códigos | Mensajes fijos por `NetworkResult` | [08 §8.3](08_App-Hibrida-Integrada.md) |

**Invariantes ejecutables** (`Integrada/Ejemplo_Maui_Hibrida.Tests/Invariantes.cs`; el comentario de `:8` habla de «los cinco invariantes»):

| Id | Invariante | Línea |
|---|---|---|
| I-1 | Toda variante no-`Success` produce exactamente una pantalla (con título, mensaje y salida) | `:15` |
| I-2 | Toda variante del resultado tipado tiene su pantalla (reflexión sobre los `sealed record`) | `:68-79` |
| I-3 | Ningún mensaje crudo del sistema llega al usuario | `:36` |
| I-4 | Toda pantalla de error tiene exactamente un botón primario | `:50` |
| I-5 | El VM devuelve la variante que recibió | `GpsOverlayTests.cs:32,46` |

⚠️ `Invariantes.cs:9` cita `Ejemplos_Maui_Devices.Documentos/Analisis/Plan-Armonizacion-Overlays.md`. Ese repo no existe: el documento está en `Ejemplos_Maui_Devices.Documentacion/Analisis/` ([10 §10](10_Documentacion-Transversal.md)). Catálogos completos: [08 §8.3](08_App-Hibrida-Integrada.md) y [03 §7.2](03_Impresion-Termica.md).

---

## 7. Decisiones clave

| Decisión | Consecuencia | Dónde se desarrolla |
|---|---|---|
| Un proyecto MAUI mínimo por técnica (ADR-0001) | No hay librería común; los tests **linkean fuentes** en vez de `ProjectReference` | [10 §11](10_Documentacion-Transversal.md), [08 §12.1](08_App-Hibrida-Integrada.md) |
| `GpsFailure`/`CallFailure`/`PrintFailure` duplicados a propósito | Unificarlos obligaba a reabrir impresión, ya validada en dispositivo; «lo que armoniza el patrón son los invariantes» (`GpsFailure.cs:7-11`) | [08 §8.1](08_App-Hibrida-Integrada.md) |
| Servicio tipado + overlay MVVM (ADR-0002) | §5-§6 de este índice | [04 §3](04_GPS.md), [06 §5](06_Telefonia.md) |
| `WebView` estándar sobre web remota + puente por URL (ADR-0003) | `Plan()` síncrono antes de cualquier `await`; `e.Cancel`; dos modos de entrega | [08 §2, §4, §6](08_App-Hibrida-Integrada.md) |
| Cierre determinista del overlay GPS + reentrada (ADR-0009) | `Hide()` en éxito sin depender de `Navigated`; guard de reentrada | [08 §4.3, §6.2](08_App-Hibrida-Integrada.md), [04 §10](04_GPS.md) |
| QR: BSN (Apple Vision) por sobre BSM (ML Kit) en iOS (ADR-0004) | Se aplicó como colección comparativa, no como reemplazo; la Híbrida usa BSN | [02 §9-§10](02_QR.md), [10 §7](10_Documentacion-Transversal.md) |
| Impresión con motor DSL sobre ESC/POS directo (ADR-0005); `MotorDsl.*` como NuGet | Valida los paquetes publicados | [03 §1-§2](03_Impresion-Termica.md) |
| CI sólo iOS al simulador, firma ad-hoc; Android local (ADR-0006) | Ningún pipeline compila Android ni corre tests | [09 §1, §6](09_CI-CD-y-Build.md) |
| Xcode propio desde Drive | Toolchain fijado; el motivo explícito **no está verificado** | [09 §5](09_CI-CD-y-Build.md) |
| Secretos fuera del repo (ADR-0007) | `ApiKeys.cs` ignorado, así que GPS no compila recién clonado. ⚠️ La key de Mapas sigue hardcodeada en el manifest | [04 §9](04_GPS.md), [09 §10](09_CI-CD-y-Build.md), [05 §8.1](05_Mapas.md) |
| TLS en WebView Android con trust anchors embebidos (ADR-0008) | `.pem` en `res/raw` + `network_security_config` | [10 §5](10_Documentacion-Transversal.md), [08 §11](08_App-Hibrida-Integrada.md) |
| ViewModels de la Híbrida sin `#if` | Lo de plataforma vive detrás de `I*Service`, así que la suite corre en `net10.0` sin emulador | [08 §12](08_App-Hibrida-Integrada.md) |

Los ADR viven en el repo de documentación (`Ejemplos_Maui_Devices-docs/docs/04-decisions/`). Sólo ADR-0009 está `accepted` y los demás `draft`; la correspondencia ADR → documento del repo de código se armó a partir de los títulos ([10 §11](10_Documentacion-Transversal.md)).

---

## 8. Dónde mirar primero

| Si necesitás… | Leé |
|---|---|
| Trabajar sobre un dispositivo concreto | el índice del dominio (01–07) |
| Tocar la app final o el backend | [08](08_App-Hibrida-Integrada.md) y el índice del dispositivo |
| Compilar, firmar, simular, la solución | [09](09_CI-CD-y-Build.md) |
| El documento que explica un porqué, o qué está desactualizado | [10 §1, §12](10_Documentacion-Transversal.md) |
