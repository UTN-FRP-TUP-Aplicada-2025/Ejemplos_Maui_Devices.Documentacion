# 09 — CI/CD y Build

> **Propósito**: entender cómo se compila, firma, simula y publica cada ejemplo en GitHub Actions (iOS sobre `macos-15`), qué ejemplos quedan sin pipeline y por qué, cómo se arma la solución `.slnx`, qué TFMs y versiones mínimas declaran los `.csproj`, y qué scripts locales (Windows / PowerShell / bash) acompañan al build.
> **Fuente primaria**: `.github/workflows/**` (18 `.yml` + `Readme.md`, `Pipelinea-Version.md`, `Analisis/log_1.md`, `Analisis/log_2.md`), `Utilities/**`, `Ejemplos_Devices/scripts/*.bat`, `vs.bat`, `Ejemplos_Devices/Ejemplos_Devices.slnx`, `.gitignore`, y los `*.csproj` en lo que hace a TFMs/RIDs. Commit indexado: `285f6fb`.
> **Índices relacionados**: [00_MASTER-INDEX](00_MASTER-INDEX.md) · [02_QR](02_QR.md) (librerías QR y la limitación de ML Kit) · [04_GPS](04_GPS.md) (API key de Google) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (la app que recorre Maestro y su proyecto de tests) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md) (mapa de `Docs/`, incluida `Docs/otros/propuesta-rosetta.md`).

Todas las rutas son relativas a la raíz del repo de código `Ejemplos_Maui_Devices`.

---

## 1. Panorama del dominio

```
.github/workflows/
├── cd-ios-<categoría>.<Proyecto>.yml   ← 18 workflows, un job "publish-ios" cada uno
├── Readme.md                           ← volcado de `dotnet workload list` local (Windows)
├── Pipelinea-Version.md                ← changelog de versiones del pipeline (PIPELINE_VERSION)
└── Analisis/log_1.md, log_2.md         ← logs pegados de corridas reales (build x64 fallido; simulación colgada)
Utilities/
├── simular.sh                          ← simulación pasiva: screenshots → GIF (16 workflows)
├── simular_ui.sh                       ← simulación activa: Maestro + video MP4 (sólo la híbrida)
├── end2end/<bundle-id>.yaml            ← flujos Maestro (se eligen por PACKAGE_NAME)
└── download_sectigo.ps1                ← baja la cadena CA de aplicada.somee.com a res/raw (Android)
Ejemplos_Devices/
├── Ejemplos_Devices.slnx               ← solución (formato XML nuevo)
└── scripts/*.bat                       ← build+deploy Android local con adb (Windows)
vs.bat                                  ← `code .`
.gitignore                              ← plantilla VisualStudio + `**/Services/ApiKeys.cs`
```

Idea central: **todos los pipelines son de iOS, construyen para el simulador (no para dispositivo), firman ad-hoc, suben un `.app.zip` y graban evidencia en el simulador**. No hay pipeline de Android, ni de tests, ni de publicación a tiendas.

---

## 2. Catálogo de los 18 workflows

Todos: `runs-on: macos-15`, job `publish-ios`, `DEVICE_SIMULATOR: 'iPhone 17 Pro Max'`, `BUILD_CONFIG_SIMULATOR: 'Release'` (salvo ⚠️), Xcode 26.0, SDK .NET 10.0.300, workloads 10.0.100.

