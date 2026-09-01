# 09 — CI/CD, build y simulación

> **Propósito:** los 18 workflows de GitHub Actions que compilan cada ejemplo para el simulador iOS, cómo instalan su propio Xcode y .NET, y la simulación con grabación de evidencia.
> **Fuente primaria:** `.github/workflows/` (18 `.yml` + 2 `.md`), `Utilities/` y `Ejemplos_Devices/Ejemplos_Devices.slnx`.
> **Índices relacionados:** [00_MASTER-INDEX](00_MASTER-INDEX.md) · [02_QR](02_QR.md) (el problema de MLKit/Rosetta que motiva la variante x64) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## 1. El árbol

```
.github/workflows/
├── cd-ios-camera.*.yml        4 workflows  → índice 01
├── cd-ios-qr.*.yml            9 workflows  → índice 02
├── cd-ios-printer.*.yml       1 workflow   → índice 03
├── cd-ios-gps.*.yml           1 workflow   → índice 04
├── cd-ios-phone.*.yml         2 workflows  → índice 06
├── cd-ios-Integrada.*.yml     1 workflow   → índice 08
├── Readme.md                  Versiones de workloads de .NET de la máquina de desarrollo
├── Pipelinea-Version.md       Bitácora de versiones del pipeline
└── Analisis/log_1.md, log_2.md
Utilities/
├── simular.sh                 332 l. — arranque del simulador + grabación + GIF
├── simular_ui.sh              233 l. — variante con recorrido Maestro y video MP4
├── download_sectigo.ps1
└── end2end/                   3 flujos Maestro («dedo virtual»)
```

## 2. Convención de nombre y categorías

`cd-ios-<categoria>.<Proyecto>.yml`, donde `<categoria>` coincide con la carpeta bajo `Ejemplos_Devices/`.

| Categoría | Workflows |
|-----------|-----------|
| `qr` | 9 (los 8 proyectos actuales + `Ejemplo_LectorQR_Dialog`) |
| `camera` | 4 (`MiMediaPicker_Callback`, `_Callback_Normalizacion`, `_Task`, `MiMediaSelfie_Callback_Normalizacion`) |
| `phone` | 2 (`Ejemplo_Dialer`, `Ejemplo_DirectCall`) |
| `gps` | 1 (`Ejemplo_GPS`) |
| `printer` | 1 (`Ejemplo_MotorDSL`) |
| `Integrada` | 1 (`Ejemplo_Maui_Hibrida`) — única categoría en mayúscula |

**Qué no tiene workflow:** `Ejemplo_Photo_MediaPicker`, `Ejemplo_Maui_Mapas`, `Ejemplo_Maui_Connectivity`, `Ejemplo_ThermalPrinter`, `Ejemplo_MotorDSL_Dialog`, `Ejemplo_ws_Blazor` y `Ejemplo_Maui_Hibrida.Tests`. **La suite de tests no se ejecuta en CI**, aunque su TFM `net10.0` plano fue elegido justamente para poder correrla ahí ([índice 08 §9](08_App-Hibrida-Integrada.md)).

⚠️ `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` apunta a un proyecto que **ya no existe** en el árbol (`com.ejemplos.devices.qr.dialog`): es el prototipo del que salieron los cuatro pares BSM/BSN/CS/ZN ([índice 02 §6](02_QR.md)).

## 3. Disparadores

Los 18 declaran `workflow_call`, así que **son invocables desde otro workflow**. El disparo por `push` está **comentado en 17 de 18**: solo `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` lo tiene activo, filtrado a `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/**` y excluyendo `**.md`, `.gitignore` y `.gitattributes`. En los demás, el bloque `push` sigue ahí comentado, con la ruta de su proyecto ya escrita.

No hay ningún workflow «paraguas» que invoque a los demás: la ejecución es manual o por llamada externa.

## 4. Anatomía del pipeline (32 pasos)

Todos corren en `runs-on: macos-15` y siguen la misma secuencia. Lo distintivo es que **no usan el Xcode ni el .NET del runner: instalan los suyos**.

