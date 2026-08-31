# Índice 09 — CI/CD y Build

> **Propósito**: describir cómo se compila, firma y publica cada ejemplo en CI (GitHub Actions, iOS/simulador), el versionado del pipeline y las utilidades de arranque local.
> **Fuente primaria**: `.github/workflows/`.
> **Entrada ia-db**: [README](../README.md) · [Índice maestro](00_MASTER-INDEX.md)

---

## 1. Panorama

- El repo publica en CI **solo iOS (simulador)**. Cada ejemplo tiene **su propio workflow** reutilizable (`workflow_call`) en `.github/workflows/`.
- El arranque **local** es distinto: los `.bat` de `Ejemplos_Devices/scripts/` despliegan a **Android físico** (`net10.0-android -t:Run`).
- Convención de nombre de workflow: **`cd-ios-<categoria>.<Ejemplo>.yml`** (categorías: `camera`, `gps`, `phone`, `printer`, `qr`, `Integrada`).
- Runner CI: `macos-15` (Apple Silicon). SDK .NET instalado en el runner: **10.0.300**; workload manifest: **10.0.100**; target: **net10.0-ios**. (Ver §5.)

```
.github/workflows/
├── cd-ios-camera.*.yml    (4)   ┐
├── cd-ios-gps.*.yml       (1)   │
├── cd-ios-Integrada.*.yml (1)   │
├── cd-ios-phone.*.yml     (2)   ├─ 18 workflows reutilizables (workflow_call)
├── cd-ios-printer.*.yml   (1)   │
├── cd-ios-qr.*.yml        (9)   ┘
├── Readme.md                   # versiones de workload .NET (dev local)
├── Pipelinea-Version.md        # bitácora de versiones del yml
└── Analisis/                   # logs de ejecuciones CI (log_1.md, log_2.md)
```

---

## 2. Convención de nombres y disparo

| Elemento | Valor | Fuente |
|---|---|---|
| Patrón de archivo | `cd-ios-<categoria>.<Ejemplo>.yml` | listado `.github/workflows/` |
| Categorías | `camera`, `gps`, `phone`, `printer`, `qr`, `Integrada` | prefijos de los `.yml` |
| Trigger activo | **`workflow_call:`** en los 18; **`push` activo solo en `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml`** | p.ej. `cd-ios-qr.CS.LectorQR.yml:15`; `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:5-14` |
| Trigger `push` | **comentado en 17 de 18** (branch `main`, con `paths:` por carpeta del ejemplo) | `cd-ios-qr.CS.LectorQR.yml:5-13` |
| `name:` interno | `CD iOS - <Categoria> - <Ejemplo>` | `cd-ios-gps.Ejemplo_GPS.yml:1` |
| Job | `publish-ios` sobre `runs-on: macos-15` | `cd-ios-gps.Ejemplo_GPS.yml:50-51` |

> Los 17 workflows de camera/gps/phone/printer/qr son solo `workflow_call`: se invocan desde un orquestador externo o manualmente y **no** disparan por sí mismos con push (bloque `push:` comentado). No hay un `cd-main.yml` orquestador presente en el árbol. **Excepción**: `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` tiene el `push` **activado** sobre `main` filtrado a `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/**` (excluyendo `**.md`, `.gitignore`, `.gitattributes`), de modo que el flujo de simulación end2end corre automáticamente ante cambios de la app híbrida (ver §4.2).

---

## 3. Workflows agrupados por categoría

### 3.1. Resumen por categoría

| Categoría | Nº workflows | `PROJECTS_ROOT` (carpeta) | Notas |
|---|---|---|---|
| **camera** | 4 | `Camera` | MediaPicker/Selfie (callback, task, normalización) |
| **gps** | 1 | `GPS` | solo `Ejemplo_GPS` (la híbrida se movió a la categoría `Integrada`) |
| **Integrada** | 1 | `Integrada` | `Ejemplo_Maui_Hibrida` — categoría propia desde `cd-ios-Integrada.*`; único con `push` activo y con simulación **end2end/Maestro** (§4.2) |
| **phone** | 2 | `Phone` | Dialer y DirectCall |
| **printer** | 1 | `Printer` | MotorDSL (impresión térmica) |
| **qr** | 9 | `QR` | 4 librerías QR × {simple, dialog} + genérico |
| **Total** | **18** | | |

