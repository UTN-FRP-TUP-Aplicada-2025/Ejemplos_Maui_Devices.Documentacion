# 00 — Índice maestro

> **Propósito:** visión general del repositorio, stack, catálogo de proyectos y las decisiones que se repiten en todos los dominios.
> **Fuente primaria:** `Ejemplos_Maui_Devices/` completo (24 proyectos, 18 workflows, ~60 documentos).
> **Entrada de la base:** [README.md](../README.md).

---

## 1. Qué es

**`Ejemplos_Maui_Devices`** es un repositorio **didáctico** de la cátedra: una colección de ejemplos .NET MAUI, cada uno mínimo y autocontenido, que muestran cómo usar una capacidad del dispositivo desde una app móvil — cámara, QR, impresión térmica Bluetooth, GPS, mapas, teléfono y red — y una app final que los **integra todos** dentro de un `WebView`.

No es un producto ni una librería: es material de estudio. Eso explica sus rasgos más constantes:

- **Cada ejemplo es copiable por separado.** Nada de proyecto compartido: el código se duplica a propósito entre ejemplos (`AsyncRelayCommand`, `ImageDeviceAutoRotate`, los overlays) para que uno se pueda llevar sin arrastrar el resto.
- **Se conserva el rastro del error.** Los caminos anteriores quedan comentados junto al nuevo, y los documentos explican el defecto que motivó cada cambio.
- **La variación es la lección.** Varios dominios son matrices: 5 formas de capturar una foto, 4 librerías de QR × 2 modos de uso, 3 generaciones de impresión.

## 2. Datos clave

| | |
|---|---|
| **Repositorio** | `APLICADA/Ejemplos_Maui_Devices` (git) |
| **Documentación** | `APLICADA/Ejemplos_Maui_Devices.Documentacion` (repositorio aparte — acá vive esta ia-db) |
| **Lenguaje / stack** | C# · .NET 10 · .NET MAUI · XAML · Blazor Server |
| **TFM** | `net10.0-android` (+ `net10.0-ios` solo al compilar en macOS); `net10.0-windows10.0.19041.0` solo en Mapas |
| **Plataformas mínimas** | Android API 25 (21 en `Ejemplo_Maui_DirectCall`) · iOS 15.0 |
| **Proyectos** | 24 (22 apps MAUI, 1 web Blazor, 1 de tests), más 8 carpetas `Ejemplo_Docs_*` sin proyecto |
| **Solución** | `Ejemplos_Devices/Ejemplos_Devices.slnx` (formato XML) — **23 proyectos: el de tests no está incluido** |
| **CI** | 18 workflows de GitHub Actions, todos build + simulador **iOS** en `macos-15` |
| **Dispositivo de referencia** | Motorola Moto g42 (Android) · simulador iPhone 17 Pro Max (CI) |
| **Backend desplegado** | `https://aplicada.somee.com` |

## 3. Arquitectura del repositorio

```
Ejemplos_Maui_Devices/
├── README.md                 Portada con enlace a cada dominio
├── CHANGELOG.md              9 entradas, 2026-07-13 → 2026-07-23
├── .gitignore                VS estándar + **/Services/ApiKeys.cs
├── vs.bat                    una línea: code .
├── .github/workflows/        18 pipelines CD iOS + 2 documentos  → índice 09
├── Utilities/                simular.sh, simular_ui.sh, end2end/  → índice 09
└── Ejemplos_Devices/
    ├── Ejemplos_Devices.slnx
    ├── Camera/     5 proyectos + Ejemplo_Docs_Photo               → índice 01
    ├── QR/         8 proyectos + Ejemplo_Docs_QR                  → índice 02
    ├── Printer/    3 proyectos + Ejemplo_Docs_Printer             → índice 03
    ├── GPS/        1 proyecto  + Ejemplo_Docs_GPS                 → índice 04
    ├── Maps/       1 proyecto  + Ejemplo_Docs_Maps                → índice 05
    ├── Phone/      2 proyectos                                    → índice 06
    ├── Red/        1 proyecto  + Ejemplo_Docs_Red                 → índice 07
    ├── Integrada/  3 proyectos + Ejemplo_Docs_Integrada           → índice 08
    ├── Docs/       18 documentos transversales                    → índice 10
    └── scripts/    3 .bat de arranque para Windows
```

## 4. Catálogo de proyectos