| Bloque | Qué hace |
|--------|----------|
| Runner | Reporta arquitectura, CPU, modelo y cores de la VM |
| Checkout | `actions/checkout@v4` |
| **XCODE** | `pipx install gdown` → descarga **Xcode desde Google Drive** por ID de archivo → `xip --expand` → borra los Xcode del runner → `xcode-select --switch` → `-runFirstLaunch` + `-license accept` → `xcodebuild -downloadPlatform iOS` → arranca el simulador de prueba |
| **.NETCORE** | **Borra `/Users/runner/.dotnet`** → instala la versión exacta con `dotnet-install.sh` → verifica |
| **WORKLOADS** | `dotnet workload install ios maui maui-ios --version 10.0.100` |
| **PATH** | Resuelve `SOLUTION_PATH`, `PROJECT_PATH`, `PLIST_PATH` con `realpath` y los exporta a `$GITHUB_ENV` |
| **VERSIONADO** | Lee `CFBundleVersion` y `CFBundleShortVersionString` del `Info.plist` con `PlistBuddy` → `APP_VERSION_BUILD`; y `VERSION_FECHA` con `date +%Y%m%d%H%M` |
| **MANIFEST** | `plutil -lint` sobre el `Info.plist`; falla el job si es inválido |
| **BUILD** | `dotnet clean` → `restore` → `build` para `net10.0-ios` |
| **FIRMA** | Firma **ad-hoc** manual (ver §4.1) |
| **PUBLISH** | Zip del `.app` → `upload-artifact@v4`, `retention-days: 1` |
| **ROSETTA** | Solo si `RUNTIME_IDENTIFIER_SIMULATOR == 'iossimulator-x64'`: `softwareupdate --install-rosetta` |
| **RUN** | `brew install ffmpeg` → ejecuta el script de simulación → sube la evidencia |
| **Cleanup** | `if: always()` — borra keychain y provisioning profile |

Variables comunes: Xcode **26.0** (con el 26.3 comentado como alternativa), .NET **10.0.300** (con 10.0.102 comentado), workload **10.0.100**, simulador **iPhone 17 Pro Max**, `BUILD_CONFIG_SIMULATOR: Release`.

Bloque de identidad del proyecto, el único que cambia entre workflows:

```yaml
PACKAGE_NAME:   'com.ejemplos.devices.qr.barcodescanner_native_maui.simple'
SOLUTION_FOLDER:'Ejemplos_Devices'
PROJECTS_ROOT:  'QR'
PROJECT_NAME:   'BSN.LectorQR'
PROJECT_FILE:   'BSN.LectorQR.csproj'
```

### 4.1 El build y la firma

```bash
dotnet build "$PROJECTS_ROOT/$PROJECT_NAME/$PROJECT_FILE" \
    -c Release -f:net10.0-ios \
    -p:RuntimeIdentifier=iossimulator-arm64 \
    -p:LinkMode=SdkOnly -p:CLI_Build=true \
    -p:CodesignEntitlements=Platforms/iOS/Entitlements.Development.plist \
    -p:EnableCodeSigning=false -p:CodesignProvision="" -p:CodesignKey="-" -v normal
```

`LinkMode=SdkOnly` es la propiedad que `QR/Ejemplo_Docs_QR/cicd.md` identificó como parte de la solución al fallo de link de MLKit ([índice 02 §2](02_QR.md)).

La firma se hace **a mano y en tres pasos**, porque el build va sin firmar: limpiar (`xattr -cr`, borrar `_CodeSignature`, `chmod +x`) → firmar cada `*.dll`/`*.dylib`/`*.aotdata*` con `codesign --sign "-"` → firmar el binario principal **con los entitlements** → sellar el bundle → verificar con `codesign -vv -d`. Sin AOT firmado el simulador rechaza la app.

Artefacto: `{VERSION_FECHA}_{APP_VERSION_BUILD}_{PACKAGE_NAME}.app.zip`.

### 4.2 La variante `Integrada`

`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` agrega sobre el pipeline estándar:

| Diferencia | Detalle |
|------------|---------|
| `push` activo | Filtrado a la carpeta de la app híbrida |
| `SCRIPT_SIMULATOR` | `./Utilities/simular_ui.sh` en vez de `simular.sh` |
| `MAESTRO_VERSION: '1.41.0'` | Instala Maestro con `curl -Ls https://get.maestro.mobile.dev \| bash` y lo agrega al `PATH` |
| Boot del simulador **por GUI** | Se abre `Simulator.app` para que levante BackBoard/SpringBoard: **el boot headless por `simctl` se cuelga en «Waiting on BackBoard»**. Fire-and-forget, con timeout y reintento |
| Evidencia | `recorrido.mp4` (video del recorrido automatizado) en vez del GIF |
| Pre-warm | La app se pre-carga antes de grabar, para que el arranque en frío (Release + WebView remoto) no quede en el video |