| Workflow (`.github/workflows/`) | Proyecto (`Ejemplos_Devices/…`) | Disparador efectivo | RID simulador | Simulación | `upload-artifact` | `PIPELINE_VERSION` |
|---|---|---|---|---|---|---|
| `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback.yml` | `Camera/Ejemplo_Photo_MiMediaPicker_Callback` | `workflow_call` | arm64 | `simular.sh` | v4 | — |
| `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.yml` | `Camera/…_Callback_Normalizacion` | `workflow_call` | arm64 | `simular.sh` | v4 | — |
| `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Task.yml` | `Camera/Ejemplo_Photo_MiMediaPicker_Task` | `workflow_call` | arm64 | `simular.sh` | v4 | — |
| `cd-ios-camera.Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion.yml` | `Camera/Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | `workflow_call` | arm64 | `simular.sh` | v4 | — |
| `cd-ios-gps.Ejemplo_GPS.yml` | `GPS/Ejemplo_Maui_GPS` | `workflow_call` | arm64 | `simular.sh` | v4 | — |
| `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` | `Integrada/Ejemplo_Maui_Hibrida` | **`push` a `main`** + `workflow_call` | arm64 | **`simular_ui.sh` + Maestro** | v5 | `202606301040_ejemplos` |
| `cd-ios-phone.Ejemplo_Dialer.yml` | `Phone/Ejemplo_Maui_Dialer` | `workflow_call` | arm64 | `simular.sh` | v4 | — |
| `cd-ios-phone.Ejemplo_DirectCall.yml` | `Phone/Ejemplo_Maui_DirectCall` | `workflow_call` | arm64 | `simular.sh` | v4 | — |
| `cd-ios-printer.Ejemplo_MotorDSL.yml` | `Printer/Ejemplo_MotorDSL` | `workflow_call` | arm64 | `simular.sh` | v4 | — |
| `cd-ios-qr.BSM.LectorQR.yml` | `QR/BSM.LectorQR` | `workflow_call` | **x64** | `simular.sh` | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.BSM.LectorQR_Dialog.yml` | `QR/BSM.LectorQR_Dialog` | `workflow_call` | **x64** | `simular.sh` | v5 | `202606300822_ejemplos` |
| `cd-ios-qr.BSN.LectorQR.yml` | `QR/BSN.LectorQR` | `workflow_call` | arm64 | `simular.sh` | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.BSN.LectorQR_Dialog.yml` | `QR/BSN.LectorQR_Dialog` | `workflow_call` | arm64 | `simular.sh` | v5 | `202606301040_ejemplos` |
| `cd-ios-qr.CS.LectorQR.yml` | `QR/CS.LectorQR` | `workflow_call` | arm64 | `simular.sh` | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.CS.LectorQR_Dialog.yml` | `QR/CS.LectorQR_Dialog` | `workflow_call` | arm64 | `simular.sh` | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.ZN.LectorQR.yml` | `QR/ZN.LectorQR` | `workflow_call` | arm64 | `simular.sh` | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.ZN.LectorQR_Dialog.yml` | `QR/ZN.LectorQR_Dialog` | `workflow_call` | arm64 | `simular.sh` | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` | ⚠️ `QR/Ejemplo_LectorQR_Dialog` **no existe** | `workflow_call` | arm64 | ⚠️ `./Utils/simular.sh` (no existe) | v4 | `202606300809_ejemplos` |

`<categoría>` coincide con la carpeta bajo `Ejemplos_Devices/`, en minúscula salvo `Integrada`.

### 2.1 Cómo leer los disparadores

- En 17 de los 18 el bloque `push:` está **comentado** y sólo queda `workflow_call:` (p. ej. `cd-ios-gps.Ejemplo_GPS.yml:5-15`). ⚠️ No hay en el árbol ningún workflow que los invoque (`uses: ./.github/workflows/…`), así que **en la práctica no corren nunca** salvo que se descomente el `push:` o se agregue un llamador.
- Sólo `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:5-13` tiene `push` activo sobre `main`, filtrado a `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/**` y excluyendo `**.md`, `**/*.gitignore`, `**/*.gitattributes`. Historia: en el commit `a994257` se renombró desde `cd-ios-gps.Ejemplo_Maui_Hibrida.yml` (que tenía el `push` activo) y en el mismo commit se comentó el `push`; se reactivó en `87cb528` (CHANGELOG `[2026-07-18] — Flujo end2end…`, «Activado»).
- ⚠️ El filtro de paths **no** incluye `Integrada/Ejemplo_Maui_Hibrida.Tests/**` ni `Integrada/Ejemplo_ws_Blazor/**`: tocar sólo los tests o el backend Blazor no dispara el pipeline.

### 2.2 Dos «generaciones» de plantilla

| Rasgo | Gen. 1 — Camera / GPS / Phone / Printer | Gen. 2 — QR (8) + Integrada |
|---|---|---|
| `PIPELINE_VERSION` en `env` | no | sí (`cd-ios-qr.CS.LectorQR.yml:18`) |
| `SCRIPT_SIMULATOR` en `env` | no: ruta fija `./Utilities/simular.sh` (`cd-ios-gps.Ejemplo_GPS.yml:418-419`) | sí (`cd-ios-qr.CS.LectorQR.yml:48`) |
| Descarga de runtime según RID | siempre `-downloadPlatform iOS` (`cd-ios-gps.Ejemplo_GPS.yml:137-140`) | `universal` si RID es x64 (`cd-ios-qr.CS.LectorQR.yml:147-155`) |
| Paso Rosetta condicional | no | sí (`cd-ios-qr.CS.LectorQR.yml:424-427`) |
| `BASE_PATH` con `realpath` | no (`cd-ios-gps.Ejemplo_GPS.yml:324`) | sí (`cd-ios-qr.CS.LectorQR.yml:344-345`) |
| Paso final de limpieza | sólo un `echo` (`cd-ios-gps.Ejemplo_GPS.yml:435-438`) | borra `app-signing.keychain-db` y `.mobileprovision` si existen (`cd-ios-qr.CS.LectorQR.yml:465-471`) |
| Copia `/tmp/app_stderr.txt` a `debug_logs/` | no (lo sube directo desde `/tmp`, `cd-ios-gps.Ejemplo_GPS.yml:431`) | sí (`cd-ios-qr.CS.LectorQR.yml:450-451`) |

Las plantillas son casi idénticas dentro de cada generación: `diff` entre dos workflows QR sólo muestra nombre, `PACKAGE_NAME`, `PROJECT_NAME/FILE`, RID y versión de `upload-artifact`. La duplicación es deliberada (un workflow autocontenido por ejemplo).

⚠️ El huérfano `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` es una plantilla intermedia: tiene `PIPELINE_VERSION` (`:18`), `BASE_PATH` con `realpath` y la limpieza de keychain de Gen. 2, pero no `SCRIPT_SIMULATOR`, ni `-architectureVariant universal`, ni el paso Rosetta, ni la copia de `app_stderr.txt`.

---

## 3. Proyectos sin pipeline

De los 24 proyectos reales (`git ls-files '*.csproj'` da 25; el 25.º es `Printer/Ejemplo_Docs_Printer/Prompts_Generacion_Apk_ejemplo/ficheros_claude.ia/Thermalprintermaui.csproj`, material de documentación, no un proyecto de la solución) hay **17 con workflow** y **7 sin**:

| Proyecto | Tipo | Observación de build relevante |
|---|---|---|
| `Camera/Ejemplo_Photo_MediaPicker` | MAUI | Tiene `Entitlements.Development.plist`; sólo le falta el `.yml`. Tiene `.bat` local (§11). |
| `Maps/Ejemplo_Maui_Mapas` | MAUI (+ Windows TFM) | ⚠️ Sin `Platforms/iOS/Entitlements.Development.plist`: la firma del pipeline (§6) lo exige. |
| `Printer/Ejemplo_MotorDSL_Dialog` | MAUI | Tiene entitlements; sólo falta el `.yml`. |
| `Printer/Ejemplo_ThermalPrinter` | MAUI | ⚠️ Sin entitlements de desarrollo. |
| `Red/Ejemplo_Maui_Connectivity` | MAUI | ⚠️ Sin entitlements de desarrollo. |
| `Integrada/Ejemplo_ws_Blazor` | Web (`Microsoft.NET.Sdk.Web`, `net10.0`) | No es app móvil; no aplica el pipeline iOS. |
| `Integrada/Ejemplo_Maui_Hibrida.Tests` | xUnit `net10.0` | ⚠️ Ningún workflow ejecuta `dotnet test` (ver §12 y §16). |

La razón de que falten los `.yml` de los cinco MAUI no está escrita en ninguna fuente → **no verificado**. Lo verificable: tres de ellos tampoco tienen el `Entitlements.Development.plist` que el paso de firma referencia.

---

## 4. Anatomía de un workflow (paso a paso)

Referencia: `cd-ios-gps.Ejemplo_GPS.yml` (Gen. 1). Las diferencias de Gen. 2 están en §2.2.

```
checkout ─► XCODE (gdown → xip → /Applications/Xcode_26.0.app → xcode-select → license)
        ─► runtime simulador (downloadPlatform) ─► simctl boot
        ─► .NET (borra ~/.dotnet → dotnet-install.sh 10.0.300) ─► workloads ios maui maui-ios 10.0.100
        ─► rutas (SOLUTION/PROJECT/PLIST_PATH) ─► versión (PlistBuddy + fecha) ─► plutil -lint
        ─► clean ─► [GPS: ApiKeys.cs desde secreto] ─► restore ─► build simulador (sin firma)
        ─► firma ad-hoc manual ─► verificación entitlements ─► zip .app ─► artefacto 1
        ─► [x64: Rosetta] ─► ffmpeg ─► simular.sh (continue-on-error, 30 min) ─► artefacto 2
```

| Fase | Líneas (`cd-ios-gps.Ejemplo_GPS.yml`) | Qué hace |
|---|---|---|
| Info del runner | 55-60 | `uname -m`, CPU, modelo, cores |
| Checkout | 62-65 | `actions/checkout@v4` |
| Xcode: `gdown` | 84-89 | `pipx install gdown` |
| Xcode: descarga | 91-97 | `gdown <XCODE_GOOGLE_FILE_INSTALLER_ID> -O <XCODE_FILE_INSTALLER>` |
| Xcode: expandir | 99-106 | `xip --expand`, borra el `.xip` |
| Xcode: instalar | 108-122 | **`sudo rm -rf /Applications/Xcode*.app`**, mueve a `Xcode_26.0.app`, symlink `Xcode.app`, `xcode-select --switch`, `-runFirstLaunch`, `-license accept` |
| Runtime simulador | 137-140 | `sudo xcodebuild -downloadPlatform iOS` |
| Boot de prueba | 142-151 | `xcrun simctl boot "iPhone 17 Pro Max"` |
| .NET | 155-182 | borra `/Users/runner/.dotnet`, instala con `dotnet-install.sh --version 10.0.300`, agrega a `GITHUB_PATH` |
| Workloads | 198-210 | `dotnet workload install ios maui maui-ios --version 10.0.100` (comentario con manifiestos `ios 26.0.11017`, `maui 10.0.0`) |
| Rutas | 218-238 | exporta `PROJECT_PATH`, `SOLUTION_PATH`, `PLIST_PATH` a `GITHUB_ENV` |
| Versionado | 242-259 | `APP_VERSION_BUILD = <CFBundleShortVersionString>_<CFBundleVersion>`; `VERSION_FECHA = date +%Y%m%d%H%M` |
| Manifest | 263-267 | `plutil -lint Info.plist` (aborta si es inválido) |
| Clean | 271-277 | `dotnet clean … -f net10.0-ios` |
| API key (sólo GPS) | 279-285 | ver §10 |
| Restore | 287-293 | `-p:TargetFramework=net10.0-ios -p:RuntimeIdentifier=<RID>` |
| Build | 295-333 | ver §6 |
| Firma | 337-371 | ver §6 |
| Test entitlements | 373-380 | `file`, `PlistBuddy Print CFBundleIdentifier`, `codesign -d --entitlements :-` |
| Zip + artefacto | 384-403 | ver §6 |
| ffmpeg | 407-409 | `brew install ffmpeg` (para el GIF) |
| Simulación | 411-419 | `continue-on-error: true`, `timeout-minutes: 30`, corre `./Utilities/simular.sh` |
| Evidencia | 421-433 | artefacto `grabacion-simulador` |
| Limpieza | 435-438 | `if: always()`, sólo `echo` |

⚠️ Etiquetas invertidas en el versionado (`cd-ios-gps.Ejemplo_GPS.yml:246-250`): imprime «Versión visible para los usuarios» junto a `CFBundleVersion` y «Versionado de compilacion» junto a `CFBundleShortVersionString`; es al revés. El valor compuesto es correcto. Con los `Info.plist` actuales (`1.0`/`1.0`, p. ej. `GPS/Ejemplo_Maui_GPS/Platforms/iOS/Info.plist:47-51`) da `1.0_1.0` (confirmado en `Analisis/log_2.md:32`).

---

## 5. Toolchain fijado: Xcode propio, .NET y workloads

| Variable | Valor vigente | Dónde | Comentado / alternativa |
|---|---|---|---|
| `XCODE_VERSION`, `_SHORT` | `26.0` | `cd-ios-gps.Ejemplo_GPS.yml:23-24` | `26.3` comentado (`:18-21`) |
| `XCODE_FILE_INSTALLER` | `Xcode_26_Universal.xip` | `:25` | Selfie usa `Xcode_26_Universal_1.xip` (`cd-ios-camera.Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion.yml:27`) |
| `XCODE_GOOGLE_FILE_INSTALLER_ID` | `1GoDmCUBOMKM5nnXCXxxnAgoKpSwZvaMP` | `:26` | Selfie: `1d6zlneZFR5_R87RRH6hI5vPYup_nc44T` (`…Selfie….yml:28`) |
| `DOTNET_VERSION` | `10.0.300` | `:29` | `10.0.102` comentado (`:28`) |
| `DOTNET_TARGET_VERSION` | `net10.0` | `:30` | — |
| `DOTNET_VERSION_WORKLOAD` | `10.0.100` | `:31` | — |
| `DOTNET_WORKLOAD_INSTALL_ADDITIONAL_ARGS` | `--ios-simulator-runtime=26.0` | `:46` | — |
| `MD_APPLE_SDK_ROOT` | `/Applications/Xcode_26.0.app` | `:47` y otra vez en el step de build (`:297-298`) | — |

**Por qué el CI instala su propio Xcode.** Ninguna fuente del repo lo dice explícitamente → el motivo es **no verificado**. Lo que el código muestra:

1. El pipeline **borra todo Xcode preinstalado** del runner (`:113`) y fija `xcode-select` a una única versión (`:117`), así que el build no depende de qué imagen trae `macos-15` ese día.
2. Esa versión tiene que casar con el workload iOS que se instala: el comentario de `:202-207` registra el manifiesto `ios 26.0.11017/10.0.100`, y el SDK de iOS que aparece en los logs es `Microsoft.iOS.Sdk.net10.0_26.0/26.0.11017` (`Analisis/log_1.md:2063`).
3. El instalador viene de **Google Drive por id** con `gdown`: dependencia externa (permisos/cuota del archivo) fuera del control de GitHub.

⚠️ `cd-ios-phone.Ejemplo_Dialer.yml:25` nombra el archivo `Xcode_26.3_Universal.xip` pero usa el mismo id de Drive que el resto (Xcode 26.0) y `XCODE_VERSION: '26.0'`: el nombre local del `.xip` es cosmético, lo que manda es el id.

⚠️ Versiones distintas en local y en CI: en la máquina Windows del autor, `.github/workflows/Readme.md` registra `dotnet workload --version` `10.0.102` (`:6-7`), `dotnet --version` `10.0.102` (`:52-53`) y los manifiestos `android 36.1.12`, `ios`/`maccatalyst 26.2.10191`, `maui-android`/`maui-windows 10.0.1` (`:26-30`); el CI usa SDK `10.0.300` + workloads `10.0.100`. Mismo TFM (`net10.0-*`).

---

## 6. Build de simulador, firma ad-hoc y artefactos

### 6.1 Build (`cd-ios-gps.Ejemplo_GPS.yml:307-317`)

```bash
dotnet build "<ROOT>/<PROYECTO>/<PROYECTO>.csproj" -c Release -f:net10.0-ios \
  -p:RuntimeIdentifier=<iossimulator-arm64|iossimulator-x64> -p:LinkMode=SdkOnly -p:CLI_Build=true \
  -p:CodesignEntitlements=Platforms/iOS/Entitlements.Development.plist \
  -p:EnableCodeSigning=false -p:CodesignProvision="" -p:CodesignKey="-" -v normal
```

Salida: `<PROJECT_PATH>/bin/Release/net10.0-ios/<RID>/<PROJECT_NAME>.app` (`:324-327`).

### 6.2 Firma «robusta» manual (`:337-371`)

No hay certificados ni perfiles de aprovisionamiento: todo es **ad-hoc** (`--sign "-"`), suficiente para el simulador.

1. `xattr -cr` sobre el `.app` y borrado de `_CodeSignature` (`:353-354`).
2. Firma uno por uno de `*.dll`, `*.dylib`, `*.aotdata*` (`:359`).
3. Firma del binario principal **con** `Platforms/iOS/Entitlements.Development.plist` (`:363`).
4. Sellado del bundle (`:366`) y `codesign -vv -d` (`:369`). En una corrida real el resultado fue `Signature=adhoc`, `TeamIdentifier=not set` (`Analisis/log_2.md:50-52`).

El `Entitlements.Development.plist` es idéntico en los 19 proyectos que lo tienen (mismo hash): `aps-environment=development` + `get-task-allow=true` (`GPS/Ejemplo_Maui_GPS/Platforms/iOS/Entitlements.Development.plist:5-9`).

⚠️ El paso «Clean up keychain and provisioning profile» sugiere una firma con keychain/perfil que **no existe** en ningún workflow: en Gen. 2 borra archivos que nunca se crean; en Gen. 1 es un `echo`.

### 6.3 Artefactos

| Artefacto | Nombre | Contenido | Retención |
|---|---|---|---|
| App de simulador | `<VERSION_FECHA>_<APP_VERSION_BUILD>_<PACKAGE_NAME>.app.zip` (`:388`) | `zip -ry` del `.app` | 1 día (`:403`) |
| Evidencia (Gen. 1 / QR) | `grabacion-simulador` | `evidencia_app.gif`, `frames/`, `debug_logs/`, `app_stream_full.txt` (+ `/tmp/app_stderr.txt` en Gen. 1) | 1 día, `if-no-files-found: warn` |
| Evidencia (Integrada) | `grabacion-simulador` | `recorrido.mp4`, `debug_logs/`, `app_stream_full.txt` (`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:457-466`) | 1 día |

⚠️ Un job verde **no** prueba que la app haya corrido: el paso de simulación tiene `continue-on-error: true`. `Analisis/log_2.md:102` documenta una corrida cortada a los 30 min en ese paso.

---

## 7. RID `iossimulator-x64` + Rosetta (QR con ML Kit)

Contenido del problema y de las librerías QR: ver [02_QR](02_QR.md). Aquí sólo la mecánica de build.

- **Síntoma**: con RID arm64, `BarcodeScanner.Mobile.Maui` enlaza `MLKitCommon` sólo con slice de dispositivo → `ld: building for 'iOS-simulator', but linking in object file … built for 'iOS'` (`Ejemplos_Devices/QR/Ejemplo_Docs_QR/cicd.md:125-130`).
- **Salida elegida**: compilar el simulador en x64 y ejecutarlo bajo Rosetta. Propuesta y riesgos en `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md` (§2-§6), que pide tres cambios: RID x64, `-downloadPlatform iOS -architectureVariant universal`, `softwareupdate --install-rosetta`.
- **Segundo obstáculo con x64**: faltaban símbolos `GUL*` (`_GULOSLogBasic`…) → `ld: symbol(s) not found for architecture x86_64` (`Analisis/log_1.md:2051-2058`, `Build FAILED` en `:2074`). Se resolvió agregando `AdamE.Google.iOS.GoogleUtilities 8.1.0.3` **sólo para `iossimulator-x64`** (`.github/workflows/Pipelinea-Version.md:2-9`), p. ej. `QR/BSM.LectorQR_Dialog/BSM.LectorQR_Dialog.csproj:78-80`.

| Pieza | Dónde está aplicada |
|---|---|
| `RUNTIME_IDENTIFIER_SIMULATOR: iossimulator-x64` | sólo `cd-ios-qr.BSM.LectorQR.yml:45` y `cd-ios-qr.BSM.LectorQR_Dialog.yml:45` |
| `-architectureVariant universal` condicional al RID | los 8 QR vigentes + Integrada (`cd-ios-qr.CS.LectorQR.yml:149-154`) |
| Paso Rosetta con `if: env.RUNTIME_IDENTIFIER_SIMULATOR == 'iossimulator-x64'` | los 8 QR vigentes + Integrada (`cd-ios-qr.CS.LectorQR.yml:424-427`) |
| `GoogleUtilities` condicional x64 en el `.csproj` | `BSM.LectorQR_Dialog`, `CS.LectorQR_Dialog`, `ZN.LectorQR_Dialog`, `Ejemplo_Maui_Hibrida` (`Integrada/Ejemplo_Maui_Hibrida/Ejemplo_Maui_Hibrida.csproj:94-96`) |

⚠️ `BSM.LectorQR` (simple) usa RID x64 pero su `.csproj` **no** tiene el `ItemGroup` de `GoogleUtilities` (sólo `BarcodeScanner.Mobile.Maui 9.0.1`, `QR/BSM.LectorQR/BSM.LectorQR.csproj:43`): reproduce el escenario del `log_1`. Resultado en corrida: **no verificado**.
⚠️ En los proyectos con RID arm64 el `ItemGroup` x64 queda inerte en CI.
⚠️ `cd-ios-qr.CS.LectorQR.yml:150` imprime «Descargando runtime para simulador arm64» justo en la rama x64/universal.

---

## 8. Simulación: `simular.sh` vs `simular_ui.sh`

Ambos leen del entorno del job `APP_PATH`, `PACKAGE_NAME`, `DEVICE_SIMULATOR`, `PROJECT_NAME` (y `simular.sh` también `PROJECTS_ROOT`).

| Aspecto | `Utilities/simular.sh` (332 líneas) | `Utilities/simular_ui.sh` (233 líneas) |
|---|---|---|
| Usado por | 16 workflows (Gen. 1 + QR) | sólo `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:48` |
| Interacción | ninguna (pasivo) | Maestro «dedo virtual» (`:174-185`) |
| Boot | `simctl boot` + espera 12×10 s (`:43-65`) | short-circuit si ya `Booted`; si no, abre `Simulator.app` + `bootstatus -b` 240 s, reintento con `erase` + 300 s (`:62-96`) |
| Lanzamiento | `simctl launch --stderr=/tmp/app_stderr.txt` (`:184`) | pre-warm `simctl launch` + `sleep 45` antes de grabar (`:153-157`) |
| Captura | 15 screenshots cada 0,5 s (`:199-217`) → GIF con ffmpeg (`:246-249`) | `simctl io recordVideo --codec h264` (`:167`), cierre con SIGINT (`:190`) |
| Salida | `evidencia_app.gif`, `frames/`, `debug_logs/` | `recorrido.mp4`, `debug_logs/` |
| Verificación «app fantasma» | `simctl listapps | grep PACKAGE_NAME` → `exit 1` (`:161-166`) | ídem (`:131-136`) |
| Timeout | `run_with_timeout` propio con `perl -e "alarm"`, alternativa al `timeout` que macOS no trae (`:27-32`) | ídem (`:33-37`) |
| Código de salida | `set -e` (`:2`): cualquier fallo corta | `set -e` (`:20`) y `exit 1` si no hay UUID, no bootea, no instala o la app no queda registrada; el fallo de **Maestro** no se propaga: termina en `exit 0` (`:231-233`) |

Pre-calentado en el workflow de la híbrida: abre `Simulator.app` y bootea en segundo plano durante el build (`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:164-171`), porque el boot headless se cuelga en «Waiting on BackBoard» (CHANGELOG `[2026-07-18] — Robustez del arranque…`).

Trampas de `simular.sh`:
- ⚠️ `:19` dice «No se encontró el simulador iPhone 16 Pro» aunque busca `DEVICE_SIMULATOR` (iPhone 17 Pro Max).
- ⚠️ `LAUNCH_OUTPUT` nunca se asigna (la asignación de `:183` está comentada), así que `:186` escribe vacío en `debug_logs/launch_output.txt` y `APP_PID` (`:191`) sale vacío.
- ⚠️ `:271` usa el predicado `'process == "${PROJECTS_ROOT}/${PROJECT_NAME}"'` entre comillas simples: las variables **no** se expanden. `simular_ui.sh:213` lo corrige.
- ⚠️ `simctl privacy … grant notifications` falla en el runner y el fallback con AppleScript tampoco funciona (`Analisis/log_2.md:84-92`); el script sigue igual.
- El archivo es UTF-8, pero los acentos ya están guardados como el carácter de reemplazo `�` (U+FFFD, 22 apariciones; p. ej. `:4`, `:19`) y los emojis como `?`: el texto quedó corrompido en el propio archivo, no es un problema de lectura. Por eso los logs muestran `aplicaci�n` (`Analisis/log_2.md:55`).

---

## 9. Flujos Maestro (`Utilities/end2end/`)

`simular_ui.sh:28` resuelve el flujo como `end2end/${PACKAGE_NAME}.yaml` y lo corre con `maestro --device <UUID> test -e APP_ID=<bundle>` (`:177`). Maestro se instala con `curl … get.maestro.mobile.dev | bash` exportando `MAESTRO_VERSION=1.41.0` (`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:49,438-443`); que el instalador respete esa variable es **no verificado**.

| Flujo | ¿Lo usa algún workflow? | Recorrido |
|---|---|---|
| `com.ejemplos.devices.integrada.hibrida.yaml` (81 líneas) | **Sí** (la híbrida) | `launchApp {stopApp:false}` → `extendedWaitUntil "Geo Pos" 120 s` → `Geo Pos` → `Llamar` → `Leer QR` → tap por coordenada `50%,92%` (botón «Volver» sólo-ícono de `QRLectorPage`) → `Volver` (`:41-81`) |
| `com.ejemplos.devices.gps.yaml` (35 líneas) | No | `Geo Posicionar` → `Llamar` → `Leer QR` → `back` |
| `com.ejemplos.devices.qr.dialog.yaml` (35 líneas) | No | idéntico al anterior |

- Los textos del flujo vigente coinciden con la botonera nativa de la híbrida: `Volver`, `Geo Pos`, `Llamar`, `Leer QR` (`Integrada/Ejemplo_Maui_Hibrida/Pages/MainPage.xaml:53-56`). Detalle de la app: [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).
- ⚠️ `gps.yaml` **no** describe `Ejemplo_Maui_GPS`: es el viejo `Utilities/flows/recorrido.yaml` de la híbrida renombrado sin cambios (`git show -M a994257`: `R100`), con el texto obsoleto «Geo Posicionar». `qr.dialog.yaml` es idéntico byte a byte a `gps.yaml` (ambos usan `appId: ${APP_ID}`); sólo su nombre lleva el bundle id del proyecto QR que ya no existe (§16).
- ⚠️ Los workflows GPS y QR nunca invocan `simular_ui.sh`, así que esos dos flujos están huérfanos.
- Notas del propio flujo vigente: en el simulador la cámara de `QRLectorPage` se ve negra y se acepta como evidencia de navegación (`:31-32`, `:61-62`); en Android a 1080 px «Leer QR» queda recortado, y queda comentado un fallback por coordenada `90%, 88%` (`:10-12`, `:63-68`). Los textos se verificaron con `adb uiautomator dump` en un Moto G42 (`:5-9`).

---

## 10. Secretos y API keys en el build

| Elemento | Dónde | Detalle |
|---|---|---|
| Secreto `GOOGLE_MAPS_API_KEY` | `cd-ios-gps.Ejemplo_GPS.yml:279-285` | `sed` reemplaza `REEMPLAZAR_CON_TU_API_KEY` en `Services/ApiKeys.cs.template` y escribe `Services/ApiKeys.cs` antes del restore |
| Plantilla | `GPS/Ejemplo_Maui_GPS/Services/ApiKeys.cs.template:11-14` | `internal static class ApiKeys { public const string GoogleMaps = "REEMPLAZAR_CON_TU_API_KEY"; }` |
| Exclusión de git | `.gitignore:420-421` | `**/Services/ApiKeys.cs` |
| Paso en el resto de Gen. 1 | los 4 de Camera, los 2 de Phone y `cd-ios-printer.Ejemplo_MotorDSL.yml`; p. ej. `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Task.yml:279-285` | el mismo paso, **comentado** (herencia de plantilla). Los 9 QR y la Integrada no lo tienen, ni comentado |

Es el único secreto que usan los workflows. Uso de la key en la app y geocodificación: ver [04_GPS](04_GPS.md). ⚠️ En local, sin copiar la plantilla a `ApiKeys.cs`, `Ejemplo_Maui_GPS` no compila (`GPS/Ejemplo_Maui_GPS/Services/GoogleMapService.cs:11` referencia `ApiKeys.GoogleMaps`).

---

## 11. Scripts locales (Windows / PowerShell)

| Script | Qué hace | Notas |
|---|---|---|
| `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat` | Agrega `%ProgramFiles(x86)%\Android\android-sdk\platform-tools` al PATH (`:11-19`), comprueba `adb devices` (`:30-35`) y corre `dotnet build <csproj> -f net10.0-android -t:Run` (`:37`) | Ruta del `.csproj` relativa al script (`:5-6`) |
| `Ejemplos_Devices/scripts/Ejemplo_Photo_MediaPicker_launch.bat` | Idéntico, apuntando a `Camera/Ejemplo_Photo_MediaPicker` (`:6`) | ⚠️ El banner dice «Ejemplo_Maui_GPS» (`:23`), copia del otro `.bat` |
| `Ejemplos_Devices/scripts/GetEnviromentVersion.bat` | `dotnet --version` + `dotnet workload list --verbosity detailed` | Explicación en `Docs/otros/GetEnviromentVersion.md` (ver [10](10_Documentacion-Transversal.md)) |
| `vs.bat` | `code .` (sin salto de línea final) | ⚠️ Pese al nombre, abre **VS Code**, no Visual Studio |
| `Utilities/download_sectigo.ps1` | Conecta por TLS a `aplicada.somee.com:443` (parámetros por defecto `:29-32`; conexión `:77-92`), arma la cadena (`:105-116`), exporta **sólo CA** (salta la hoja, `:157-163`) como `.pem` con nombre válido para recurso Android (`:40-56`) en `Integrada/Ejemplo_Maui_Hibrida/Platforms/Android/Resources/raw` (`:61-62`) e imprime el bloque `<certificates src="@raw/…"/>` para `network_security_config.xml` (`:189-203`) | ⚠️ El nombre dice «sectigo» pero la CA real es Let's Encrypt / ISRG Root X2; el propio encabezado lo aclara «por historia del repo» (`:3-5`). Los `.pem` versionados son `isrg_root_x2.pem`, `root_ye.pem`, `ye2.pem`. Compatible con PowerShell 5.1 (`:27`) |

No hay `.bat` para los otros 22 proyectos. El tema TLS/certificados en profundidad está en `Docs/Certificados-SSL/` (ver [10](10_Documentacion-Transversal.md)).

---

## 12. La solución `Ejemplos_Devices/Ejemplos_Devices.slnx`

Formato XML nuevo (`<Solution>` con `<Folder>`/`<Project>`/`<File>`). Organiza por categorías `/Ejemplos/<Categoría>/` y agrega carpetas de solución con archivos sueltos (docs, scripts, workflows).

| Carpeta de solución | Contenido |
|---|---|
| `/.Utilities/`, `/.Utilities/end2end/` | `download_sectigo.ps1`, `simular.sh`, `simular_ui.sh`, los 3 flujos Maestro (`:2-11`) |
| `/Ejemplos/<Cat>/` | 23 proyectos; todos con `<Deploy />` salvo `Ejemplo_ws_Blazor` (`:68`) |
| `/Ejemplos/QR/<Librería>/` | subcarpetas por librería: `BarcodeScanner.Mobile.Maui`, `BarcodeScanning.Native.Maui`, `CameraScanner.Maui`, `ZXing.Net.Maui` (`:149-186`) |
| `/Ejemplos/Utils/Scripts/` | los 3 `.bat` (`:196-200`) |
| `/Ejemplos/Utils/Utilities/` | `simular.sh` otra vez, duplicado del de `/.Utilities/` (`:201-203`) |
| `/github.workflow/` | 17 `.yml` + `Pipelinea-Version.md` + `Readme.md` (`:204-224`) |

**El proyecto que falta: `Integrada/Ejemplo_Maui_Hibrida.Tests`.** La solución lista 23 de los 24 proyectos; el de tests no aparece. Se creó en el commit `a994257`, que **sí** editó el `.slnx` (sumó Utilities y renombró el workflow) pero no agregó el proyecto. Ninguna fuente explica por qué → motivo **no verificado**. Contexto verificable que lo hace autónomo: es `net10.0` plano con `UseMaui=true` (`Ejemplo_Maui_Hibrida.Tests.csproj:9-11`) y accede al código **linkeando fuentes**, no por `ProjectReference`, porque un `net10.0` no puede referenciar la app `net10.0-android` (`:31-40`). Es un proyecto de tests (`IsTestProject`, `:16`; xUnit `2.9.2`, `:21`); ninguna fuente del repo documenta el comando con que se corre (**no verificado**). Diseño de la suite: [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

Otras inconsistencias del `.slnx` (rutas verificadas con `git ls-files`):

| Línea | Ruta en la solución | Realidad |
|---|---|---|
| `:16-18` | `Docs/certificados-ssl/…` | la carpeta es `Docs/Certificados-SSL/`: ⚠️ se rompe en sistemas de archivos que distinguen mayúsculas |
| `:62` | `GPS/Docs_GPS/Readme.md` | no existe; la real es `GPS/Ejemplo_Docs_GPS/Readme.md` |
| `:111` | `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md` | no existe; la real es `Printer/Ejemplo_Docs_Printer/Borradores/Readme.md` |
| `:129` | `…/ficheros_claude.ia/Mainpage.XAML` | el archivo es `MainPage.XAML` (mayúsculas) |
| `:34` | `/Docs/web-hibrida/` vacía | `Docs/web-hibrida/` tiene 5 `.md` |
| `:101`, `:108` | `Printer/Ejemplo_Docs_Printer/Readme.md` | listado dos veces |
| `:204-224` | workflows | falta `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` y `Analisis/` |

---

## 13. TFMs, RIDs y versiones mínimas

| Proyecto(s) | `TargetFrameworks` | iOS mín. | Android mín. | Windows |
|---|---|---|---|---|
| 19 apps MAUI (todas salvo las de abajo) | `net10.0-android`; `+net10.0-ios` sólo si `IsOSPlatform('osx')` (p. ej. `GPS/Ejemplo_Maui_GPS/Ejemplo_Maui_GPS.csproj:4-5`) | 15.0 | 25.0 | — |
| `Phone/Ejemplo_Maui_DirectCall` | ídem | 15.0 | **21.0** (`Ejemplo_Maui_DirectCall.csproj:26`) | — |
| `Maps/Ejemplo_Maui_Mapas` | ídem `+ net10.0-windows10.0.19041.0` si Windows (`Ejemplo_Maui_Mapas.csproj:4-6`) | 15.0 | 25.0 | 10.0.17763.0 (`:26`) |
| `Red/Ejemplo_Maui_Connectivity` | android/ios | 15.0 | 25.0 | ⚠️ declara mín. Windows 10.0.17763.0 (`Ejemplo_Maui_Connectivity.csproj:25`) sin TFM Windows |
| `Integrada/Ejemplo_ws_Blazor` | `net10.0` (SDK Web) | — | — | — |
| `Integrada/Ejemplo_Maui_Hibrida.Tests` | `net10.0` + `UseMaui` | — | — | — |

Consecuencia: **en Windows/Linux sólo se compila Android**; `net10.0-ios` existe únicamente en macOS (por eso el CI es `macos-15`). Todas las apps: `ApplicationDisplayVersion 1.0`, `ApplicationVersion 1`.

Bloques de build comunes (ejemplo `Ejemplo_Maui_GPS.csproj`):

| Bloque | Líneas | Contenido |
|---|---|---|
| Android | `:33-36` | `AndroidLdFlags -Wl,-z,max-page-size=16384`; RIDs `android-arm;android-arm64;android-x86;android-x64` |
| Android Release/Debug | `:38-50` | `apk`, `aapt2`, sin APK por ABI; Debug con `EmbedAssembliesIntoApk` |
| iOS Release | `:54-64` | `UseInterpreter`, `PublishTrimmed`, `MtouchLink SdkOnly`, `--linkskip=System.Net.Mail/Ping`, AOT sólo para `ios-arm64` |

- `Printer/Ejemplo_ThermalPrinter/Ejemplo_ThermalPrinter.csproj` repite los bloques Android e iOS Release (`:30`/`:65`, `:51`/`:70`) y agrega uno para `iossimulator-arm64` con `-ignore_auto_link` y `NoWarn MT5209;MT5212;MT5213` (`:82-86`): es la «solución» anotada en `QR/Ejemplo_Docs_QR/cicd.md:248-254`.
- Los cuatro QR «simples» (`BSM/BSN/CS/ZN.LectorQR`) y `Camera/Ejemplo_Photo_MediaPicker` sólo tienen el bloque Android base: les faltan Android Release/Debug e iOS Release (p. ej. `QR/BSM.LectorQR/BSM.LectorQR.csproj:28-31`, `Camera/Ejemplo_Photo_MediaPicker/Ejemplo_Photo_MediaPicker.csproj:29`). En CI eso afecta a los cuatro QR simples, que sí tienen workflow.

### 13.1 Bundle id del workflow vs bundle id real

`simular.sh` desinstala, verifica y lanza por `PACKAGE_NAME`. Cruce `PACKAGE_NAME` ↔ `ApplicationId` ↔ `CFBundleIdentifier` de `Info.plist`:

| Workflow | `PACKAGE_NAME` | `.csproj` `ApplicationId` | `Info.plist` `CFBundleIdentifier` |
|---|---|---|---|
| `cd-ios-qr.CS.LectorQR_Dialog.yml:36` | `…qr.camerascanner_maui.dialog` | ⚠️ `…qr.camerascanner.maui.dialog` (`CS.LectorQR_Dialog.csproj:17`) | ⚠️ `…qr.camerascanner.maui.dialog` (`Info.plist:44`) |
| `cd-ios-qr.ZN.LectorQR.yml:36` | `…qr.zxing_net_maui.simple` | igual (`ZN.LectorQR.csproj:17`) | ⚠️ `com.ejemplos.devices.qr` (`Info.plist:44`) |
| `cd-ios-qr.ZN.LectorQR_Dialog.yml:36` | `…qr.zxing_net_maui.dialog` | igual (`ZN.LectorQR_Dialog.csproj:17`) | ⚠️ `com.ejemplos.devices.qr.dialog` (`Info.plist:44`) |
| los otros 14 (el huérfano no tiene proyecto) | coincide en los tres | | |

En CS_Dialog el desajuste es seguro: el `simctl listapps | grep` de `simular.sh:161` no encontraría la app. En ZN, cuál de los dos ids termina en el `.app` (el paso `Test entitlements` lo imprime) es **no verificado**. Origen del desajuste en ZN: el commit `2d625f5` renombró `QR/Ejemplo_LectorQR` → `QR/ZN.LectorQR` (R094) y `QR/Ejemplo_LectorQR_Dialog` → `QR/ZN.LectorQR_Dialog` (R096); el `Info.plist` conservó los bundle ids viejos (`com.ejemplos.devices.qr` y `…qr.dialog`, este último el mismo `PACKAGE_NAME` del workflow huérfano).

---

## 14. `.gitignore` relevante

Base: plantilla oficial `VisualStudio.gitignore` (`.gitignore:1-4`), 421 líneas.

| Patrón | Línea | Efecto en este repo |
|---|---|---|
| `*.env` | `:12` | ignora archivos de entorno |
| `[Dd]ebug/`, `[Rr]elease/`, `x64/`, `x86/`, `[Aa][Rr][Mm]64/` | `:21-30` | ⚠️ cualquier carpeta con esos nombres queda fuera, no sólo las de salida |
| `[Oo]bj/`, `**/[Bb]in/*` | `:32`, `:38` | salidas de build |
| `[Ll]og/`, `[Ll]ogs/` | `:34-35` | ⚠️ una carpeta `Logs/` de código quedaría ignorada |
| `.vs/` | `:44` | caché de Visual Studio |
| `#wwwroot/` (comentado) | `:46` | `wwwroot` **sí** se versiona |
| `*.pfx`, `*.publishsettings` | `:255-256` | certificados y perfiles de publicación |
| `.vscode/*` con excepciones | `:400-405` | conserva `settings/tasks/launch/extensions.json` |
| `**/Services/ApiKeys.cs` | `:420-421` | **única regla agregada a la plantilla** (el único cambio desde el commit inicial, en `4ca2772`): la API key de Google (§10) |

No se ignoran `frames/`, `debug_logs/`, `evidencia_app.gif` ni `recorrido.mp4`: si corrés los scripts de simulación en un clon, quedan como archivos sin seguimiento en la raíz.

---

## 15. Documentos de soporte dentro de `.github/workflows/`

| Archivo | Contenido | Nota |
|---|---|---|
| `Readme.md` (53 líneas) | Salidas de `dotnet workload --version/update/list/search` y `dotnet --version` en Windows («Necesario para configuar el yml», sic, `:4`) | Latin-1 (ISO-8859). No describe los workflows |
| `Pipelinea-Version.md` (15 líneas) | Dos entradas: `202606262101_ejemplos` (Rosetta x64 + `GoogleUtilities`, parametrización del script, `:2-11`) y `202606261239_ejemplos` (primera versión estandarizada, `:13-15`) | ⚠️ Las versiones que usan hoy los `.yml` (`202606300809/0822/1040`) no están registradas |
| `Analisis/log_1.md` (2110 líneas) | Build x64 de `Ejemplo_LectorQR_Dialog` con `PIPELINE_VERSION 202606261239` (`:37`, `:50`): falla por símbolos `GUL*` | Motivó el `GoogleUtilities` condicional |
| `Analisis/log_2.md` (102 líneas) | Corrida de `simular.sh` con app firmada ad-hoc x86_64: instala y registra, `privacy` falla y el paso cae por timeout a los 30 min (`:102`) | Evidencia de §6.3 y §8 |

---

## 16. Preguntas frecuentes y trampas

**¿Por qué mi push no dispara el pipeline de mi ejemplo?** Porque 17 de los 18 workflows sólo tienen `workflow_call` y nadie los llama (§2.1). Sólo la híbrida corre sola, y sólo ante cambios en `Integrada/Ejemplo_Maui_Hibrida/**`.

**¿Qué pasa con `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml`?** ⚠️ Es un workflow huérfano: en el commit `2d625f5` («alternativas qr») el proyecto `QR/Ejemplo_LectorQR_Dialog` se renombró a `QR/ZN.LectorQR_Dialog` (R096) y se agregaron los pares BSM/BSN/CS; el workflow viejo quedó apuntando a la ruta anterior. Además invoca `./Utils/simular.sh` (`:422-423`), una carpeta que no existe (`Utilities/`). Quedan referencias al proyecto viejo en comentarios de todas las plantillas (p. ej. `cd-ios-gps.Ejemplo_GPS.yml:78,220-221`), en `QR/Ejemplo_Docs_QR/cicd.md` y en `Analisis/`.

**¿`cd-ios-qr.BSM.LectorQR.yml` compila?** ⚠️ Tiene `BUILD_CONFIG_SIMULATOR` comentado (`:46`), así que `-c ${{ env.BUILD_CONFIG_SIMULATOR }}` (`:330`) queda vacío y `BASE_PATH` (`:344`) pierde el segmento de configuración. Se espera que falle (no verificado en corrida). El resto de los workflows lo define.

**¿Hay CI de tests?** No. El comentario de `Ejemplo_Maui_Hibrida.Tests.csproj:5-6` dice que la suite «corre en el runner de escritorio y en CI», pero ⚠️ ningún workflow ejecuta `dotnet test`.

**¿Se genera un IPA o un APK firmado?** No. Sólo `.app` de simulador con firma ad-hoc. Los `.csproj` traen AOT para `ios-arm64` y empaquetado `apk`, pero ningún workflow los usa.

**¿Por qué `Release` en el simulador?** Es lo que fija `BUILD_CONFIG_SIMULATOR`. En los proyectos que lo tienen (§13), activa el bloque iOS Release del `.csproj` (`UseInterpreter`, `MtouchLink SdkOnly`, sin AOT porque el RID no es `ios-arm64`). El pipeline además fuerza `-p:LinkMode=SdkOnly`. Ninguna fuente explica la elección → motivo **no verificado**.

**¿Qué hay que tener para agregar un workflow a un ejemplo nuevo?** Basta copiar un `.yml` de Gen. 2 y cambiar `PACKAGE_NAME`, `PROJECTS_ROOT`, `PROJECT_NAME`, `PROJECT_FILE`. El proyecto necesita `Platforms/iOS/Info.plist` válido (lo valida `plutil`), `Platforms/iOS/Entitlements.Development.plist` (firma) y `PACKAGE_NAME` igual al bundle id real (§13.1). Si usa ML Kit en iOS, ver §7.

**¿Puedo correr el pipeline localmente?** No hay script para eso. Los pasos son bash para macOS (`xip`, `xcode-select`, `PlistBuddy`, `simctl`) y suponen permisos de `sudo` y que se puede borrar Xcode de `/Applications`: no los corras tal cual en tu Mac.

**¿Los artefactos duran?** Un día (`retention-days: 1` en todos los `upload-artifact`).