| Dominio | Proyecto | `ApplicationId` | CI | Índice |
|---------|----------|-----------------|:--:|--------|
| Cámara | `Ejemplo_Photo_MediaPicker` | `com.ejemplos.devices.mediapicker` | — | [01](01_Camara.md) |
| | `Ejemplo_Photo_MiMediaPicker_Task` | `…mimediapicker.task` | ✅ | |
| | `Ejemplo_Photo_MiMediaPicker_Callback` | `com.ejemplos.photo.mimediapicker.callback` | ✅ | |
| | `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` | `…imagen.callback.normalizacion` | ✅ | |
| | `Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | `…normalizacion.miselfie` | ✅ | |
| QR | `BSM.LectorQR{,_Dialog}` | `…qr.barcodescanner_mobile_maui.{simple,dialog}` | ✅ | [02](02_QR.md) |
| | `BSN.LectorQR{,_Dialog}` | `…qr.barcodescanner_native_maui.{simple,dialog}` | ✅ | |
| | `CS.LectorQR{,_Dialog}` | `…qr.camerascanner{_maui.simple,.maui.dialog}` | ✅ | |
| | `ZN.LectorQR{,_Dialog}` | `…qr.zxing_net_maui.{simple,dialog}` | ✅ | |
| Impresión | `Ejemplo_ThermalPrinter` | `…devices.ThermalPrinter` | — | [03](03_Impresion-Termica.md) |
| | `Ejemplo_MotorDSL` | `…devices.MotorDSL` | ✅ | |
| | `Ejemplo_MotorDSL_Dialog` | `…devices.MotorDSL.Dialog` | — | |
| GPS | `Ejemplo_Maui_GPS` | `com.ejemplos.devices.gps` | ✅ | [04](04_GPS.md) |
| Mapas | `Ejemplo_Maui_Mapas` | `com.ejemplos.devices.mapas` | — | [05](05_Mapas.md) |
| Teléfono | `Ejemplo_Maui_Dialer` | `com.ejemplos.phone.dialer` | ✅ | [06](06_Telefonia.md) |
| | `Ejemplo_Maui_DirectCall` | `com.ejemplos.phone.directcall` | ✅ | |
| Red | `Ejemplo_Maui_Connectivity` | `com.ejemplos.red.connectivity` | — | [07](07_Red-Conectividad.md) |
| Integrada | `Ejemplo_Maui_Hibrida` | `…devices.integrada.hibrida` | ✅ | [08](08_App-Hibrida-Integrada.md) |
| | `Ejemplo_ws_Blazor` | — (web) | — | |
| | `Ejemplo_Maui_Hibrida.Tests` | — (xUnit, `net10.0`) | — | |

## 5. Stack por dominio

| Necesidad | Paquete | Dónde |
|-----------|---------|-------|
| Cámara con UI propia | `CommunityToolkit.Maui.Camera` 6.0.0 / 6.1.0 | Cámara, híbrida |
| Normalización de imagen | `MetadataExtractor` 2.9.0 + `SkiaSharp` 3.119.x | Cámara (2), híbrida |
| Escaneo QR | `BarcodeScanner.Mobile.Maui` 9.0.1 · `BarcodeScanning.Native.Maui` 3.0.4 · `CameraScanner.Maui` 1.8.31 · `CameraMaui` 1.4.5 | QR (uno por par), híbrida (`BSN`) |
| Impresión ESC/POS a mano | `ESCPOS_NET` 2.2.1 | `Ejemplo_ThermalPrinter` |
| Impresión por documento | `MotorDsl.*` (7 paquetes) 1.0.12 / **1.0.13** | `Ejemplo_MotorDSL{,_Dialog}`, híbrida |
| Mapas | `Microsoft.Maui.Controls.Maps` 10.0.40 | Mapas |
| MVVM | `CommunityToolkit.Mvvm` 8.4.2 | Híbrida, `MotorDSL_Dialog`, tests |
| Toolkit (behaviors, converters) | `CommunityToolkit.Maui{,.Core}` 14.x | Híbrida, `MotorDSL_Dialog` |
| Backend | `Microsoft.AspNetCore.OpenApi` 10.0.8 + `Scalar.AspNetCore` 2.14.14 | `Ejemplo_ws_Blazor` |
| Tests | `xunit` 2.9.2 + `Microsoft.NET.Test.Sdk` 17.11.1 | `Ejemplo_Maui_Hibrida.Tests` |

Los ejemplos simples usan `Microsoft.Maui.Controls` con `$(MauiVersion)`; los que necesitan una versión concreta la fijan (10.0.30 … 10.0.80).

## 6. Los patrones que se repiten

Cinco decisiones aparecen, con variaciones, en casi todos los dominios. Conocerlas es entender el repositorio.

### 6.1 Resultado tipado en vez de excepciones

Un `abstract record` con un `sealed record` por situación que la UI debe distinguir. El servicio **nunca lanza**; el ViewModel hace `switch` sin `try/catch`.

`GpsResult` (8 casos) · `CallResult` (7) · `NetworkResult` (6) · `PrintResult` + `DiscoverResult` + `DocumentResult` · `ApiCallResult`.

> Regla que el código deja escrita: **cada variante debe ser alcanzable**. Modelar un caso y no producirlo nunca es «el defecto más caro del patrón» — `BluetoothOff` existió todo un PoC sin que nadie la construyera.

### 6.2 Overlay de estado sobre el contenido

Una máquina de tres estados (`None` / `Busy` / `Error`) con capa de espera, capa de error y **botonera dinámica** generada desde una colección de `OverlayAction(Text, Command, Style)`. Vive en `StatusOverlayViewModel` + `StatusOverlayView`.

Variante temprana en `Ejemplo_Maui_GPS` y `Ejemplo_Maui_DirectCall` (dos `bool` y callbacks por constructor); versión madura en `Ejemplo_MotorDSL_Dialog` y en los cuatro overlays de la híbrida.

### 6.3 Catálogo de errores con código estable

`{Dominio}Failure(Code, Title, UserMessage, TechnicalMessage)` + un `{Dominio}ErrorCatalog` estático. Prefijos: **`GPS-*`**, **`TEL-*`**, **`PRN-*`**.

Dos reglas declaradas: los códigos **son contrato con soporte** (se agregan, no se renombran) y hay **un código por acción distinta del usuario**, no por causa técnica distinta. `DisplayMessage` pone el código separado y al final, para que se pueda dictar a soporte sin competir con la acción.

### 6.4 Coordinador dueño del overlay y de la cancelación

Un singleton (`GpsCoordinator`, `CallCoordinator`, o el propio `*OverlayViewModel` en la híbrida) que es **el punto único de entrada** desde cualquier parte de la app: página, comando, deep link o navegación por URL. Es dueño del `CancellationTokenSource` y hace todo cambio de UI por `MainThread`/`IUiDispatcher`.

### 6.5 Permisos normalizados a cuatro casos

`Granted` / `DeniedCanRetry` / `Denied` / `Restricted`. `DeniedCanRetry` sale de `ShouldShowRationale`, que **solo existe en Android**; en iOS siempre se cae en `Denied`, que fuerza el camino de ajustes. Aparece en `LocationPermissionResult`, `CallPermissionResult` y `BluetoothPermissionResult`.

## 7. La app integrada como destino

Los dominios aislados son los ensayos; `Ejemplo_Maui_Hibrida` es la obra. Cada dominio llega ahí con una versión endurecida:

| Dominio | Qué aporta a la híbrida |
|---------|------------------------|
| Cámara | `MyMediaPickerPage` y `MyMediaSelfiePickerPage` completas, más la normalización EXIF |
| QR | `QRLectorPage` con `BarcodeScanning.Native.Maui`, la recomendación 🥇 del estudio de NuGets |
| Impresión | El overlay con catálogo `PRN-*`, más el caso «documento por red» que el PoC no tenía |
| GPS | El servicio y el overlay, ahora con catálogo `GPS-*` y `IUiDispatcher` |
| Teléfono | El servicio de llamada directa, con catálogo `TEL-*` |
| Red | `NetworkService` con **sonda activa** de internet real (el ejemplo aislado es un esqueleto) |

Lo que la híbrida agrega y no existe en ningún ejemplo aislado: el **puente de comandos por URL** (`LibApp/UrlCommands/`), el **bridge del WebView** y la **suite de tests con invariantes ejecutables**.

## 8. Decisiones estructurales

| Decisión | Consecuencia |
|----------|--------------|
| **Cada ejemplo autocontenido** (ADR-0001) | No hay proyecto de librería compartida; el `.csproj` de tests **linkea fuentes** en vez de referenciar el proyecto |
| **Duplicación deliberada de tipos con la misma forma** | `GpsFailure`/`CallFailure`/`PrintFailure` no se unificaron: unificarlos obligaba a reabrir el dominio ya validado en dispositivo. «Lo que armoniza el patrón son los invariantes, no un tipo compartido» |
| **Los `MotorDsl.*` se consumen como NuGet, no como `ProjectReference`** | El ejemplo valida los paquetes publicados |
| **Los ViewModels no tienen ninguna directiva `#if`** | Todo el código de plataforma vive detrás de `I*Service`, y por eso la suite corre en `net10.0` plano sin emulador |
| **CI solo para simulador iOS** | Android, la plataforma principal de casi todos los ejemplos, no se compila en CI |

## 9. Números del repositorio

| | |
|---|---|
| Proyectos (`.csproj`) | 24 · en la solución: 23 |
| Archivos `.cs` (sin `bin`/`obj`) | 322 |
| Archivos `.xaml` | 129 · `.razor`: 12 |
| Documentos `.md` | 62 |
| Workflows de CI | 18 |
| Tests | 67 `[Fact]` + 11 `[Theory]` (solo en la híbrida) |
| Entradas del CHANGELOG | 9 (2026-07-13 → 2026-07-23) |

## 10. Dónde mirar primero

| Si necesitás… | Leé |
|---------------|-----|
| Entender el repositorio como sistema | este índice |
| Trabajar sobre un dispositivo concreto | el índice del dominio (01–07) |
| Tocar la app final | [08](08_App-Hibrida-Integrada.md), y el índice del dispositivo involucrado |
| Cambiar el pipeline o la simulación | [09](09_CI-CD-y-Build.md) |
| Encontrar el documento que explica un porqué | [10](10_Documentacion-Transversal.md) |