### 3.2. Detalle (ejemplos cubiertos)

| Categoría | Workflow (`.yml`) | `PROJECT_NAME` | `PACKAGE_NAME` | RID simulador |
|---|---|---|---|---|
| camera | `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback.yml` | `Ejemplo_Photo_MiMediaPicker_Callback` | `com.ejemplos.photo.mimediapicker.callback` | arm64 |
| camera | `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.yml` | `Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion` | arm64 |
| camera | `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Task.yml` | `Ejemplo_Photo_MiMediaPicker_Task` | `com.ejemplos.devices.mimediapicker.task` | arm64 |
| camera | `cd-ios-camera.Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion.yml` | `Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | `com.ejemplos.devices.imagen.callback.normalizacion.miselfie` | arm64 |
| gps | `cd-ios-gps.Ejemplo_GPS.yml` | `Ejemplo_Maui_GPS` | `com.ejemplos.devices.gps` | arm64 |
| Integrada | `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` | `Ejemplo_Maui_Hibrida` | `com.ejemplos.devices.integrada.hibrida` | arm64 |
| phone | `cd-ios-phone.Ejemplo_Dialer.yml` | `Ejemplo_Maui_Dialer` | `com.ejemplos.phone.dialer` | arm64 |
| phone | `cd-ios-phone.Ejemplo_DirectCall.yml` | `Ejemplo_Maui_DirectCall` | `com.ejemplos.phone.directcall` | arm64 |
| printer | `cd-ios-printer.Ejemplo_MotorDSL.yml` | `Ejemplo_MotorDSL` | `com.ejemplos.devices.MotorDSL` | arm64 |
| qr | `cd-ios-qr.BSM.LectorQR.yml` | `BSM.LectorQR` | `com.ejemplos.devices.qr.barcodescanner_mobile_maui.simple` | **x64** |
| qr | `cd-ios-qr.BSM.LectorQR_Dialog.yml` | `BSM.LectorQR_Dialog` | `com.ejemplos.devices.qr.barcodescanner_mobile_maui.dialog` | **x64** |
| qr | `cd-ios-qr.BSN.LectorQR.yml` | `BSN.LectorQR` | `com.ejemplos.devices.qr.barcodescanner_native_maui.simple` | arm64 |
| qr | `cd-ios-qr.BSN.LectorQR_Dialog.yml` | `BSN.LectorQR_Dialog` | `com.ejemplos.devices.qr.barcodescanner_native_maui.dialog` | arm64 |
| qr | `cd-ios-qr.CS.LectorQR.yml` | `CS.LectorQR` | `com.ejemplos.devices.qr.camerascanner_maui.simple` | arm64 |
| qr | `cd-ios-qr.CS.LectorQR_Dialog.yml` | `CS.LectorQR_Dialog` | `com.ejemplos.devices.qr.camerascanner_maui.dialog` | arm64 |
| qr | `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` | `Ejemplo_LectorQR_Dialog` | `com.ejemplos.devices.qr.dialog` | arm64 |
| qr | `cd-ios-qr.ZN.LectorQR.yml` | `ZN.LectorQR` | `com.ejemplos.devices.qr.zxing_net_maui.simple` | arm64 |
| qr | `cd-ios-qr.ZN.LectorQR_Dialog.yml` | `ZN.LectorQR_Dialog` | `com.ejemplos.devices.qr.zxing_net_maui.dialog` | arm64 |

**Prefijos QR** (librería usada; detalle en índice 02): `BSM` = BarcodeScanner.Mobile.Maui · `BSN` = BarcodeScanner Native Maui · `CS` = CameraScanner Maui · `ZN` = ZXing.Net.Maui.

> **Caso especial `BSM` → RID `iossimulator-x64`.** Solo los dos workflows `BSM` usan `RUNTIME_IDENTIFIER_SIMULATOR: iossimulator-x64` (los demás, `arm64`). Motivo: `BarcodeScanner.Mobile.Maui` usa Google ML Kit, que **no publica slice de simulador arm64**; se compila x86_64 y se ejecuta bajo Rosetta 2. Análisis completo en `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md` (ver índice 10). En esos workflows se activa el step `ROSETTA. Instalar traductor` y la descarga del runtime `-architectureVariant universal`.

---

## 4. Pipeline iOS estándar

Todos los workflows comparten la **misma secuencia** de steps (verificado en `cd-ios-qr.CS.LectorQR.yml`, `cd-ios-gps.Ejemplo_GPS.yml`, `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback.yml`). Difieren solo en el bloque `env:` (datos del proyecto) y en dos condicionales (ApiKeys, Rosetta). La única variante estructural es la del ejemplo `Integrada` en el tramo final de simulación (§4.2).

```mermaid
flowchart TD
    A[Verificar arquitectura runner] --> B[actions/checkout@v4]
    B --> C[XCODE: pipx install gdown]
    C --> D[XCODE: descargar Xcode .xip desde Google Drive]
    D --> E[XCODE: xip --expand + instalar en /Applications/Xcode_26.0.app]
    E --> F[XCODE: xcode-select --switch, license accept]
    F --> G[XCODE: -downloadPlatform iOS + boot iPhone 17 Pro Max]
    G --> H[.NETCORE: dotnet-install.sh --version 10.0.300]
    H --> I[WORKLOADS: install ios maui maui-ios --version 10.0.100]
    I --> J[PATH: calcular SOLUTION_PATH / PROJECT_PATH / PLIST_PATH]
    J --> K{PRE-BUILD ApiKeys.cs?}
    K -->|solo GPS| K1[sed ApiKeys.cs.template + secret GOOGLE_MAPS_API_KEY]
    K -->|resto| L
    K1 --> L[VERSIONADO: PlistBuddy CFBundleVersion/Short + fecha]
    L --> M[MANIFEST: plutil -lint Info.plist]
    M --> N[BUILD: dotnet clean + restore + build -c Release -f net10.0-ios]
    N --> O[FIRMA robusta: codesign ad-hoc '-' dll/dylib/aotdata + entitlements + bundle]
    O --> P[PUBLISH: zip .app -> upload-artifact@v4 retention 1d]
    P --> Q{RID x64?}
    Q -->|BSM| Q1[ROSETTA: softwareupdate --install-rosetta]
    Q -->|resto| R
    Q1 --> R[brew install ffmpeg]
    R --> S[SIMULAR: Utilities/simular.sh -> GIF evidencia, continue-on-error, timeout 30m]
    S --> T[Subir GIF + limpiar keychain/perfiles]