### 4.3 La variante x64 / Rosetta

Dos workflows —`cd-ios-qr.BSM.LectorQR.yml` y `cd-ios-qr.BSM.LectorQR_Dialog.yml`, los de `BarcodeScanner.Mobile.Maui`— usan `RUNTIME_IDENTIFIER_SIMULATOR: iossimulator-x64` en vez de `arm64`. Es exactamente el caso que documenta el estudio de NuGets: **ML Kit no publica slice de simulador arm64**, así que hay que compilar x64 y ejecutar bajo Rosetta ([índice 02 §2](02_QR.md)).

Consecuencias en el pipeline, activadas por condición sobre esa variable:
- La descarga del runtime usa `-architectureVariant universal`.
- Se instala Rosetta 2 (`softwareupdate --install-rosetta --agree-to-license`) para que `simctl launch` pueda ejecutar el binario x86_64.

`Pipelinea-Version.md` registra la otra consecuencia: por eso los proyectos condicionan `AdamE.Google.iOS.GoogleUtilities` 8.1.0.3 a `'$(RuntimeIdentifier)' == 'iossimulator-x64'` — «porque la versión 9.0.1 no tiene esa librería para x64».

## 5. Versionado del pipeline

Los workflows de QR y el de Integrada llevan `PIPELINE_VERSION` con formato `AAAAMMDDhhmm_ejemplos`:

| Versión | Workflows |
|---------|-----------|
| `202606300809_ejemplos` | BSM.LectorQR, BSN.LectorQR, CS.LectorQR{,_Dialog}, ZN.LectorQR{,_Dialog}, Ejemplo_LectorQR_Dialog |
| `202606300822_ejemplos` | BSM.LectorQR_Dialog |
| `202606301040_ejemplos` | BSN.LectorQR_Dialog, **Integrada** |

Los 8 workflows de camera, gps, phone y printer **no declaran `PIPELINE_VERSION` ni `SCRIPT_SIMULATOR`**: son de la generación anterior, invocan `./Utilities/simular.sh` con la ruta escrita a mano en el paso de grabación. Es la deuda visible del dominio: la estandarización llegó hasta QR e Integrada.

`Pipelinea-Version.md` es la bitácora: `202606261239` («primera versión de yml estandarizada, incorpora mejoras en las rutas relativas») y `202606262101` (Rosetta + parametrización del script de simulación).

## 6. La simulación y la evidencia

| Script | Usa | Produce |
|--------|-----|---------|
| `Utilities/simular.sh` (332 l.) | 17 workflows | `evidencia_app.gif`, `frames/`, `debug_logs/`, `app_stream_full.txt` |
| `Utilities/simular_ui.sh` (233 l.) | Integrada | `recorrido.mp4` + logs |

`simular.sh` recibe `APP_PATH`, `PACKAGE_NAME` y `DEVICE_SIMULATOR` por entorno; resuelve el UUID del simulador con `xcrun simctl list devices`, lo arranca si no está *booted*, instala y lanza la app, graba y arma el GIF con `ffmpeg`. Trae un `run_with_timeout` propio basado en `perl -e "alarm"` porque **macOS no tiene el comando `timeout`**.

El paso de grabación es `continue-on-error: true` con `timeout-minutes: 30`: la evidencia es deseable, no bloqueante.

### 6.1 Pruebas end2end sobre la UI real

`Utilities/end2end/*.yaml` son flujos de **Maestro** («dedo virtual») que tocan controles **nativos por su texto visible** — sin `AutomationId` ni inspección del WebView, porque los botones son controles MAUI y su texto se expone como accessibility label en iOS. El `appId` se inyecta con `maestro test -e APP_ID=<bundle>`.

| Flujo | Para |
|-------|------|
| `com.ejemplos.devices.integrada.hibrida.yaml` (81 l.) | La app híbrida: `[Volver] [Geo Pos] [Llamar] [Leer QR]` |
| `com.ejemplos.devices.gps.yaml` (35 l.) | `Ejemplo_Maui_GPS` |
| `com.ejemplos.devices.qr.dialog.yaml` (35 l.) | El lector QR en modo diálogo |

