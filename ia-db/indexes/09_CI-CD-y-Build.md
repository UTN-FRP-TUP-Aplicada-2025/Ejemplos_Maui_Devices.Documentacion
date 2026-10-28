# 09 — CI/CD y build

> **Propósito:** mapa del ciclo de compilación del repositorio: qué proyectos existen y cómo están declarados (TFM, versiones mínimas, `ApplicationId`), cuáles figuran en la solución `.slnx`, qué ejemplos tienen pipeline de GitHub Actions y cuáles no, cómo se arma, firma, versiona, publica y «simula» el `.app` de iOS, y qué scripts sirven para el build local en Android.
>
> **Fuente primaria:** `.github/workflows/*.yml` (18 workflows), `Utilities/simular.sh`, `Utilities/simular_ui.sh`, `Utilities/end2end/*.yaml`, `Ejemplos_Devices/Ejemplos_Devices.slnx` y los 25 `.csproj` versionados. Contexto de decisión: ADR-0006 y ADR-0007 del repo hermano `Ejemplos_Maui_Devices.Documentacion` (`Ejemplos_Maui_Devices-docs/docs/04-decisions/`).
>
> Rutas relativas a la raíz del repo de código (commit `285f6fb`). Lo que no se pudo comprobar leyendo archivos va marcado «(no verificado)». Después de `285f6fb` entraron `85384c2` (mayúsculas de 4 rutas de `Ejemplos_Devices/Ejemplos_Devices.slnx`, ver §15 #6) y `cc80ce1` (mayúsculas de 2 enlaces en `Docs/web-hibrida/maui-hibrido.md`); ninguno toca código ni workflows.

---

## §1 Resumen en una tabla

| Aspecto | Estado observado | Fuente |
|---|---|---|
| Plataforma con CI | **Sólo iOS** (simulador), runner `macos-15` | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:51` |
| Android | Sin CI; build + deploy local con `.bat` (`dotnet build -t:Run`) | `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat:37` |
| Cantidad de workflows | 18 (`cd-ios-<categoría>.<Ejemplo>.yml`) | `.github/workflows/` |
| Workflows con disparador automático | 1 (`push` a `main` de la híbrida); los otros 17 sólo `workflow_call` | `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:5-15` |
| Firma | Ad-hoc (`codesign --sign "-"`), sin certificado ni perfil | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:337-369` |
| Artefactos | `.app.zip` del simulador + evidencia (GIF/frames o MP4) con `retention-days: 1` | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:397-433` |
| Xcode | Descargado de Google Drive con `gdown` (26.0), no el preinstalado | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:23-26,84-122` |
| .NET | SDK `10.0.300` por `dotnet-install.sh`; workloads `ios maui maui-ios` `10.0.100` | `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml:29-31,159-200` |
| Prueba de UI | Maestro 1.41.0 sólo en la híbrida; el resto captura screenshots → GIF | `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:48-49` |
| Tests unitarios en CI | **Ninguno** (no hay workflow con `dotnet test`) | `.github/workflows/*.yml` |
| `global.json` / `Directory.Build.props` / `nuget.config` | No hay ninguno versionado | `git ls-files` |

---

## §2 Inventario de proyectos (25 `.csproj`)

TFM base de todos los MAUI: `net10.0-android`, más `net10.0-ios` **sólo si se compila en macOS** (`Condition="$([MSBuild]::IsOSPlatform('osx'))"`, línea 5 de cada `.csproj`). En Windows/Linux el proyecto **no** tiene target iOS.

| # | Proyecto | Ruta | Tipo | Mín. iOS / Android | `ApplicationId` | En `.slnx` | Workflow |
|---|---|---|---|---|---|---|---|
| 1 | Ejemplo_Photo_MediaPicker | `Ejemplos_Devices/Camera/Ejemplo_Photo_MediaPicker` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.mediapicker` | sí | **no** |
| 2 | Ejemplo_Photo_MiMediaPicker_Callback | `Ejemplos_Devices/Camera/…_Callback` | MAUI app | 15.0 / 25.0 | `com.ejemplos.photo.mimediapicker.callback` | sí | sí |
| 3 | Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion | `Ejemplos_Devices/Camera/…` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.imagen.callback.normalizacion` | sí | sí |
| 4 | Ejemplo_Photo_MiMediaPicker_Task | `Ejemplos_Devices/Camera/…` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.mimediapicker.task` | sí | sí |
| 5 | Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion | `Ejemplos_Devices/Camera/…` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.imagen.callback.normalizacion.miselfie` | sí | sí |
| 6 | Ejemplo_Maui_GPS | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.gps` | sí | sí (con ApiKeys) |
| 7 | Ejemplo_Maui_Hibrida | `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.integrada.hibrida` | sí | sí (push + Maestro) |
| 8 | Ejemplo_Maui_Hibrida.Tests | `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida.Tests` | xUnit `net10.0` + `UseMaui` | — | — | **no** | **no** |
| 9 | Ejemplo_ws_Blazor | `Ejemplos_Devices/Integrada/Ejemplo_ws_Blazor` | `Microsoft.NET.Sdk.Web` `net10.0` | — | — | sí (sin `<Deploy/>`) | **no** |
| 10 | Ejemplo_Maui_Mapas | `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas` | MAUI app (+ `net10.0-windows10.0.19041.0` en Windows) | 15.0 / 25.0 / Win 10.0.17763.0 | `com.ejemplos.devices.mapas` | sí | **no** |
| 11 | Ejemplo_Maui_Dialer | `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer` | MAUI app | 15.0 / 25.0 | `com.ejemplos.phone.dialer` | sí | sí |
| 12 | Ejemplo_Maui_DirectCall | `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall` | MAUI app | 15.0 / **21.0** | `com.ejemplos.phone.directcall` | sí | sí |
| 13 | Ejemplo_MotorDSL | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.MotorDSL` | sí | sí |
| 14 | Ejemplo_MotorDSL_Dialog | `Ejemplos_Devices/Printer/Ejemplo_MotorDSL_Dialog` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.MotorDSL.Dialog` | sí | **no** |
| 15 | Ejemplo_ThermalPrinter | `Ejemplos_Devices/Printer/Ejemplo_ThermalPrinter` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.ThermalPrinter` | sí | **no** |
| 16 | Thermalprintermaui | `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/Prompts_Generacion_Apk_ejemplo/ficheros_claude.ia` | artefacto documental (`net8.0-android34.0`) | — / 21.0 | `com.companyname.thermalprintermaui` | sólo como `<File>` (`.slnx:132`) | no |
| 17 | BSM.LectorQR | `Ejemplos_Devices/QR/BSM.LectorQR` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.qr.barcodescanner_mobile_maui.simple` | sí | sí (x64) |
| 18 | BSM.LectorQR_Dialog | `Ejemplos_Devices/QR/BSM.LectorQR_Dialog` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.qr.barcodescanner_mobile_maui.dialog` | sí | sí (x64) |
| 19 | BSN.LectorQR | `Ejemplos_Devices/QR/BSN.LectorQR` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.qr.barcodescanner_native_maui.simple` | sí | sí |
| 20 | BSN.LectorQR_Dialog | `Ejemplos_Devices/QR/BSN.LectorQR_Dialog` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.qr.barcodescanner_native_maui.dialog` | sí | sí |
| 21 | CS.LectorQR | `Ejemplos_Devices/QR/CS.LectorQR` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.qr.camerascanner_maui.simple` | sí | sí |
| 22 | CS.LectorQR_Dialog | `Ejemplos_Devices/QR/CS.LectorQR_Dialog` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.qr.camerascanner.maui.dialog` | sí | sí (⚠️ §15.2) |
| 23 | ZN.LectorQR | `Ejemplos_Devices/QR/ZN.LectorQR` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.qr.zxing_net_maui.simple` | sí | sí |
| 24 | ZN.LectorQR_Dialog | `Ejemplos_Devices/QR/ZN.LectorQR_Dialog` | MAUI app | 15.0 / 25.0 | `com.ejemplos.devices.qr.zxing_net_maui.dialog` | sí | sí |
| 25 | Ejemplo_Maui_Connectivity | `Ejemplos_Devices/Red/Ejemplo_Maui_Connectivity` | MAUI app | 15.0 / 25.0 | `com.ejemplos.red.connectivity` | sí | **no** |

Recuento exacto:

| Conjunto | Cantidad | Detalle |
|---|---|---|
| Apps MAUI ejecutables | 22 | Camera 5 · GPS 1 · Integrada 1 · Maps 1 · Phone 2 · Printer 3 · QR 8 · Red 1 |
| Otros proyectos reales | 2 | `Ejemplo_ws_Blazor` (web) · `Ejemplo_Maui_Hibrida.Tests` (xUnit) |
| Artefacto documental | 1 | `Thermalprintermaui.csproj` (generado en un prompt de IA, no compila en la solución) |
| **Total `.csproj`** | **25** | `git ls-files '*.csproj'` |
| `<Project>` en `.slnx` | 23 | 22 apps + `Ejemplo_ws_Blazor` |
| Apps con workflow | 17 | ver §4 |
| Apps **sin** workflow | 5 | `Ejemplo_Photo_MediaPicker`, `Ejemplo_Maui_Mapas`, `Ejemplo_MotorDSL_Dialog`, `Ejemplo_ThermalPrinter`, `Ejemplo_Maui_Connectivity` |

Todas las apps declaran `ApplicationDisplayVersion 1.0` y `ApplicationVersion 1` (p. ej. `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Ejemplo_Maui_Hibrida.csproj:20-21`).

### §2.1 Preparación para CI por proyecto

Un ejemplo «está listo» para el pipeline cuando tiene `Platforms/iOS/Entitlements.Development.plist` (lo usa el `codesign`) y `CFBundleVersion`/`CFBundleShortVersionString` en `Info.plist` (lo lee el paso de versionado).

| Proyecto | Entitlements | Claves de versión en `Info.plist` | Workflow |
|---|---|---|---|
| 17 apps con workflow | sí | sí (`1.0` / `1.0`) | sí |
| `Ejemplo_Photo_MediaPicker`, `Ejemplo_MotorDSL_Dialog` | sí | sí | **no** (preparados, sin YAML) |
| `Ejemplo_Maui_Mapas`, `Ejemplo_ThermalPrinter`, `Ejemplo_Maui_Connectivity` | **no** | **no** | no |

Fuente: `git ls-files '*Platforms/iOS/*'`; p. ej. `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Platforms/iOS/Info.plist:47-51`.

---

## §3 La solución `Ejemplos_Devices/Ejemplos_Devices.slnx`

Formato XML nuevo (`.slnx`). No existe `.sln` clásico. Estructura de carpetas de solución:

```
/.Utilities/              → download_sectigo.ps1, simular.sh, simular_ui.sh        (.slnx:2-6)
/.Utilities/end2end/      → 3 flujos Maestro                                       (.slnx:7-11)
/Docs/ …                  → README raíz, certificado-ssl, otros, qr-nuget, web-hibrida (.slnx:12-34)
/Ejemplos/<Dominio>/      → <Project Path=… Id=…><Deploy/></Project>               (.slnx:35-194)
/Ejemplos/Utils/Scripts/  → los 3 .bat                                             (.slnx:196-200)
/Ejemplos/Utils/Utilities/→ simular.sh (otra vez)                                  (.slnx:201-203)
/github.workflow/         → 17 YAML + Pipelinea-Version.md + Readme.md             (.slnx:204-224)
```

- `<Deploy />` marca los 22 proyectos MAUI como desplegables desde Visual Studio; `Ejemplo_ws_Blazor` no lo lleva (`.slnx:68`).
- Los proyectos QR se agrupan por librería: `BarcodeScanner.Mobile.Maui`, `BarcodeScanning.Native.Maui`, `CameraScanner.Maui`, `ZXing.Net.Maui` (`.slnx:148-186`).
- Los `Id` de `Ejemplo_Maui_DirectCall` y `Ejemplo_MotorDSL` difieren sólo en el último dígito (`…4603c` / `…4603d`, `.slnx:85,90`).

Rutas de `<File>` que **no existen** en el árbol versionado ⚠️ (ver §15 #6): `:62` y `:111` (hasta `285f6fb` también `:16-18` y `:129`, corregidas por `85384c2`).

---

## §4 Catálogo de workflows

Todos: `runs-on: macos-15`, un único job `publish-ios`, `XCODE_VERSION 26.0`, `DOTNET_VERSION 10.0.300`, `DEVICE_SIMULATOR 'iPhone 17 Pro Max'`, `BUILD_CONFIG_SIMULATOR 'Release'` (salvo excepción).

| Workflow | Proyecto (`PROJECTS_ROOT/PROJECT_NAME`) | Disparador | RID simulador | Script | `upload-artifact` | `PIPELINE_VERSION` |
|---|---|---|---|---|---|---|
| `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback.yml` | Camera/Ejemplo_Photo_MiMediaPicker_Callback | `workflow_call` | arm64 | `./Utilities/simular.sh` | v4 | — |
| `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion.yml` | Camera/…_Callback_Normalizacion | `workflow_call` | arm64 | simular.sh | v4 | — |
| `cd-ios-camera.Ejemplo_Photo_MiMediaPicker_Task.yml` | Camera/…_Task | `workflow_call` | arm64 | simular.sh | v4 | — |
| `cd-ios-camera.Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion.yml` | Camera/…Selfie… | `workflow_call` | arm64 | simular.sh | v4 | — |
| `cd-ios-gps.Ejemplo_GPS.yml` | GPS/Ejemplo_Maui_GPS | `workflow_call` | arm64 | simular.sh | v4 | — |
| `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` | Integrada/Ejemplo_Maui_Hibrida | **`push` main** + `workflow_call` | arm64 | **`simular_ui.sh`** + Maestro | **v5** | `202606301040_ejemplos` |
| `cd-ios-phone.Ejemplo_Dialer.yml` | Phone/Ejemplo_Maui_Dialer | `workflow_call` | arm64 | simular.sh | v4 | — |
| `cd-ios-phone.Ejemplo_DirectCall.yml` | Phone/Ejemplo_Maui_DirectCall | `workflow_call` | arm64 | simular.sh | v4 | — |
| `cd-ios-printer.Ejemplo_MotorDSL.yml` | Printer/Ejemplo_MotorDSL | `workflow_call` | arm64 | simular.sh | v4 | — |
| `cd-ios-qr.BSM.LectorQR.yml` | QR/BSM.LectorQR | `workflow_call` | **x64** | simular.sh | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.BSM.LectorQR_Dialog.yml` | QR/BSM.LectorQR_Dialog | `workflow_call` | **x64** | simular.sh | v5 | `202606300822_ejemplos` |
| `cd-ios-qr.BSN.LectorQR.yml` | QR/BSN.LectorQR | `workflow_call` | arm64 | simular.sh | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.BSN.LectorQR_Dialog.yml` | QR/BSN.LectorQR_Dialog | `workflow_call` | arm64 | simular.sh | v5 | `202606301040_ejemplos` |
| `cd-ios-qr.CS.LectorQR.yml` | QR/CS.LectorQR | `workflow_call` | arm64 | simular.sh | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.CS.LectorQR_Dialog.yml` | QR/CS.LectorQR_Dialog | `workflow_call` | arm64 | simular.sh | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.ZN.LectorQR.yml` | QR/ZN.LectorQR | `workflow_call` | arm64 | simular.sh | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.ZN.LectorQR_Dialog.yml` | QR/ZN.LectorQR_Dialog | `workflow_call` | arm64 | simular.sh | v4 | `202606300809_ejemplos` |
| `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` | QR/Ejemplo_LectorQR_Dialog **(no existe)** | `workflow_call` | arm64 | `./Utils/simular.sh` **(no existe)** | v4 | `202606300809_ejemplos` |

- Los 17 con sólo `workflow_call` tienen el bloque `push:` **comentado** (p. ej. `cd-ios-gps.Ejemplo_GPS.yml:5-13`). Ningún YAML del repo los invoca (`grep "uses: ./"` sin resultados) y ninguno declara `workflow_dispatch` → no se pueden lanzar desde la pestaña Actions sin editar el YAML.
- El disparador de la híbrida filtra `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/**` excluyendo `**.md`, `.gitignore`, `.gitattributes` (`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:9-13`). Cambios en `Utilities/`, en el propio YAML, en `Ejemplo_ws_Blazor` o en `.Tests` **no** lo disparan.
- `PIPELINE_VERSION` sólo existe en la híbrida y en los QR; es una etiqueta informativa (no se usa en ningún paso).

---

## §5 Anatomía del pipeline iOS (plantilla común)

Referencia de líneas: `cd-ios-gps.Ejemplo_GPS.yml` (los demás son copias; diferencias en §6–§11).

```
checkout (actions/checkout@v4)                                 :62-65
│
├─ XCODE ── pipx install gdown → gdown <ID> → xip --expand      :84-106
│          sudo rm -rf /Applications/Xcode*.app; mv; ln -s;
│          xcode-select --switch; -runFirstLaunch; -license accept :108-122
│          xcrun simctl list runtimes; xcodebuild -downloadPlatform iOS :133-140
│          xcrun simctl boot "iPhone 17 Pro Max"                 :142-151
│
├─ .NET ── rm -rf ~/.dotnet → dotnet-install.sh --version 10.0.300 :155-182
│          dotnet workload install ios maui maui-ios --version 10.0.100 :198-210
│
├─ RUTAS ─ SOLUTION_PATH / PROJECT_PATH / PLIST_PATH → $GITHUB_ENV :218-238
├─ VERSIÓN PlistBuddy CFBundleVersion + CFBundleShortVersionString;
│          VERSION_FECHA=date +%Y%m%d%H%M                         :242-259
├─ MANIFEST plutil -lint Info.plist                              :263-267
├─ BUILD ─ dotnet clean -f net10.0-ios                           :271-277
│          [sólo GPS] ApiKeys.cs desde secreto                   :279-285
│          dotnet restore -p:TargetFramework -p:RuntimeIdentifier :287-293
│          dotnet build -c Release -f:net10.0-ios
│               -p:RuntimeIdentifier=iossimulator-arm64 -p:LinkMode=SdkOnly
│               -p:CLI_Build=true -p:CodesignEntitlements=…Development.plist
│               -p:EnableCodeSigning=false -p:CodesignProvision="" -p:CodesignKey="-" :295-333
├─ FIRMA ─ ad-hoc manual (§6)                                    :337-380
├─ ZIP ─── <FECHA>_<SHORT>_<BUNDLE>_<PACKAGE_NAME>.app.zip → upload :384-403
└─ SIMULA  brew install ffmpeg → ./Utilities/simular.sh (continue-on-error, 30 min)
           → upload "grabacion-simulador"                        :407-433
           "Clean up keychain…" (if: always(), sólo echo)        :435-438
```

Propiedades iOS en los `.csproj` (grupo `Release` + `-ios`): `UseInterpreter=true`, `PublishTrimmed=true`, `MtouchLink=SdkOnly`, `MtouchExtraArgs --linkskip=System.Net.Mail --linkskip=System.Net.Ping`, `EmbedAssembliesIntoApk=true` y `RunAOTCompilation` sólo para `ios-arm64` (device) (p. ej. `Ejemplo_Maui_Hibrida.csproj:54-64`). Las tienen **17 de las 22 apps**; ⚠️ **no** las tienen `Ejemplo_Photo_MediaPicker` ni los cuatro QR «simples» (`BSM.LectorQR`, `BSN.LectorQR`, `CS.LectorQR`, `ZN.LectorQR`, `.csproj` de 48–55 líneas), y los cuatro QR sí tienen workflow: su build Release en CI depende sólo de lo que pasa el YAML (`-p:LinkMode=SdkOnly`).

Propiedades Android: las 22 apps declaran `AndroidLdFlags -Wl,-z,max-page-size=16384` (páginas de 16 KB) y `RuntimeIdentifiers android-arm;android-arm64;android-x86;android-x64`; las mismas 17 de arriba suman `AndroidPackageFormat apk`, `AndroidCreatePackagePerAbi False` y, en Debug, `EmbedAssembliesIntoApk True` (`Ejemplo_Maui_Hibrida.csproj:33-50`; mínimo en `BSM.LectorQR.csproj:28-31`). No hay `AndroidKeyStore`/firma Android configurada en ningún `.csproj`.

---

## §6 Firma ad-hoc y entitlements

| Paso | Qué hace | Fuente |
|---|---|---|
| Build | `EnableCodeSigning=false`, `CodesignKey="-"`, `CodesignProvision=""` → el SDK no firma con identidad | `cd-ios-gps.Ejemplo_GPS.yml:313-316` |
| Limpieza | `xattr -cr` + borra `_CodeSignature` + `chmod +x` del binario | `:353-356` |
| Interior | `find *.dll *.dylib *.aotdata* \| xargs codesign --force --sign "-" --timestamp=none` | `:359` |
| Binario | `codesign … --entitlements Platforms/iOS/Entitlements.Development.plist <app>/<PROJECT_NAME>` | `:363` |
| Bundle | `codesign --force --sign "-"` del `.app` y `codesign -vv -d` | `:366-369` |
| Verificación | `PlistBuddy Print CFBundleIdentifier` + `codesign -d --entitlements :-` | `:373-380` |

`Entitlements.Development.plist` y `Entitlements.plist` declaran `aps-environment=development` y `get-task-allow=true` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Platforms/iOS/Entitlements.Development.plist:5-9`); hay sólo dos contenidos distintos entre los 38 archivos (difieren en BOM/espaciado). El log de una corrida confirma `Signature=adhoc`, `TeamIdentifier=not set` (`.github/workflows/Analisis/log_2.md:50-52`).

El paso final «Clean up keychain and provisioning profile» no crea ni usa keychain: en la plantilla sólo hace `echo` (`cd-ios-gps.Ejemplo_GPS.yml:435-438`); en la híbrida y QR borra `$RUNNER_TEMP/app-signing.keychain-db` si existiera (`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:468-473`). Es un resto de otra plantilla (los pasos comentados `CPY.` citan `/Users/runner/work/GDA.APP/GDA.APP`, `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:266-275`).

---

## §7 Versionado y artefactos

| Artefacto | Nombre | Contenido | Retención |
|---|---|---|---|
| App del simulador | `${VERSION_FECHA}_${SHORT}_${BUNDLE}_${PACKAGE_NAME}.app.zip`, p. ej. `202606270037_1.0_1.0_com.ejemplos.devices.qr.dialog.app.zip` | `zip -ry` del `.app` | 1 día |
| Evidencia (simular.sh) | `grabacion-simulador` | `evidencia_app.gif`, `frames/`, `debug_logs/`, `app_stream_full.txt`, `/tmp/app_stderr.txt` | 1 día |
| Evidencia (simular_ui.sh) | `grabacion-simulador` | `recorrido.mp4`, `debug_logs/`, `app_stream_full.txt` | 1 día |

Fuentes: `cd-ios-gps.Ejemplo_GPS.yml:384-433`, `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:406-466`, ejemplo real en `.github/workflows/Analisis/log_2.md:36`.

- `APP_VERSION_BUILD` sale del **`Info.plist` fuente** (`CFBundleShortVersionString` + `CFBundleVersion`, ambos `1.0`), no de `ApplicationDisplayVersion`/`ApplicationVersion` del `.csproj` (`1.0`/`1`) ⚠️ §15.9.
- No se publica `.ipa` de dispositivo ni APK: no hay build `ios-arm64` ni Android en ningún workflow.

---

## §8 Simulación: `simular.sh` vs `simular_ui.sh`

Ambos reciben por entorno `APP_PATH`, `PACKAGE_NAME`, `DEVICE_SIMULATOR`, `PROJECT_NAME` (los define el workflow); `simular.sh` usa además `PROJECTS_ROOT` (`simular.sh:271,282`).

| Aspecto | `Utilities/simular.sh` (16 workflows) | `Utilities/simular_ui.sh` (sólo híbrida) |
|---|---|---|
| Boot | `simctl boot` + sondeo 12×10 s (`:43-62`) | Precalentado en el workflow + `Simulator.app` por GUI, `bootstatus -b` 240 s, reintento con `erase` 300 s (`:62-96`) |
| Instalación | `uninstall` → `install` → `privacy grant notifications` → `listapps \| grep PACKAGE_NAME` (`:122-166`) | ídem, sin el fallback AppleScript (`:115-136`) |
| Lanzamiento | `simctl launch --stderr=/tmp/app_stderr.txt` (`:184`) | pre-warm `simctl launch` + `sleep 45` (`:155-157`) |
| Evidencia | 15 screenshots cada 0,5 s → GIF con `ffmpeg` (`:199-258`) | `simctl io recordVideo --codec h264` cerrado con `SIGINT` (`:167,190`) |
| Interacción | ninguna | Maestro `test -e APP_ID=<bundle> end2end/<PACKAGE_NAME>.yaml` (`:28,177`) |
| Salida | `exit 1` ante errores (`set -e`) | `exit 1` si falla boot, instalación o registro (`:93,126,135`); el fallo de Maestro no se propaga: termina en `exit 0` (`:231-233`) |
| Codificación | 24 caracteres U+FFFD literales en 22 líneas (acentos perdidos) | UTF-8 limpio |

El step que los invoca tiene `continue-on-error: true` y `timeout-minutes: 30`, así que una falla de simulación **no** marca el job en rojo (`cd-ios-gps.Ejemplo_GPS.yml:411-413`).

---

## §9 Pruebas end2end con Maestro

| Flujo | Usado por | Botones que toca | Estado frente al código |
|---|---|---|---|
| `Utilities/end2end/com.ejemplos.devices.integrada.hibrida.yaml` | `simular_ui.sh` vía workflow de la híbrida | `Geo Pos` → `Llamar` → `Leer QR` → punto `50%,92%` → `Volver` | Coincide con `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Pages/MainPage.xaml:53-56` |
| `Utilities/end2end/com.ejemplos.devices.gps.yaml` | ningún workflow (GPS usa `simular.sh`) | `Geo Posicionar`, `Llamar`, `Leer QR` | ⚠️ no existen en `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Pages/MainPage.xaml:15-26` |
| `Utilities/end2end/com.ejemplos.devices.qr.dialog.yaml` | ningún workflow; su bundle es el del proyecto inexistente `Ejemplo_LectorQR_Dialog` | ídem al de GPS | ⚠️ copia del flujo viejo |

Instalación: `curl -Ls https://get.maestro.mobile.dev | bash` con `MAESTRO_VERSION=1.41.0` (`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml:438-443`). El flujo usa `launchApp: { stopApp: false }` y `extendedWaitUntil: visible "Geo Pos" timeout 120000` para no grabar el arranque en frío (`com.ejemplos.devices.integrada.hibrida.yaml:41-49`; motivación en `CHANGELOG.md:83-101`). El detalle de qué hace cada botón de la híbrida está en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## §10 Simulador x64 + Rosetta (librerías ML Kit)

| Pieza | Dónde | Efecto |
|---|---|---|
| `RUNTIME_IDENTIFIER_SIMULATOR: iossimulator-x64` | `cd-ios-qr.BSM.LectorQR.yml:45`, `cd-ios-qr.BSM.LectorQR_Dialog.yml:45` | compila el simulador en x86_64 |
| `xcodebuild -downloadPlatform iOS -architectureVariant universal \|\| true` | `cd-ios-qr.BSM.LectorQR.yml:149-154` | runtime de simulador con slice x86_64 |
| `softwareupdate --install-rosetta --agree-to-license` con `if: env.RUNTIME_IDENTIFIER_SIMULATOR == 'iossimulator-x64'` | `cd-ios-qr.BSM.LectorQR.yml:420-428` | ejecuta el binario x86_64 en Apple Silicon |
| `AdamE.Google.iOS.GoogleUtilities 8.1.0.3` condicionado a `iossimulator-x64` | `BSM.LectorQR_Dialog.csproj:78-79`, `CS.LectorQR_Dialog.csproj:78-79`, `ZN.LectorQR_Dialog.csproj:78-79`, `Ejemplo_Maui_Hibrida.csproj:94-96` | resuelve `Undefined symbols for architecture x86_64: _GULOSLogBasic…` |

Historia: el log `.github/workflows/Analisis/log_1.md` (fin del archivo) muestra el fallo de link x86_64 por símbolos `GUL*` de `MLKitCommon`; `.github/workflows/Pipelinea-Version.md:2-9` registra la solución («la versión 9.0.1 no tiene esa librería para x64»). Justificación completa del enfoque Rosetta: `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md` (§1–§9). El fallo previo en arm64 (`building for 'iOS-simulator', but linking in object file … MLKitCommon[arm64]`) está en `Ejemplos_Devices/QR/Ejemplo_Docs_QR/cicd.md:121-141`. Los workflows de la híbrida, BSN, CS y ZN tienen el bloque condicional pero corren en `arm64`.

---

## §11 Secretos de build

| Mecanismo | Detalle | Fuente |
|---|---|---|
| Plantilla | `Services/ApiKeys.cs.template` con `GoogleMaps = "REEMPLAZAR_CON_TU_API_KEY"` | `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/ApiKeys.cs.template:11-13` |
| Ignorado | `**/Services/ApiKeys.cs` | `.gitignore:420-421` |
| CI | `sed` reemplaza el marcador por `secrets.GOOGLE_MAPS_API_KEY` y escribe `ApiKeys.cs` | `cd-ios-gps.Ejemplo_GPS.yml:279-285` |
| Resto de workflows | mismo paso **comentado**; en la híbrida y QR, eliminado | p. ej. `cd-ios-phone.Ejemplo_Dialer.yml:279-285` |

La existencia del secreto en el repositorio de GitHub no se puede verificar desde el código (no verificado). Sin `ApiKeys.cs` local, `GoogleMapService` no compila porque referencia `ApiKeys.GoogleMaps` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/GoogleMapService.cs:11`).

---

## §12 Build local (Windows / Android)

| Archivo | Qué hace | Fuente |
|---|---|---|
| `Ejemplos_Devices/scripts/Ejemplo_Maui_GPS_launch.bat` | agrega `%ProgramFiles(x86)%\Android\android-sdk\platform-tools` al `PATH`, exige dispositivo en `adb devices`, `dotnet build <csproj> -f net10.0-android -t:Run` | `:11-37` |
| `Ejemplos_Devices/scripts/Ejemplo_Photo_MediaPicker_launch.bat` | idéntico, apuntando a `Camera/Ejemplo_Photo_MediaPicker` | `:6` |
| `Ejemplos_Devices/scripts/GetEnviromentVersion.bat` | `dotnet --version` + `dotnet workload list --verbosity detailed` | `:1-3` |
| `vs.bat` (raíz) | una sola línea `code .` (abre **VS Code**, no Visual Studio) | `vs.bat:1` |
| `Utilities/download_sectigo.ps1` | baja intermedios + raíz de la cadena TLS de `aplicada.somee.com` como `.pem` a `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Platforms/Android/Resources/raw` e imprime el bloque `<certificates src="@raw/…">` | `:61-62,165-203` |

Sólo 2 de las 22 apps tienen `.bat`. `download_sectigo.ps1` aclara que el nombre es histórico: la CA real es Let's Encrypt / ISRG Root X2 (`:3-5`). El uso de los `.pem` en la app está en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

Entorno documentado del autor: SDK `10.0.102`, workloads `android 36.1.12`, `ios 26.2.10191`, `maui-android 10.0.1` (`.github/workflows/Readme.md:6-31,52-53`); otra captura con SDK `10.0.300` y requisito de 7zip para el SDK de Android (`Ejemplos_Devices/Docs/otros/GetEnviromentVersion.md:2-14`).

---

## §13 Paquetes NuGet por proyecto (versiones fijadas en `.csproj`)

| Paquete | Versión(es) | Proyectos |
|---|---|---|
| `Microsoft.Maui.Controls` | `$(MauiVersion)` (implícita del workload) | MediaPicker, GPS, Mapas, DirectCall, MotorDSL, ThermalPrinter, BSM ×2, CS ×2, ZN ×2, Connectivity |
| | `10.0.30` | MiMediaPicker_Callback, MiMediaPicker_Task |
| | `10.0.31` | Callback_Normalizacion, Selfie_Callback_Normalizacion |
| | `10.0.60` | Dialer, MotorDSL_Dialog |
| | `10.0.80` | Hibrida, BSN ×2 |
| `Microsoft.Extensions.Logging.Debug` | `10.0.0` / `10.0.2` / `10.0.3` / `10.0.8` / `10.0.9` | DirectCall / mayoría / MotorDSL ×2 y ThermalPrinter / Dialer / Hibrida y BSN ×2 |
| `CommunityToolkit.Maui.Camera` · `.Core` | `6.0.0` · `14.0.0` | 4 Camera `MiMedia*` |
| | `6.1.0` · `14.2.0` | Hibrida |
| `CommunityToolkit.Maui` · `CommunityToolkit.Mvvm` | `14.2.0` · `8.4.2` | Hibrida, MotorDSL_Dialog (Mvvm también en `.Tests`) |
| `MetadataExtractor` · `SkiaSharp` | `2.9.0` · `3.119.1` (Camera) / `3.119.2` (Hibrida) | 2 Camera `*Normalizacion`, Hibrida |
| `MotorDsl.*` (Core, Parser, Rendering, Extensions, Printing.Abstractions, Bluetooth, Maui) | `1.0.12` | MotorDSL |
| | `1.0.13` | MotorDSL_Dialog, Hibrida (`.Tests` sólo Core + Printing.Abstractions) |
| `BarcodeScanner.Mobile.Maui` | `9.0.1` | BSM ×2, ThermalPrinter |
| `BarcodeScanning.Native.Maui` | `3.0.4` | BSN ×2, Hibrida |
| `CameraScanner.Maui` | `1.8.31` | CS ×2 |
| `CameraMaui` | `1.4.5` | ZN ×2 |
| `AdamE.Google.iOS.GoogleUtilities` | `8.1.0.3` (sólo `iossimulator-x64`) | BSM_Dialog, CS_Dialog, ZN_Dialog, Hibrida |
| `Microsoft.Maui.Controls.Maps` | `10.0.40` | Mapas |
| `ESCPOS_NET` | `2.2.1` | ThermalPrinter (`4.5.0` en el artefacto `Thermalprintermaui`) |
| `Microsoft.AspNetCore.OpenApi` · `Scalar.AspNetCore` | `10.0.8` · `2.14.14` | Ejemplo_ws_Blazor |
| `Microsoft.NET.Test.Sdk` · `xunit` · `xunit.runner.visualstudio` | `17.11.1` · `2.9.2` · `2.8.2` | `.Tests` |

Otras propiedades singulares: la híbrida usa `MauiXamlInflator=SourceGen` (`Ejemplo_Maui_Hibrida.csproj:14`); `.Tests` usa `SkipValidateMauiImplicitPackageReferences=true` y **linkea fuentes** de `LibApp/` en vez de `ProjectReference` (`Ejemplo_Maui_Hibrida.Tests.csproj:11,31-40`).

---

## §14 Documentación de CI dentro de `.github/workflows/`

| Archivo | Contenido | Nota |
|---|---|---|
| `Readme.md` | salida de `dotnet workload --version/update/list/search` y `dotnet --version` en la máquina Windows del autor | ISO-8859-1; snapshot con SDK `10.0.102` |
| `Pipelinea-Version.md` | bitácora de versiones del pipeline: `202606261239_ejemplos` (YAML estandarizado, rutas relativas) y `202606262101_ejemplos` (GoogleUtilities para x64, parametrización del script) | ISO-8859-1; no registra `202606300809/0822/1040` |
| `Analisis/log_1.md` | log completo (2110 líneas) de un `dotnet build` x64 de `Ejemplo_LectorQR_Dialog` que termina en `1 Error(s)` de link x86_64 | material de diagnóstico |
| `Analisis/log_2.md` | log del step de simulación: instalación OK, `simctl privacy` falla, el step expira a los 30 min en «Lanzamiento de la APP» | `:84-102` |

---

## §15 Trampas ⚠️ y discrepancias verificadas

1. ⚠️ **Workflow huérfano.** `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml:40` apunta a `QR/Ejemplo_LectorQR_Dialog`, que ya no existe en el árbol, y a `./Utils/simular.sh` (`:422-423`) cuando la carpeta es `Utilities/`. Tampoco está listado en `.slnx:204-224`. Es la versión anterior de los lectores QR (el `.csproj` viejo está transcripto en `Ejemplos_Devices/QR/Ejemplo_Docs_QR/cicd.md:147-231`).
2. ⚠️ **Bundle id distinto en CS Dialog.** `cd-ios-qr.CS.LectorQR_Dialog.yml:36` usa `com.ejemplos.devices.qr.camerascanner_maui.dialog`, pero el `.csproj` declara `com.ejemplos.devices.qr.camerascanner.maui.dialog` (`CS.LectorQR_Dialog.csproj:17`). `simular.sh` busca la app por `PACKAGE_NAME` (`simular.sh:161-165`) → la verificación «fantasma» sale con `exit 1` y el script no llega al `launch`; como el step tiene `continue-on-error`, el job queda verde (el build y el zip no se ven afectados).
3. ⚠️ **`BUILD_CONFIG_SIMULATOR` comentado en BSM.LectorQR.** `cd-ios-qr.BSM.LectorQR.yml:46` deja la variable sin valor, pero se usa en `-c ${{ env.BUILD_CONFIG_SIMULATOR }}` (`:330`) y en `BASE_PATH` (`:344`). El comportamiento resultante de `dotnet build -c` vacío no se verificó en una corrida (no verificado).
4. ⚠️ **BSM.LectorQR en x64 sin GoogleUtilities.** Su workflow compila `iossimulator-x64` (`cd-ios-qr.BSM.LectorQR.yml:45`) pero `BSM.LectorQR.csproj` no tiene el `ItemGroup` condicional que `Pipelinea-Version.md:5-9` declara necesario (sí lo tiene `BSM.LectorQR_Dialog.csproj:78-79`). Resultado en CI no verificado.
5. ⚠️ **Flujos Maestro desactualizados.** `end2end/com.ejemplos.devices.gps.yaml:20` toca «Geo Posicionar», «Llamar», «Leer QR»; la app GPS tiene «Obtener ubicación», «Cancelar», «Mostrar en mapa», «Prueba», «Obtener Domicilio» (`GPS/Ejemplo_Maui_GPS/Pages/MainPage.xaml:15-26`). `end2end/com.ejemplos.devices.qr.dialog.yaml` es copia del mismo flujo. Ningún workflow los usa hoy.
6. ⚠️ **Rutas rotas en la `.slnx`.** Siguen rotas `GPS/Docs_GPS/Readme.md` (`:62`; real: `GPS/Ejemplo_Docs_GPS/`) y `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md` (`:111`; real: `Printer/Ejemplo_Docs_Printer/Borradores/`). Hasta `285f6fb` también fallaban, sólo por mayúsculas, `Docs/certificados-ssl/*` (`:16-18`) y `Mainpage.XAML` (`:129`); `85384c2` las corrigió. La carpeta `/Docs/web-hibrida/` está vacía (`:34`) aunque existen 5 `.md` en `Ejemplos_Devices/Docs/web-hibrida/`.
7. ⚠️ **Proyectos fuera de la `.slnx`.** `Ejemplo_Maui_Hibrida.Tests` no figura: abrir la solución no muestra la suite. Su comentario dice que «corre en el runner de escritorio y en CI» (`Ejemplo_Maui_Hibrida.Tests.csproj:5`), pero no hay workflow que ejecute `dotnet test`.
8. ⚠️ **ADR-0006 vs. código.** El ADR menciona «ApiKeys en GPS/Maps» como particularidad de los YAML y «captura GIF» como evidencia; en el código sólo GPS inyecta ApiKeys (Maps no tiene workflow) y la híbrida ya graba MP4 (`simular_ui.sh:167`).
9. ⚠️ **La versión del zip no es la del `.csproj`.** El nombre del artefacto usa `CFBundleVersion`/`CFBundleShortVersionString` del `Info.plist` fuente (`1.0_1.0`, `log_2.md:32`), no `ApplicationVersion=1`.
10. ⚠️ **`Info.plist` vs `ApplicationId` en ZN.** `ZN.LectorQR/Platforms/iOS/Info.plist:44` dice `com.ejemplos.devices.qr` y `ZN.LectorQR_Dialog/…/Info.plist:44` `com.ejemplos.devices.qr.dialog`, mientras los `.csproj` y los workflows usan `…zxing_net_maui.simple/.dialog`. Cuál de los dos queda en el `.app` compilado no se verificó (no verificado); el resto de los proyectos coincide.
11. ⚠️ **iOS sólo en macOS.** El TFM `net10.0-ios` se agrega condicionalmente; en Windows/Linux `dotnet build` nunca compila iOS y los `Platforms/iOS` no se validan.
12. ⚠️ **Mensajes engañosos en scripts.** `simular.sh:19` dice «No se encontró el simulador iPhone 16 Pro» aunque busca `${DEVICE_SIMULATOR}`; `LAUNCH_OUTPUT` nunca se asigna (la línea que lo hacía está comentada, `:183`) y por eso `debug_logs/launch_output.txt` y `APP_PID` quedan vacíos (`:186,191`); el predicado de `:271` va entre comillas simples y no expande `${PROJECTS_ROOT}/${PROJECT_NAME}`. En los YAML x64, el `echo` dice «Descargando runtime para simulador arm64» dentro de la rama x64 (`cd-ios-qr.BSM.LectorQR.yml:150`). `Ejemplo_Photo_MediaPicker_launch.bat:23` muestra el título «Ejemplo_Maui_GPS».
13. ⚠️ **Etiquetas copiadas.** El job del Selfie se llama «…Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion» (`cd-ios-camera.Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion.yml:54`) y usa otro instalador de Xcode (`Xcode_26_Universal_1.xip`, ID distinto, `:27-28`). El Dialer nombra el archivo `Xcode_26.3_Universal.xip` con el ID de Drive de 26.0 (`cd-ios-phone.Ejemplo_Dialer.yml:25-26`).
14. ⚠️ **Variables declaradas sin uso.** `DOTNET_WORKLOAD_INSTALL_ADDITIONAL_ARGS` y `PIPELINE_VERSION` no se referencian en ningún paso (`cd-ios-gps.Ejemplo_GPS.yml:46`): el `--ios-simulator-runtime=26.0` que declara la primera nunca llega a `dotnet workload install ios maui maui-ios --version …` (`:200`). En el paso de versionado los `echo` tienen las etiquetas cruzadas: llama «visible para los usuarios» a `CFBundleVersion` y «de compilación» a `CFBundleShortVersionString` (`:246-250`).
15. ⚠️ **Fallas de simulación silenciosas.** Por `continue-on-error: true` un job puede quedar verde aunque la app no arranque (caso de `Analisis/log_2.md:101-102`, timeout de 30 min).
16. ⚠️ **Detalles de `.csproj`.** `Ejemplo_ThermalPrinter.csproj` repite los grupos Android/iOS (`:30` y `:65`, `:51` y `:70`) y referencia `BarcodeScanner.Mobile.Maui` (`:102`) sin ser un lector QR; `Ejemplo_Maui_Connectivity.csproj:25-26` define versión mínima de Windows sin TFM Windows; `BSN.LectorQR.csproj:51-53` deja comentado el `$(MauiVersion)` y fija `10.0.80`; `Ejemplo_Maui_DirectCall` baja Android a 21.0 (`:26`).
17. ⚠️ **Codificación mixta.** `Readme.md` y `Pipelinea-Version.md` en ISO-8859-1; los YAML en UTF-8 con BOM; `simular.sh` con caracteres de reemplazo ya grabados.

---

## §16 Preguntas frecuentes

| Pregunta | Respuesta corta | Sección |
|---|---|---|
| ¿Qué ejemplos tienen pipeline y cuáles no? | 17 apps sí; 5 no (MediaPicker, Mapas, MotorDSL_Dialog, ThermalPrinter, Connectivity) | §2, §4 |
| ¿Por qué no hay CI de Android? | Decisión: Android se prueba local con `.bat` en dispositivo físico (ADR-0006) | §1, §12 |
| ¿Cómo se lanza un workflow que no sea el de la híbrida? | Sólo tienen `workflow_call`; hay que descomentar `push` o agregar `workflow_dispatch` | §4 |
| ¿Cómo se firma el `.app` sin cuenta de Apple? | Firma ad-hoc `codesign --sign "-"` con `Entitlements.Development.plist` | §6 |
| ¿Dónde bajo el `.app` compilado? | Artefacto `<fecha>_<ver>_<bundle>.app.zip`, retención 1 día | §7 |
| ¿Por qué el Xcode se baja de Google Drive? | Para fijar Xcode 26.0 independiente de la imagen del runner | §5 |
| ¿Por qué BSM compila para x64? | ML Kit no trae slice de simulador arm64; se usa x64 + Rosetta | §10 |
| ¿Qué hace Maestro y dónde están los flujos? | «Dedo virtual» sobre la UI nativa; `Utilities/end2end/<bundle>.yaml` | §9 |
| ¿Por qué el job está verde si la app no arrancó? | El step de simulación tiene `continue-on-error: true` | §8, §15.15 |
| ¿Cómo se inyecta la API key de Google en CI? | `sed` sobre `ApiKeys.cs.template` con `secrets.GOOGLE_MAPS_API_KEY` (sólo GPS) | §11 |
| ¿Qué proyectos abre la solución? | 23 (22 apps + Blazor); faltan `.Tests` y el artefacto `Thermalprintermaui` | §3, §15.7 |
| ¿Se ejecutan los tests en CI? | No | §1, §15.7 |
| ¿Qué versión de .NET/workloads usa el CI? | SDK 10.0.300, workloads `ios maui maui-ios` 10.0.100 | §5 |
| ¿Qué versión de MAUI usa cada ejemplo? | Varía: `$(MauiVersion)` o fija 10.0.30–10.0.80 | §13 |
| ¿Para qué sirve `download_sectigo.ps1`? | Regenera los `.pem` de trust anchors de la híbrida | §12 |
| ¿Por qué falla abrir algunos docs desde la solución? | Rutas `<File>` rotas o con mayúsculas distintas | §15 #6 |

---

## §17 Fuentes

| Archivo | Para qué se usó |
|---|---|
| `.github/workflows/cd-ios-gps.Ejemplo_GPS.yml` | plantilla de referencia (§5–§7, §11) |
| `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` | push, Maestro, precalentado, v5 (§4, §8, §9) |
| `.github/workflows/cd-ios-qr.*.yml` (9) | RID x64/arm64, Rosetta, `PIPELINE_VERSION`, discrepancias (§4, §10, §15) |
| `.github/workflows/cd-ios-camera.*.yml` (4), `cd-ios-phone.*.yml` (2), `cd-ios-printer.Ejemplo_MotorDSL.yml` | diferencias por `diff` contra la plantilla (§4, §15.13) |
| `.github/workflows/Readme.md`, `Pipelinea-Version.md`, `Analisis/log_1.md`, `Analisis/log_2.md` | entorno, bitácora, diagnósticos (§10, §14) |
| `Utilities/simular.sh`, `Utilities/simular_ui.sh` | simulación y evidencia (§8) |
| `Utilities/end2end/*.yaml` (3) | flujos Maestro (§9) |
| `Utilities/download_sectigo.ps1` | trust anchors (§12) |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | solución (§3, §15 #6) |
| `Ejemplos_Devices/scripts/*.bat`, `vs.bat` | build local (§12) |
| Los 25 `.csproj` | TFM, mínimos, `ApplicationId`, NuGets (§2, §13) |
| `*/Platforms/iOS/Info.plist`, `Entitlements*.plist` | preparación para CI, bundle ids (§2.1, §6, §15.10) |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/ApiKeys.cs.template`, `.gitignore` | secretos (§11) |
| `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Pages/MainPage.xaml`, `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Pages/MainPage.xaml` | contraste con flujos Maestro (§9) |
| `Ejemplos_Devices/QR/Ejemplo_Docs_QR/cicd.md`, `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md`, `Ejemplos_Devices/Docs/qr-nuget/03-matriz-rid.md`, `Ejemplos_Devices/Docs/otros/GetEnviromentVersion.md` | contexto de fallos de link y entorno (§10, §12) |
| `CHANGELOG.md` (entradas 2026-07-17 y 2026-07-18) | historia del CI de la híbrida (§9) |
| Repo hermano: `Ejemplos_Maui_Devices-docs/docs/04-decisions/0006-ci-ios-adhoc-android-local.md`, `0007-secretos-fuera-del-repo.md` | decisión de arquitectura (§1, §11, §15.8) |