```

### 4.1. Pasos clave (con comando representativo)

| Fase | Qué hace | Referencia |
|---|---|---|
| Xcode | Descarga el instalador `Xcode_26_Universal.xip` desde Google Drive con `gdown` (id `XCODE_GOOGLE_FILE_INSTALLER_ID`), lo instala y selecciona | `cd-ios-qr.CS.LectorQR.yml:91-131` |
| Simulador | `sudo xcodebuild -downloadPlatform iOS` (para x64: `-architectureVariant universal`); `simctl boot "iPhone 17 Pro Max"` | `cd-ios-qr.CS.LectorQR.yml:147-164` |
| .NET | Borra `~/.dotnet` e instala `DOTNET_VERSION` con `dotnet-install.sh` | `cd-ios-gps.Ejemplo_GPS.yml:155-182` |
| Workloads | `dotnet workload install ios maui maui-ios --version 10.0.100` | `cd-ios-gps.Ejemplo_GPS.yml:198-210` |
| ApiKeys (GPS) | Genera `Services/ApiKeys.cs` desde `.template` inyectando `secrets.GOOGLE_MAPS_API_KEY` con `sed` | `cd-ios-gps.Ejemplo_GPS.yml:279-285` |
| Versionado | `PlistBuddy` lee `CFBundleVersion`/`CFBundleShortVersionString`; `date +%Y%m%d%H%M` | `cd-ios-qr.CS.LectorQR.yml:272-289` |
| Build | `dotnet build -c Release -f net10.0-ios -p:RuntimeIdentifier=<RID> -p:LinkMode=SdkOnly -p:EnableCodeSigning=false` | `cd-ios-qr.CS.LectorQR.yml:317-339` |
| Firma | `codesign --force --sign "-" --timestamp=none` sobre `*.dll/*.dylib/*.aotdata*`, binario (con entitlements) y bundle → **ad-hoc** (`Signature=adhoc`) | `cd-ios-qr.CS.LectorQR.yml:357-387` |
| Publicación | `zip -ry` del `.app` con nombre `<fecha>_<version>_<package>.app.zip` → `actions/upload-artifact@v4` (`retention-days: 1`) | `cd-ios-qr.CS.LectorQR.yml:399-418` |
| Evidencia | `brew install ffmpeg` + `Utilities/simular.sh` graba video → GIF (`continue-on-error`, `timeout-minutes: 30`) | `cd-ios-qr.CS.LectorQR.yml:431-463` |

> El build **desactiva la firma real** (`EnableCodeSigning=false`, `CodesignKey="-"`) y luego re-firma **ad-hoc** manualmente: es un smoke test de simulador, **no** genera IPA firmado para distribución. El entregable es el `.app.zip` + un GIF de evidencia con retención de 1 día.

### 4.2. Variante `Integrada` — simulación end2end con «dedo virtual»

`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` sigue el pipeline estándar hasta la publicación del `.app.zip`, pero **reemplaza el tramo de evidencia**: en vez de `Utilities/simular.sh` → GIF, ejecuta un recorrido automatizado sobre la UI real y sube un **video**.

| Elemento | Valor | Fuente |
|---|---|---|
| `SCRIPT_SIMULATOR` | `./Utilities/simular_ui.sh` (los demás: `simular.sh`) | `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:48` |
| `MAESTRO_VERSION` | `1.41.0` (env exclusiva de este workflow) | `…Hibrida.yml:49` |
| Step extra | «Instalar Maestro (dedo virtual)»: `curl -Ls https://get.maestro.mobile.dev \| bash` + `$HOME/.maestro/bin` al `PATH` | `…Hibrida.yml:438-443` |
| Step de evidencia | «RELEASE SIMULADOR. GRABAR VIDEO (recorrido automatizado)» — `continue-on-error`, `timeout-minutes: 30` | `…Hibrida.yml:445-455` |
| Artefacto | `recorrido.mp4` + `debug_logs/` + `app_stream_full.txt` (`upload-artifact@v5`, retención 1 día) | `…Hibrida.yml:457-466` |
| Sin `brew install ffmpeg` | el video sale directo de `simctl io recordVideo` (no hay conversión a GIF) | `…Hibrida.yml` (ausencia del step) |

**Arranque del simulador (endurecido).** El boot *headless* por `simctl` se cuelga en «Waiting on BackBoard» porque no levanta BackBoard/SpringBoard; por eso:

- El step «Verificando simulador instalado» hace **precalentado por GUI, fire-and-forget**: `open -a "$(xcode-select -p)/Applications/Simulator.app"` + `xcrun simctl boot "$DEVICE_SIMULATOR" || true`, para que el simulador arranque durante el build (`…Hibrida.yml:161-171`).
- `Utilities/simular_ui.sh` repite el patrón con cota: short-circuit si ya está `Booted`; si no, `bootstatus -b` con **240 s** → `shutdown`+`erase`+reapertura de GUI → reintento de **300 s**; si falla, vuelca `~/Library/Logs/CoreSimulator/CoreSimulator.log`. Peor caso ~9 min (antes, cuelgue de 30 min).
- Antes de grabar hace **pre-warm** (lanza la app + `sleep 45`) y recién después arranca `simctl io … recordVideo --codec h264`, de modo que el arranque en frío no consume el video; el flujo Maestro usa `launchApp: { stopApp: false }` y espera activa (`extendedWaitUntil` sobre el botón `Geo Pos`, 120 s) en vez de un `waitForAnimationToEnd` fijo.

> **Alcance.** El detalle interno de `Utilities/simular_ui.sh` y del flujo `Utilities/end2end/<PACKAGE_NAME>.yaml` **no se indexa** (decisión vigente desde v1.2); aquí se documenta solo su acoplamiento con el workflow. Descripción funcional para humanos: [`../../Ejemplos_Maui_Devices-docs/README.md`](../../Ejemplos_Maui_Devices-docs/README.md) § Pruebas End2End.

---

## 5. Versiones (CI vs. desarrollo local)

| Ámbito | SDK .NET | Workload | Xcode / runtime | Fuente |
|---|---|---|---|---|
| **CI (runner)** | `DOTNET_VERSION: 10.0.300` (alt. comentada `10.0.102`) | `DOTNET_VERSION_WORKLOAD: 10.0.100` (`ios maui maui-ios`) | `XCODE_VERSION: 26.0`, `--ios-simulator-runtime=26.0`, `macos-15` | `cd-ios-qr.CS.LectorQR.yml:25-51` |
| **Dev local** | `dotnet --version` = **10.0.102** | workload **10.0.102** (android `36.1.12`, ios `26.2.10191`, maccatalyst, maui-*) | — | `.github/workflows/Readme.md` |
| Target framework | `DOTNET_TARGET_VERSION: net10.0` → `net10.0-ios` | | | `cd-ios-gps.Ejemplo_GPS.yml:30` |

- `.github/workflows/Readme.md` documenta las salidas de `dotnet workload --version` / `list` / `search maui` en la máquina de desarrollo (para configurar el yml).
- Nota de discrepancia deliberada: el runner instala SDK **10.0.300** con manifest de workload **10.0.100**, mientras que el entorno local corre **10.0.102**. El target de compilación es `net10.0-ios` en ambos.

### 5.1. Bitácora del pipeline — `Pipelinea-Version.md`

`PIPELINE_VERSION` es un sello `AAAAMMDDhhmm_ejemplos` presente **solo** en los workflows QR y en `Ejemplo_Maui_Hibrida` (los de camera/gps/phone/printer lo omiten).

| Versión | Cambio | Fuente |
|---|---|---|
| `202606261239_ejemplos` | Primera versión de yml estandarizada; mejoras en rutas relativas | `Pipelinea-Version.md:13-15` |
| `202606262101_ejemplos` | Simulador Rosetta (x64) para la librería QR; parametrización del script de simulación | `Pipelinea-Version.md:2-11` |
| `202606300809` / `0822` / `301040_ejemplos` | Sellos vigentes en los `.yml` QR / `_Dialog` / `Hibrida` | `env: PIPELINE_VERSION` de cada `.yml` |

### 5.2. Logs de ejecución — `.github/workflows/Analisis/`

| Archivo | Contenido |
|---|---|
| `Analisis/log_1.md` | Log CI extenso (~0.5 MB) de una corrida de build iOS |
| `Analisis/log_2.md` | Traza del step de simulación de `Ejemplo_LectorQR_Dialog` (RID `x64`): firma `Signature=adhoc`, boot del simulador, e instalación OK, pero el step **timeout a los 30 min** en `simctl privacy`/lanzamiento (`log_2.md:84-102`) |

---

## 6. Utilidades de arranque local

Contraste clave: **CI compila iOS/simulador; los scripts locales despliegan a Android físico.**

| Script | Ubicación | Qué hace | Fuente |
|---|---|---|---|
| `Ejemplo_Maui_GPS_launch.bat` | `Ejemplos_Devices/scripts/` | `dotnet build ...GPS\Ejemplo_Maui_GPS.csproj -f net10.0-android -t:Run`; verifica dispositivo con `adb devices` (usa `platform-tools`) | script:6-37 |
| `Ejemplo_Photo_MediaPicker_launch.bat` | `Ejemplos_Devices/scripts/` | Igual patrón para `Camera\Ejemplo_Photo_MediaPicker.csproj` (nota: el banner `echo` aún dice "Ejemplo_Maui_GPS" por copy/paste) | script:6, 23 |
| `GetEnviromentVersion.bat` | `Ejemplos_Devices/scripts/` | `dotnet --version` + `dotnet workload list --verbosity detailed` | script:1-3 |
| `vs.bat` | raíz del repo | `code .` (abre VS Code en el repo) | `vs.bat:1` |
| `Ejemplos_Devices.slnLaunch` | `Ejemplos_Devices/` | Perfil de arranque múltiple de Visual Studio **«Hibrida-Con-WS»**: lanza a la vez `Integrada/Ejemplo_Maui_Hibrida` (target `Motorola moto g42 (Android 13.0 - API 33)`) y `Integrada/Ejemplo_ws_Blazor` (target `https`) — es la forma de correr la híbrida contra su backend local | `Ejemplos_Devices.slnLaunch:1-16` |

> El `.slnLaunch` está **sin versionar** (archivo *untracked*, no ignorado por `.gitignore`) y su `DebugTarget` de Android nombra un dispositivo concreto: es configuración local de la máquina de desarrollo, no del repo. Se indexa por documentar el par de proyectos que la app híbrida necesita en simultáneo (índice 08).

- Patrón `.bat` de deploy: `SCRIPT_DIR=%~dp0` para ruta relativa al `.csproj`; `EnableDelayedExpansion`; agrega `platform-tools` (`adb.exe`) al `PATH`; si no hay dispositivo conectado, aborta con `[ERROR]`.
- Doc asociada: `Ejemplos_Devices/Docs/otros/GetEnviromentVersion.md` (prerequisitos: 7-Zip, NuGets, `dotnet sdk check`) — se detalla en el índice 10.

---

## 7. Seguridad de claves — `.gitignore` raíz

| Regla | Efecto | Fuente |
|---|---|---|
| **`**/Services/ApiKeys.cs`** | Las claves API **no se versionan**; el archivo se genera en build. | `.gitignore:420-421` |
| `*.env` | Variables de entorno fuera del repo | `.gitignore:12` |
| `*.pfx`, `*.publishsettings` | Certificados/credenciales de publicación excluidos | `.gitignore:255-256` |

> Ciclo completo de la clave: `.gitignore` excluye `Services/ApiKeys.cs`; en el repo vive `ApiKeys.cs.template` con `REEMPLAZAR_CON_TU_API_KEY`; el step **PRE-BUILD** del workflow GPS reemplaza el placeholder con `secrets.GOOGLE_MAPS_API_KEY` vía `sed` justo antes de `dotnet restore`. En los workflows de camera este step está **comentado** (no requieren clave).

El `.gitignore` es la plantilla estándar `VisualStudio.gitignore` (ignora `bin/`, `obj/`, `.vs/`, `*.binlog`, etc.) con la única regla de seguridad propia añadida al final.

---

## 8. Trazabilidad rápida

| Afirmación | Fuente project-relative |
|---|---|
| 18 workflows, patrón `cd-ios-<cat>.<Ejemplo>.yml` | `.github/workflows/*.yml` |
| Pipeline estándar (steps) | `.github/workflows/cd-ios-qr.CS.LectorQR.yml` |
| Variante end2end/Maestro + `push` activo | `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:5-14, 48-49, 161-171, 438-466` |
| Variante con ApiKeys | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:279-285` |
| BSM usa RID x64 (Rosetta) | `.github/workflows/cd-ios-qr.BSM.LectorQR.yml:44-45` |
| Versiones dev local | `.github/workflows/Readme.md` |
| Bitácora del pipeline | `.github/workflows/Pipelinea-Version.md` |
| Logs de corrida | `.github/workflows/Analisis/log_1.md`, `log_2.md` |
| Scripts locales (Android) | `Ejemplos_Devices/scripts/*.bat`, `vs.bat` |
| Regla ApiKeys | `.gitignore:421` |

**Cruces**: rosetta/x64 y `GetEnviromentVersion` → índice 10 (Docs/otros) · librerías QR (BSM/BSN/CS/ZN) → índice 02 (qr-nuget) · `Ejemplo_Maui_Hibrida` → índice 08 (web-híbrida).
