# 00 — Índice maestro de `Ejemplos_Maui_Devices`

> **Propósito:** visión general del repositorio de código: qué es, con qué stack, qué proyectos hay, qué patrones se repiten y dónde está cada tema. Es el índice a leer primero; el detalle de cada dominio vive en los índices `01`–`10`.
> **Fuentes primarias:** `README.md`, `CHANGELOG.md`, `Ejemplos_Devices/Ejemplos_Devices.slnx`, los `*.csproj` de `Ejemplos_Devices/`, `.github/workflows/`. Origen indexado: HEAD local `cc80ce1` (ver manifiesto en [`../README.md`](../README.md)).

---

## 1. Qué es

| | |
|---|---|
| Naturaleza | Colección **didáctica** de ejemplos .NET MAUI de acceso a dispositivos, uno por técnica, más una app híbrida que los integra (`README.md:2-24`) |
| Dominios | Cámara, QR, impresión térmica Bluetooth, GPS, mapas, telefonía, red, app híbrida integrada (carpetas de `Ejemplos_Devices/`; `Ejemplos_Devices.slnx:36-188`) |
| Estado de la prosa del código | El `README.md` raíz marca casi todos los dominios «En construcción ...» (`README.md:12-24`); la prosa completa está en el repo de documentación → [10](10_Documentacion-Transversal.md) |
| Bitácora del código | `CHANGELOG.md` (Keep a Changelog; última entrada 2026-07-23, `CHANGELOG.md:1-6`) |

## 2. Stack

| Capa | Valor | Cita |
|---|---|---|
| Lenguaje / UI | C# · .NET MAUI · XAML | `*.csproj` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Ejemplo_Maui_GPS.csproj:4-5`) |
| TFM apps MAUI | `net10.0-android`; `+net10.0-ios` sólo al compilar en macOS | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Ejemplo_Maui_GPS.csproj:4-5`, `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Ejemplo_Maui_Hibrida.csproj:4-5` |
| Plataformas mínimas | iOS 15.0, Android 25 (DirectCall: 21 → [06](06_Telefonia.md)) | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Ejemplo_Maui_GPS.csproj:25-26` |
| Backend | Blazor Server `net10.0` (`Ejemplo_ws_Blazor`) | `Ejemplos_Devices/Integrada/Ejemplo_ws_Blazor/Ejemplo_ws_Blazor.csproj:4` |
| Tests | xUnit `net10.0` (`Ejemplo_Maui_Hibrida.Tests`) | `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida.Tests/Ejemplo_Maui_Hibrida.Tests.csproj:9` |
| SDK fijado | Ninguno: no hay `global.json` ni `Directory.Build.*`; `$(MauiVersion)` lo pone el workload | → [09 §1](09_CI-CD-y-Build.md) |
| CI | GitHub Actions, `macos-15`, Xcode 26.0, .NET SDK 10.0.300 | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:23-47` → [09](09_CI-CD-y-Build.md) |

## 3. Catálogo de proyectos

25 `*.csproj` versionados: 24 proyectos reales + 1 copia histórica en prosa (`Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/Prompts_Generacion_Apk_ejemplo/ficheros_claude.ia/Thermalprintermaui.csproj`, no se compila). 23 están en la solución.

| Dominio | Proyecto | En `.slnx` | Índice |
|---|---|---|---|
| Cámara | `Ejemplo_Photo_MediaPicker` · `_MiMediaPicker_Callback` · `_MiMediaPicker_Callback_Normalizacion` · `_MiMediaPicker_Task` · `_MiMediaSelfie_Callback_Normalizacion` | `:37`, `:40`, `:43`, `:46`, `:49` | [01](01_Camara.md) |
| QR | `BSM` · `BSN` · `CS` · `ZN` `.LectorQR`, cada uno con `_Dialog` | `:150-183` | [02](02_QR.md) |
| Impresión | `Ejemplo_MotorDSL` · `Ejemplo_MotorDSL_Dialog` · `Ejemplo_ThermalPrinter` | `:90`, `:93`, `:96` | [03](03_Impresion-Termica.md) |
| GPS | `Ejemplo_Maui_GPS` | `:57` | [04](04_GPS.md) |
| Mapas | `Ejemplo_Maui_Mapas` | `:74` | [05](05_Mapas.md) |
| Telefonía | `Ejemplo_Maui_Dialer` · `Ejemplo_Maui_DirectCall` | `:82`, `:85` | [06](06_Telefonia.md) |
| Red | `Ejemplo_Maui_Connectivity` | `:188` | [07](07_Red-Conectividad.md) |
| Integrada | `Ejemplo_Maui_Hibrida` · `Ejemplo_ws_Blazor` · `Ejemplo_Maui_Hibrida.Tests` | `:65`, `:68`; Tests **no** | [08](08_App-Hibrida-Integrada.md) |

(Líneas de `Ejemplos_Devices/Ejemplos_Devices.slnx`.) Las carpetas `Ejemplo_Docs_*` de cada dominio son prosa/material, no proyectos.

## 4. Mapa del árbol

```
Ejemplos_Maui_Devices/
├── README.md · CHANGELOG.md · .gitignore · vs.bat
├── .github/workflows/        18 cd-ios-*.yml + Readme.md + Pipelinea-Version.md + Analisis/   → 09
├── Utilities/                simular.sh, simular_ui.sh, end2end/ (Maestro), download_sectigo.ps1 → 09
└── Ejemplos_Devices/
    ├── Ejemplos_Devices.slnx
    ├── Camera/ QR/ Printer/ GPS/ Maps/ Phone/ Red/   un ejemplo por técnica         → 01–07
    ├── Integrada/            app híbrida + backend Blazor + tests                   → 08
    ├── Docs/                 Certificados-SSL, otros, qr-nuget, web-hibrida         → 10
    └── scripts/              .bat de arranque Windows                                → 09