El YAML de la híbrida documenta su propia procedencia y sus límites: los textos fueron **verificados contra dispositivo real** por `adb uiautomator dump` (Motorola Moto G42, Android 1080 px); «Geo Pos» corrige un «Geo Posicionar» anterior que **no matcheaba**; en Android a 1080 px la barra de 4 botones no entra a lo ancho y «Leer QR» queda recortado, pero el target del CI es el simulador iOS (iPhone 17 Pro Max), más ancho. Usa `stopApp: false` para no reiniciar la app y no grabar el arranque en frío.

También anota lo que es esperado y no un fallo: en el simulador la cámara del lector QR **se ve negra** (no hay cámara real) y el overlay de GPS puede pasar a capa de error por falta de ubicación — igual sirven como evidencia de navegación.

## 7. La solución y el arranque local

`Ejemplos_Devices/Ejemplos_Devices.slnx` es el formato **`.slnx`** (XML) de solución. Agrupa **23 proyectos** en carpetas por dominio y, además, **enlaza los `.md` de documentación y los `.yml` de workflows** como archivos de solución (carpetas `/Docs/`, `/github.workflow/`, `/.Utilities/`), de modo que la documentación se navega desde el IDE.

⚠️ **`Ejemplo_Maui_Hibrida.Tests` no está en la solución**: existen 24 `.csproj` en el árbol y el `.slnx` referencia 23. Sumado a que ningún workflow la ejecuta, la suite queda fuera tanto del IDE como del CI.

⚠️ Tres rutas del `.slnx` apuntan a archivos que **no existen** (verificado): `GPS/Docs_GPS/Readme.md` (la carpeta real es `GPS/Ejemplo_Docs_GPS/`), `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md` (mezcla Printer con Maps) y la carpeta `/Docs/web-hibrida/` está declarada vacía aunque `Docs/web-hibrida/` tiene 5 documentos.

`vs.bat` en la raíz contiene una sola línea: `code .`

`Ejemplos_Devices/scripts/` trae tres `.bat` de conveniencia para Windows: `Ejemplo_Maui_GPS_launch.bat`, `Ejemplo_Photo_MediaPicker_launch.bat` y `GetEnviromentVersion.bat` (documentado en `Docs/otros/GetEnviromentVersion.md`).

## 8. El entorno de referencia

`.github/workflows/Readme.md` deja registrado el estado de la máquina de desarrollo, que es lo que el YAML replica:

| Workload | Versión del manifiesto |
|----------|------------------------|
| `android` | 36.1.12/10.0.100 |
| `ios`, `maccatalyst` | 26.2.10191/10.0.100 |
| `maui-android`, `maui-windows` | 10.0.1/10.0.100 |
| `wasm-tools` | 10.0.102/10.0.100 |

`dotnet --version` y `dotnet workload --version`: **10.0.102** (el CI instala 10.0.300).

## 9. Observaciones

- **El pipeline instala su propio Xcode desde Google Drive** (`gdown` + ID de archivo público) en vez de usar el del runner o `xcode-select` sobre uno preinstalado. Es lo que permite fijar Xcode 26.0, pero ata el CI a la disponibilidad de ese archivo en Drive.
- Cada job **borra los Xcode del runner** (`sudo rm -rf /Applications/Xcode*.app`) y `/Users/runner/.dotnet` antes de instalar los suyos.
- Los pipelines **solo compilan y ejecutan para el simulador iOS**. No hay build de Android en CI, pese a que Android es la plataforma principal de casi todos los ejemplos (y la única en la que funciona la impresión Bluetooth).
- No hay firma con certificado real ni publicación a TestFlight: todo es firma ad-hoc y artefacto `.app.zip` con **1 día de retención**.
- `.github/workflows/Analisis/log_1.md` y `log_2.md` guardan logs de análisis del pipeline.

## 10. Fuentes

| Ruta | Contenido |
|------|-----------|
| `.github/workflows/cd-ios-qr.BSN.LectorQR.yml` | Workflow de referencia de la generación estandarizada (471 líneas) |
| `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` | La variante con Maestro, boot por GUI y video |
| `.github/workflows/Pipelinea-Version.md` | Bitácora de versiones del pipeline y el porqué de Rosetta |
| `.github/workflows/Readme.md` | Versiones de workloads de la máquina de desarrollo |
| `Utilities/simular.sh`, `simular_ui.sh` | Arranque del simulador, instalación, lanzamiento y grabación |
| `Utilities/end2end/*.yaml` | Los tres flujos Maestro y sus notas de verificación |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | Estructura de la solución y enlaces a documentación |