```

## 5. Patrones transversales

| Patrón | Dónde se ve | Índice |
|---|---|---|
| Servicio → resultado tipado → overlay MVVM | GPS: `GpsService` → `GpsResult` → `GpsCoordinator.Aplicar` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GpsService.cs:15-69`, `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GpsCoordinator.cs:56-70`); DirectCall: `CallResult`/`CallCoordinator` (`Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Services/CallResult.cs:6-28`); impresión: `PrinterService` → `PrintResult` → `PrinterOverlayViewModel` | [04](04_GPS.md) · [06](06_Telefonia.md) · [03](03_Impresion-Termica.md) · ADR 0002 |
| Dos generaciones de overlay | `ViewModelBase` + `AsyncRelayCommand` propios (GPS, DirectCall) vs base genérica de tres estados sobre CommunityToolkit (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/ViewModels/StatusOverlayViewModel.cs:11-77`) | [03 §7](03_Impresion-Termica.md) · [04 §6](04_GPS.md) |
| Permiso normalizado en 4 valores | `Granted`/`DeniedCanRetry`/`Denied`/`Restricted` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/LocationPermissionResult.cs:5-11`, `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/BluetoothPermissionResult.cs:5-11`) | [04 §3](04_GPS.md) · [03 §5](03_Impresion-Termica.md) |
| Retorno de una página por `TaskCompletionSource` | QR `_Dialog` (`Ejemplos_Devices/QR/BSM.LectorQR_Dialog/Pages/QRLectorPage.xaml.cs:12`), cámara ej. 3 | [02 §5](02_QR.md) · [01 §3](01_Camara.md) |
| Registro en `MauiProgram` | QR/Mapas: sólo el handler de la librería (`.UseBarcodeScanning()`, `.UseMauiMaps()`); GPS/impresión: extensiones `AddServices`/`AddPrintServices` | [02 §3](02_QR.md) · [05 §2](05_Mapas.md) |
| Catálogo de errores con código estable | `PRN-*`, `DisplayMessage` = mensaje + código (`Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog/Models/PrinterErrorCatalog.cs:9-39`) | [03 §6](03_Impresion-Termica.md) |
| Puente WebView por comandos URL | intercepción de la navegación en la híbrida; clasificación separada de ejecución desde `0ef370b` | [08 §3](08_App-Hibrida-Integrada.md) · ADR 0003 |
| Secretos fuera del repo | `Services/ApiKeys.cs` generado desde plantilla, ignorado por `.gitignore:421`; en CI desde `secrets.GOOGLE_MAPS_API_KEY` | [04 §5](04_GPS.md) · [09 §4](09_CI-CD-y-Build.md) · ADR 0007 |

## 6. Decisiones (ADR) — dónde

Los 9 ADR viven en el repo de documentación: [`04-decisions/`](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/). Tabla con estado y decisión en una línea → [10 §2](10_Documentacion-Transversal.md).

## 7. Avisos de mayor alcance (⚠️)

Detalle y citas en cada índice; acá sólo los que cruzan dominios.

- ⚠️ **Clave de Google Maps en texto plano** en el `AndroidManifest.xml` de `Ejemplo_Maui_Mapas` (no se copia acá) → [05 §5](05_Mapas.md); ya registrada en el GAP-REPORT (S-01).
- ⚠️ **`.slnx` con rutas muertas**: `GPS/Docs_GPS/Readme.md` (`Ejemplos_Devices.slnx:62`) y `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md` (`:111`); `/Docs/web-hibrida/` vacía (`:34`) → [09 §9](09_CI-CD-y-Build.md).
- ⚠️ **Workflow huérfano** `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` apunta a un proyecto inexistente → [09 §2](09_CI-CD-y-Build.md).
- ⚠️ **Los tests de la híbrida no corren en CI** ni están en la solución → [08 §9](08_App-Hibrida-Integrada.md) · [09](09_CI-CD-y-Build.md).
- ⚠️ **Prosa anterior a `0ef370b`**: bridge-contract, pieza integrada y `Docs/web-hibrida/maui-hibrido.md` describen el despacho viejo del puente; manda el código → [08 §10](08_App-Hibrida-Integrada.md).
- ⚠️ **Nombre ≠ librería en QR**: `ZN.LectorQR` usa `CameraMaui`, no ZXing.Net.Maui → [02 §2](02_QR.md).
- ⚠️ **Versiones no unificadas**: `Microsoft.Maui.Controls` va de `$(MauiVersion)` a fijos 10.0.30–10.0.80 según el proyecto; `MotorDsl.*` 1.0.12 vs 1.0.13 → [01](01_Camara.md) · [02](02_QR.md) · [03 §1](03_Impresion-Termica.md).

## 8. Fuentes

| Ruta | Contiene |
|---|---|
| `README.md` | Portada con enlaces a los `Ejemplo_Docs_*` |
| `CHANGELOG.md` | Bitácora de cambios del código |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | Solución: carpetas y 23 proyectos |
| `Ejemplos_Devices/**/*.csproj` | TFM, versiones mínimas, paquetes |
| `.github/workflows/` | Pipelines → [09](09_CI-CD-y-Build.md) |

**Prosa relacionada:** [README del conjunto documental](../../Ejemplos_Maui_Devices-docs/README.md) · [system-map](../../Ejemplos_Maui_Devices-docs/docs/00-overview/system-map.md) · [vision](../../Ejemplos_Maui_Devices-docs/docs/00-overview/vision.md) · [GLOSSARY](../../Ejemplos_Maui_Devices-docs/GLOSSARY.md).
