# 10 — Documentación transversal

> **Propósito**: mapa de la **prosa** del repositorio `Ejemplos_Maui_Devices`: qué documento explica cada porqué, de qué código habla, si sigue vigente frente al código actual y qué índice de dominio lo desarrolla. **No repite** contenido técnico: remite al índice del dominio.
>
> **Fuente primaria**: `README.md` y `CHANGELOG.md` de la raíz, `Ejemplos_Devices/Docs/**`, las carpetas `Ejemplos_Devices/*/Ejemplo_Docs_*/**`, los README dentro de proyectos y `.github/workflows/*.md`. Como contexto, el repo hermano `Ejemplos_Maui_Devices.Documentacion` (`Ejemplos_Maui_Devices-docs/docs/**`, `Analisis/`).
>
> **Estado indexado**: HEAD `cc80ce1` (`85384c2` corrige mayúsculas de 4 rutas en `Ejemplos_Devices/Ejemplos_Devices.slnx` y `cc80ce1` las de 2 enlaces en `Docs/web-hibrida/maui-hibrido.md`; el resto del árbol es igual a `285f6fb`). Las rutas son relativas a la raíz del repo de código salvo que se indique *(hermano)*.

---

## §1 Cómo usar este índice

| Si necesitás… | Andá a |
|---|---|
| Saber **qué documento** explica algo | §8 (porqués → documento) |
| Saber si un documento **sigue vigente** | §11 (vigencia frente al código) |
| Ver la prosa de **un dominio** | §6 (por dominio) y el índice hermano que se indica |
| Ubicar un **ADR** que cita el código | §10 |
| Conocer las **trampas** de lectura de la prosa | §13 |

Índices hermanos: [01_Camara](01_Camara.md) · [02_QR](02_QR.md) · [03_Impresion-Termica](03_Impresion-Termica.md) · [04_GPS](04_GPS.md) · [05_Mapas](05_Mapas.md) · [06_Telefonia](06_Telefonia.md) · [07_Red-Conectividad](07_Red-Conectividad.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md).

---

## §2 Inventario de la prosa versionada

Conteo con `git ls-files` (líneas por `wc -l`). Se excluyen imágenes y binarios.

```
README.md                                   24   portada: enlaces a 6 carpetas de docs
CHANGELOG.md                               371   Keep a Changelog, 2026-07-13 → 2026-07-23
.github/workflows/
├── Readme.md                               53   versión de workloads .NET (ISO-8859)
├── Pipelinea-Version.md                    14   bitácora de PIPELINE_VERSION (ISO-8859)
└── Analisis/log_1.md · log_2.md       2110+102  logs de corridas de CI
Ejemplos_Devices/Docs/
├── Certificados-SSL/  README 353 · Anexo-A-Comandos 223 · Anexo-B-Glosario 48
├── otros/             AsyncRelayCommandOptions 412 · Eventos 312 · GetEnviromentVersion 115 · propuesta-rosetta 192
├── qr-nuget/          README 52 · 01 158 · 02 174 · 03 47 · 04 77 · 05 258
└── web-hibrida/       maui-hibrido 677 · lectura-qr 393 · captura-foto 326 · envio-api 244 · llamada 222
Ejemplos_Devices/<Dominio>/Ejemplo_Docs_*/
├── Camera/Ejemplo_Docs_Photo/     Readme 81 · Transferencia_Foto_Entre_Pantallas 762
├── GPS/Ejemplo_Docs_GPS/          Readme 2 · secret 115 · servicio 129
├── Integrada/Ejemplo_Docs_Integrada/  Readme 0 (vacío)
├── Maps/Ejemplo_Docs_Maps/        Readme 0 (vacío)
├── Printer/Ejemplo_Docs_Printer/  Readme 21 + Manuales/ (5) + Borradores/ + Prompts_*/ + Pruebas_realizadas/
├── QR/Ejemplo_Docs_QR/            Readme 115 · cicd 263
└── Red/Ejemplo_Docs_Red/          Readme 64
README dentro de proyectos
├── Camera/Ejemplo_Photo_MediaPicker/MEDIAPICKER_FIX.md                 142
├── Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/README.md 125
├── Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/{README 0, Camera 146, Images 41, MotorDSL 65}
├── Integrada/Ejemplo_Maui_Hibrida/LibApp/UrlCommands/Handlers/Readme.md 0 (vacío)
├── Maps/Ejemplo_Maui_Mapas/Readme.md                                     4
├── Printer/Ejemplo_MotorDSL_Dialog/README.md                            39
└── QR/{BSM,BSN,CS,ZN}.LectorQR/Readme.md                          13 c/u (idénticos)
```

Dominio **Phone** no tiene carpeta `Ejemplo_Docs_*` ni README: su prosa vive sólo en `Docs/web-hibrida/llamada.md` (camino híbrido) y en el repo hermano (§10). Ver [06_Telefonia](06_Telefonia.md).

---

## §3 Mapa de proyectos y su cobertura documental

25 `.csproj` versionados. «En .slnx» = figura como `<Project>` en `Ejemplos_Devices/Ejemplos_Devices.slnx`. «Workflow» = existe `.github/workflows/cd-ios-*.yml` cuyo `PROJECT_FILE` lo nombra.

| Proyecto | Técnica | Prosa propia (dónde está el porqué) | En .slnx | Workflow | Índice |
|---|---|---|---|---|---|
| `Camera/Ejemplo_Photo_MediaPicker` | `MediaPicker` nativo | `MEDIAPICKER_FIX.md`; `Ejemplo_Docs_Photo/Readme.md` | sí | no | [01](01_Camara.md) |
| `Camera/Ejemplo_Photo_MiMediaPicker_Callback` | `CameraView` + callback | `Ejemplo_Docs_Photo/*` | sí | sí | [01](01_Camara.md) |
| `Camera/Ejemplo_Photo_MiMediaPicker_Task` | `CameraView` + `Task` | `Ejemplo_Docs_Photo/*` | sí | sí | [01](01_Camara.md) |
| `Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` | + EXIF/Skia | `README.md` del proyecto | sí | sí | [01](01_Camara.md) |
| `Camera/Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | selfie + máscara | (sin prosa propia) | sí | sí | [01](01_Camara.md) |
| `QR/BSM.LectorQR` · `_Dialog` | `BarcodeScanner.Mobile.Maui` 9.0.1 | `Readme.md` (sólo BSM.LectorQR); `Docs/qr-nuget/*` | sí | sí | [02](02_QR.md) |
| `QR/BSN.LectorQR` · `_Dialog` | `BarcodeScanning.Native.Maui` 3.0.4 | `Readme.md` copiado de BSM ⚠️ | sí | sí | [02](02_QR.md) |
| `QR/CS.LectorQR` · `_Dialog` | `CameraScanner.Maui` 1.8.31 | `Readme.md` copiado de BSM ⚠️ | sí | sí | [02](02_QR.md) |
| `QR/ZN.LectorQR` · `_Dialog` | `CameraMaui` 1.4.5 | `Readme.md` copiado de BSM ⚠️ | sí | sí | [02](02_QR.md) |
| `Printer/Ejemplo_ThermalPrinter` | ESCPOS_NET 2.2.1 | `Ejemplo_Docs_Printer/**` | sí | no | [03](03_Impresion-Termica.md) |
| `Printer/Ejemplo_MotorDSL` | MotorDsl.* 1.0.12 | (sin prosa propia) | sí | sí | [03](03_Impresion-Termica.md) |
| `Printer/Ejemplo_MotorDSL_Dialog` | MotorDsl.* 1.0.13 + overlay | `README.md` del proyecto | sí | no | [03](03_Impresion-Termica.md) |
| `Printer/Ejemplo_Docs_Printer/…/ficheros_claude.ia/Thermalprintermaui` | net8, ESCPOS_NET 4.5.0, Plugin.BLE | `Prompts_Generacion_Apk_ejemplo/Readme.md` | **no** (sólo como `<File>`) | no | [03](03_Impresion-Termica.md) |
| `GPS/Ejemplo_Maui_GPS` | `Geolocation` + geocodificación | `Ejemplo_Docs_GPS/{secret,servicio}.md` | sí | sí | [04](04_GPS.md) |
| `Maps/Ejemplo_Maui_Mapas` | `Microsoft.Maui.Controls.Maps` 10.0.40 | `Readme.md` del proyecto (4 líneas) | sí | no | [05](05_Mapas.md) |
| `Phone/Ejemplo_Maui_Dialer` · `Ejemplo_Maui_DirectCall` | `PhoneDialer` / intent | ninguna en el repo | sí | sí | [06](06_Telefonia.md) |
| `Red/Ejemplo_Maui_Connectivity` | `Connectivity` | `Ejemplo_Docs_Red/Readme.md` | sí | no | [07](07_Red-Conectividad.md) |
| `Integrada/Ejemplo_Maui_Hibrida` | WebView + puente URL | `Docs/web-hibrida/*`, `Docs/Certificados-SSL/*`, `LibApp/Devices/*/README.md`, `CHANGELOG.md` | sí | sí | [08](08_App-Hibrida-Integrada.md) |
| `Integrada/Ejemplo_ws_Blazor` | Blazor Interactive Server | `Docs/web-hibrida/maui-hibrido.md` §2–§3 | sí | no | [08](08_App-Hibrida-Integrada.md) |
| `Integrada/Ejemplo_Maui_Hibrida.Tests` | xUnit, linkeo de fuentes | `CHANGELOG.md` [2026-07-17]; comentarios del `.csproj` | **no** | no | [08](08_App-Hibrida-Integrada.md) |

Totales: 23 proyectos en la `.slnx`; 17 con workflow (18 `.yml`: `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` apunta a un proyecto que ya no existe, ver §13). Detalle en [09_CI-CD-y-Build](09_CI-CD-y-Build.md) §2–§4.

---

## §4 Documentación de la raíz

### §4.1 `README.md`

| Línea | Enlace | Texto |
|---|---|---|
| 5–9 | `Printer/Ejemplo_Docs_Printer/Readme.md` + GIF | «Ejemplo MAUI .NET y Documentación de la Impresora térmica 58H6» |
| 11–12 | `QR/Ejemplo_Docs_QR/Readme.md` | «En construcción …» |
| 14–15 | `Camera/Ejemplo_Docs_Photo/Readme.md` | «En construcción …» |
| 17–18 | `Red/Ejemplo_Docs_Red/Readme.md` | «En construcción …» |
| 20–21 | `GPS/Ejemplo_Docs_GPS/Readme.md` | «En construcción …» |
| 23–24 | `Maps/Ejemplo_Docs_Maps/Readme.md` | «En construcción …» (archivo vacío) |

Los seis enlaces resuelven. No hay entrada para **Integrada**, **Phone** ni `Ejemplos_Devices/Docs/`: la documentación más extensa del repo (`Docs/web-hibrida`, `Docs/Certificados-SSL`, `Docs/qr-nuget`) no se alcanza desde la portada.

### §4.2 `CHANGELOG.md`

Formato Keep a Changelog. Todas las entradas tocan la app híbrida o su CI; los ejemplos aislados no tienen historial aquí.

| Entrada | Línea | Tema | Desarrollado en |
|---|---|---|---|
| 2026-07-23 Panel con ambos modos de GPS + namespaces `LibApp.*` | 6 | segunda tarjeta GPS (`Injection`); `ApiRelayService`/`PrintCommandHandler` a `LibApp.*` | [08](08_App-Hibrida-Integrada.md) §5 |
| 2026-07-23 Puente: clasificación separada de ejecución (Plan 1) | 30 | `CommandDelivery`, `UrlPlan`, `Plan()`/`ExecuteAsync()`, centinela `0.0/0.0` | [08](08_App-Hibrida-Integrada.md) §3, §5 |
| 2026-07-18 Video end2end completo | 83 | pre-warm + `extendedWaitUntil` en Maestro | [09](09_CI-CD-y-Build.md) §8–§9 |
| 2026-07-18 Robustez del simulador iOS | 103 | boot por GUI de `Simulator.app`, timeout/retry | [09](09_CI-CD-y-Build.md) §8 |
| 2026-07-18 Flujo end2end rehecho | 131 | textos `Volver`·`Geo Pos`·`Llamar`·`Leer QR`; reactiva `push` | [09](09_CI-CD-y-Build.md) §9 |
| 2026-07-17 Armonización de overlays + primer proyecto de tests | 159 | 116 tests, 5 invariantes, `IGpsService`… | [08](08_App-Hibrida-Integrada.md) §6, §10 |
| 2026-07-16 UX de impresión | 240 | catálogo `PRN-*`, `DocumentResult`, H-1…H-7 | [03](03_Impresion-Termica.md) §6; [08](08_App-Hibrida-Integrada.md) §6 |
| 2026-07-15 Fix navegación WebView, overlay de impresión, permisos BT | 304 | doble navegación en pull-to-refresh; quita Windows | [08](08_App-Hibrida-Integrada.md) §9 |
| 2026-07-13 MotorDSL + reorganización a `LibApp/` | 338 | ~40 archivos movidos; backend `TikectsController` | [08](08_App-Hibrida-Integrada.md) §2 |

---

## §5 `Ejemplos_Devices/Docs/` — documentos transversales

### §5.1 `Certificados-SSL/` (guía TLS del WebView Android)

| Archivo | Contenido | Código que describe | Índice |
|---|---|---|---|
| `README.md` | Diagnóstico de `Trust anchor for certification path not found` con `https://aplicada.somee.com` en Moto g42; cadena ISRG Root X2 ← Root YE ← YE2; estrategia «raíz + 2 intermedios» (§4.2, §6); reglas de `res/raw` (§7); errores del setup original (§8) | `Integrada/Ejemplo_Maui_Hibrida/Platforms/Android/Resources/xml/network_security_config.xml`, `…/Resources/raw/*.pem`, `…/AndroidManifest.xml:9`, `Utilities/download_sectigo.ps1` | [08](08_App-Hibrida-Integrada.md) §9 |
| `Anexo-A-Comandos.md` | openssl, PowerShell + `X509Chain`, `adb logcat` de `cr_X509Util` | idem | — |
| `Anexo-B-Glosario.md` | glosario PKI/TLS | — | — |

Verificado: el XML actual coincide con el bloque de `README.md` §4.2 (sólo `system` + tres `@raw`). Ver ⚠️ T-6 sobre §4.3.

### §5.2 `otros/` (apuntes sueltos)

| Archivo | Tema | Código relacionado | Índice |
|---|---|---|---|
| `AsyncRelayCommandOptions.md` | clase didáctica `AsyncRelayCommand` sin `CommunityToolkit.Mvvm`; anti-reentrancia, `IsRunning`, comparación con el toolkit | `GPS/Ejemplo_Maui_GPS/ViewModels/AsyncRelayCommand.cs`, `Phone/Ejemplo_Maui_DirectCall/ViewModels/AsyncRelayCommand.cs` | [04](04_GPS.md), [06](06_Telefonia.md) |
| `Eventos.md` | `EventToCommandBehavior` + compiled bindings: por qué el comando no se invoca (§2) y por qué `e` llega `null` (§7); solución `Source={x:Reference page}` (§4) y `EventArgsConverter` (§7.4) | aplicado en `Integrada/Ejemplo_Maui_Hibrida/Pages/MainPage.xaml:31-37` y `LibApp/CustomWebView/Converts/WebNavigatingEventArgsConverter.cs` | [08](08_App-Hibrida-Integrada.md) §2 |
| `GetEnviromentVersion.md` | 7zip para el SDK de Android, NuGets usados, `dotnet --version`, `dotnet workload list` | `Ejemplos_Devices/scripts/GetEnviromentVersion.bat` | [09](09_CI-CD-y-Build.md) §12 |
| `propuesta-rosetta.md` | propuesta de compilar el simulador iOS en `iossimulator-x64` + Rosetta por ML Kit; 3 cambios de YAML, riesgos, rollback | workflows QR; `AdamE.Google.iOS.GoogleUtilities` condicional en `.csproj` | [02](02_QR.md) §6; [09](09_CI-CD-y-Build.md) §10 |

### §5.3 `qr-nuget/` (evaluación de librerías QR, 2026-06-27)

| Archivo | Responde | Índice |
|---|---|---|
| `README.md` | objetivo, hallazgo central (ML Kit sin slice arm64 de simulador), ranking TL;DR | [02](02_QR.md) §7 |
| `01-analisis-uso-actual.md` | uso de `BarcodeScanner.Mobile.Maui` 9.0.1 en la híbrida y en `Ejemplo_LectorQR_Dialog` | [02](02_QR.md) §7 |
| `02-alternativas-nuget.md` | fichas de BSN, CameraScanner, ZXing.Net.Maui, CameraMaui (janusw), comerciales | [02](02_QR.md) §7 |
| `03-matriz-rid.md` | matriz por RID/arquitectura | [02](02_QR.md) §6 |
| `04-recomendacion.md` | ranking y ruta de migración «no ejecutada» (§4.4) | [02](02_QR.md) §6 |
| `05-plan-migracion.md` | diffs de `QRLectorPage` hacia BSN | [02](02_QR.md) §4 |

Es un estudio **anterior** a los ocho proyectos `QR/{BSM,BSN,CS,ZN}.*` y a la migración de la híbrida a BSN (ver ⚠️ T-4).

### §5.4 `web-hibrida/` (arquitectura Web ↔ MAUI)

| Archivo | Comando / tema | Secciones que más se citan |
|---|---|---|
| `maui-hibrido.md` | principio del híbrido: Canal A (circuito SignalR de Blazor) y Canal B (puente URL); GPS en detalle; diagnóstico del fallo de interactividad en iOS | §3.1–§3.5, §5.1–§5.5, §6 asimetría por plataforma, §7 diagnóstico iOS |
| `lectura-qr.md` | `qr=qr&param=contenidoQR` | §4 secuencia, §7 «discrepancia del callback» |
| `captura-foto.md` | `photo=photo`, `selfie=selfie` | §4.5 normalización, §8 cableado `param` ↔ DOM |
| `llamada.md` | `phone=phone` | §4.5 estrategia por plataforma de `CallService` |
| `envio-api.md` | `sendAPI=sendAPI` (relay REST) | §4.5 `EntregarAlHook`, §7 observaciones |

Los cinco describen el puente **antes** del refactor «Plan 1» (2026-07-23) y antes de la reorganización a `LibApp/` en sus tablas de archivos: ver vigencia en §11 y desarrollo actual en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) §12. La carpeta figura en la `.slnx` **vacía** (`Ejemplos_Devices.slnx:34`).

---

## §6 Prosa por dominio (`Ejemplo_Docs_*` y README de proyectos)

| Dominio | Archivo | Qué contiene | Índice |
|---|---|---|---|
| Cámara | `Camera/Ejemplo_Docs_Photo/Readme.md` | dos enfoques (MediaPicker nativo vs diálogo propio con `CommunityToolkit.Maui.Camera`), permisos Android, `UseMauiCommunityToolkitCore()` | [01](01_Camara.md) §14 |
| Cámara | `Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` | guía conceptual: `Stream` vs `byte[]` vs archivo temporal vs singleton vs mensajería (§3), apps reales (§4), downsampling/EXIF (§5), `Dispatcher`/`MainThread` (§7), base64 vs multipart (§8), recomendación (§11) | [01](01_Camara.md) §6, §14 |
| Cámara | `Camera/Ejemplo_Photo_MediaPicker/MEDIAPICKER_FIX.md` | stream cerrado → crash de Glide en Debug; opción A `FromFile`, opción B `MemoryStream`; carpeta propia en galería | [01](01_Camara.md) §5 |
| Cámara | `Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/README.md` | NuGets, `AddServices()` con `IImageService`, uso en página, manifiestos | [01](01_Camara.md) §9 |
| QR | `QR/Ejemplo_Docs_QR/Readme.md` | instalación de BSM 9.0.1, `UseMauiCommunityToolkitCamera`, `Info.plist`, flags `MtouchExtraArgs`/`NoWarn` para conflictos de ML Kit | [02](02_QR.md) §6 |
| QR | `QR/Ejemplo_Docs_QR/cicd.md` | errores de compilación reales (`NETSDK1084`, linker ML Kit «built for iOS») sobre `Ejemplo_LectorQR_Dialog` | [02](02_QR.md) §6; [09](09_CI-CD-y-Build.md) §10 |
| QR | `QR/{BSM,BSN,CS,ZN}.LectorQR/Readme.md` | nombre del NuGet, repo de JimmyPun610, error `XAPRAS7028` | [02](02_QR.md) §8 |
| Impresión | `Printer/Ejemplo_Docs_Printer/Readme.md` | índice: prompts del APK, prompts de documentación, manuales, pruebas; dos videos de referencia | [03](03_Impresion-Termica.md) §11 |
| Impresión | `…/Manuales/{01..04,Readme}.md` | manual de usuario, técnico ESC/POS, guía ESCPOS_NET v2.2.1, borrador; `Readme.md` resume los tres primeros | [03](03_Impresion-Termica.md) §11 |
| Impresión | `…/Prompts_Generacion_Apk_ejemplo/` | prompt a claude.ai + ajuste con Copilot; `ficheros_claude.ia/` = código generado original (net8, Plugin.BLE) con su propia prosa: `Instalacion.md` (200 líneas) y `Thermalprintermaui_readme.md` (143) | [03](03_Impresion-Termica.md) §11 |
| Impresión | `…/Prompts_generacion_documentacion/` | metodología ChatGPT → Copilot para generar los manuales | [03](03_Impresion-Termica.md) §11 |
| Impresión | `…/Pruebas_realizadas/Readme.md` | evidencia: `Ejemplo_ThermalPrinter` en Android 13 (GIF + foto del ticket) | [03](03_Impresion-Termica.md) §11 |
| Impresión | `…/Borradores/Readme.md` | borrador de características + protocolo ESC/POS | [03](03_Impresion-Termica.md) §11 |
| Impresión | `Printer/Ejemplo_MotorDSL_Dialog/README.md` | snippet de `AddMotorDslEngine()`, perfil `thermal_58mm`, `AddBluetoothPrinterTransport()` | [03](03_Impresion-Termica.md) §3 |
| GPS | `GPS/Ejemplo_Docs_GPS/Readme.md` | sólo un enlace a Microsoft Learn (Geolocation) | [04](04_GPS.md) |
| GPS | `GPS/Ejemplo_Docs_GPS/secret.md` | 5 opciones para API keys; la cátedra usa la **opción 3** (`Services/ApiKeys.cs.template` + `.gitignore`) | [04](04_GPS.md) §6 |
| GPS | `GPS/Ejemplo_Docs_GPS/servicio.md` | 5 alternativas para que `TestService` use el GPS sin inyectar el VM; recomienda la 1 | [04](04_GPS.md) §8 |
| Mapas | `Maps/Ejemplo_Maui_Mapas/Readme.md` | enlace a Learn + NuGet `Microsoft.Maui.Controls.Maps` | [05](05_Mapas.md) |
| Red | `Red/Ejemplo_Docs_Red/Readme.md` | valores de `NetworkAccess` y snippet `ConnectivityTest` | [07](07_Red-Conectividad.md) §3–§4 |
| Integrada | `LibApp/Devices/Camera/README.md` | propósito, NuGets, registro de páginas de captura, manifiestos | [08](08_App-Hibrida-Integrada.md) §7 |
| Integrada | `LibApp/Devices/Images/README.md` | normalización con MetadataExtractor + SkiaSharp | [08](08_App-Hibrida-Integrada.md) §7 |
| Integrada | `LibApp/Devices/MotorDSL/README.md` | registro del motor y del overlay; permisos Bluetooth/ubicación del manifiesto | [08](08_App-Hibrida-Integrada.md) §6 |
| Integrada | `LibApp/Devices/README.md`, `LibApp/UrlCommands/Handlers/Readme.md`, `Ejemplo_Docs_Integrada/Readme.md` | **vacíos** (0 bytes) | — |

---

## §7 Prosa de CI (`.github/workflows/`)

| Archivo | Contenido | Índice |
|---|---|---|
| `Readme.md` | salidas de `dotnet workload --version/list/search` en Windows (workload 10.0.102) | [09](09_CI-CD-y-Build.md) §14 |
| `Pipelinea-Version.md` | bitácora `202606262101_ejemplos` (Rosetta, `GoogleUtilities` 8.1.0.3 para `iossimulator-x64`) y `202606261239_ejemplos` (primer YAML estandarizado) | [09](09_CI-CD-y-Build.md) §7 |
| `Analisis/log_1.md`, `log_2.md` | logs crudos de corridas (build iOS de `Ejemplo_LectorQR_Dialog`, firma ad-hoc) | [09](09_CI-CD-y-Build.md) §14 |

---

## §8 ¿Dónde está documentado cada porqué?

| Pregunta de diseño | Documento en el repo | ADR del hermano (§10) |
|---|---|---|
| ¿Por qué un proyecto mínimo por técnica? | — (sólo implícito en la estructura) | ADR-0001 |
| ¿Por qué servicio tipado + overlay MVVM? | `CHANGELOG.md:159-238` | ADR-0002 |
| ¿Por qué el puente por URL y no JS interop? | `Docs/web-hibrida/maui-hibrido.md` §5 | ADR-0003 |
| ¿Por qué cancelar la navegación de forma síncrona? | `Docs/web-hibrida/lectura-qr.md` §4.4; comentarios en `ViewModels/MainViewModel.cs:92-99` | ADR-0009 D1 |
| ¿Por qué `Injection` vs `Substitution` en el GPS? | `CHANGELOG.md:30-81`; `LibApp/UrlCommands/CommandDelivery.cs` | ADR-0009 D4 + «Plan 1» (no verificado, §10) |
| ¿Por qué BSN es la QR recomendada en iOS? | `Docs/qr-nuget/README.md`, `04-recomendacion.md` | ADR-0004 |
| ¿Por qué hace falta Rosetta / `iossimulator-x64`? | `Docs/otros/propuesta-rosetta.md`; `QR/Ejemplo_Docs_QR/cicd.md`; `.github/workflows/Pipelinea-Version.md:1-8` | ADR-0004, ADR-0006 |
| ¿Por qué motor DSL y no ESC/POS directo? | `Printer/Ejemplo_Docs_Printer/**` (sólo ESC/POS); `CHANGELOG.md:338-366` | ADR-0005 |
| ¿Por qué firma ad-hoc en CI? | — | ADR-0006 |
| ¿Cómo manejar API keys? | `GPS/Ejemplo_Docs_GPS/secret.md` §3 | ADR-0007 |
| ¿Por qué embeber raíz + intermedios TLS? | `Docs/Certificados-SSL/README.md` §4.2, §6 | ADR-0008 |
| ¿Por qué `Source={x:Reference page}` en los behaviors? | `Docs/otros/Eventos.md` §4 | — |
| ¿Por qué `AllowConcurrentExecutions = true` + guard? | comentario `MainViewModel.cs:86-90` | ADR-0009 D1 |
| ¿Por qué `InvariantCulture` en coordenadas? | comentario `GpsCommandHandler.cs:57-58` | ADR-0009 D2 |
| ¿Por qué los tests linkean fuentes y no referencian el proyecto? | `CHANGELOG.md:167-171`; `Ejemplo_Maui_Hibrida.Tests.csproj:34` | ADR-0001 |
| ¿Cómo pasar una foto entre páginas? | `Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` §11 | — |
| ¿Por qué el crash del `MediaPicker` sólo en Debug? | `Camera/Ejemplo_Photo_MediaPicker/MEDIAPICKER_FIX.md` | — |

---

## §9 La prosa dentro de `Ejemplos_Devices.slnx`

La solución lista documentos como `<File>` en carpetas virtuales `/Docs/` y `/Ejemplos/<Dominio>/Docs/`.

| Carpeta virtual | Archivos | Observación |
|---|---|---|
| `/Docs/` | `../README.md` | `CHANGELOG.md` no figura |
| `/Docs/certificado-ssl/` | 3 archivos de `Docs/Certificados-SSL/` | rutas corregidas en `85384c2` |
| `/Docs/otros/`, `/Docs/qr-nuget/` | 4 y 6 archivos | completos |
| `/Docs/web-hibrida/` | **ninguno** (`slnx:34`) | los 5 `.md` existen pero no se ven en el Explorador |
| `/Ejemplos/GPS/Docs/` | `GPS/Docs_GPS/Readme.md` (`slnx:62`) | ⚠️ ruta inexistente (es `GPS/Ejemplo_Docs_GPS/`); `secret.md` y `servicio.md` no figuran |
| `/Ejemplos/Printer/Docs/…/Borradores/` | `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md` (`slnx:111`) | ⚠️ ruta inexistente (es `Printer/Ejemplo_Docs_Printer/Borradores/`) |
| `/Ejemplos/Printer/Docs/` y `/…/Ejemplo_ThermalPrinter/` | `Printer/Ejemplo_Docs_Printer/Readme.md` | el mismo archivo listado dos veces (`slnx:101`, `slnx:108`) |
| `/github.workflow/` | 17 `.yml` + 2 `.md` | falta `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` y `Analisis/` |

Tampoco figuran: `Transferencia_Foto_Entre_Pantallas.md`, `MEDIAPICKER_FIX.md`, los README de proyectos, ni `LibApp/**/README.md` (estos últimos se ven igual porque están dentro de un proyecto SDK). Detalle de la `.slnx` en [09_CI-CD-y-Build](09_CI-CD-y-Build.md) §3.

---

## §10 Repo hermano `Ejemplos_Maui_Devices.Documentacion` (contexto)

Rutas *(hermano)* relativas a ese repo. Todo el corpus `Ejemplos_Maui_Devices-docs/` está en `status: draft` / `origin: reverse-engineered`, salvo ADR-0009 (`accepted`, `origin: designed`, v2.1.0, `last_review: 2026-07-17`).

### §10.1 ADR (`Ejemplos_Maui_Devices-docs/docs/04-decisions/`)

| ADR | Título | Citado desde el código |
|---|---|---|
| 0001 | Un proyecto MAUI mínimo por técnica | `Integrada/Ejemplo_Maui_Hibrida.Tests/Ejemplo_Maui_Hibrida.Tests.csproj:34` |
| 0002 | Servicio tipado + overlay MVVM | — |
| 0003 | Puente WebView ↔ nativo por comandos en la URL | — |
| 0004 | QR: BSN (Apple Vision) recomendada frente a BSM en iOS | — (se apoya en `Docs/qr-nuget/04-recomendacion.md`) |
| 0005 | Impresión: motor DSL sobre ESC/POS directo | — |
| 0006 | CI iOS ad-hoc; Android local | — (cita `Docs/otros/propuesta-rosetta.md`) |
| 0007 | Secretos fuera del repo | — |
| 0008 | Trust anchors embebidos en WebView | — (reconstruye `Docs/Certificados-SSL/README.md` §6) |
| 0009 | Cierre determinista del overlay GPS y cancelación robusta | `MainViewModel.cs:36,89`; `GpsCommandHandler.cs:12,58`; `GpsService.cs:37`; `GpsOverlayViewModel.cs:42`; tests `GpsCommandHandlerTests.cs:13`, `GpsOverlayTests.cs:20`, `NavigatingReentrancyTests.cs:14`, `Tests.csproj:73,80` |

### §10.2 Otros documentos del hermano que el código o la prosa citan

| Cita en el repo | Destino real | Estado |
|---|---|---|
| `Ejemplos_Maui_Devices.Documentos/Analisis/Plan-Armonizacion-Overlays.md` (`CHANGELOG.md:161`, `Tests/Invariantes.cs:9`) | *(hermano)* `Analisis/Plan-Armonizacion-Overlays.md` | existe; ⚠️ el repo se llama `.Documentacion`, no `.Documentos` |
| «Plan 1 §3/§4/§5/§7» (`CommandDelivery.cs:3,30`, `IUrlCommandHandler.cs:13,25`, `UrlCommandDispatcher.cs:36,47,78`, `GpsCommandHandler.cs:12,37,77-78`, `UrlCommandDispatcherTests.cs`) | sin archivo identificable en *(hermano)* `Analisis/` | (no verificado): sólo aparece como nombre del commit `0ef370b` en el `CHANGELOG.md` del hermano |
| `PrintThermal_Motor_Maui.Documentacion/Analisis/Analisis-UX-UI.md` (`CHANGELOG.md:242`) | otro repositorio | (no verificado) |
| `GDA.APP/APPGDA/Services/NotionsLocationsService.cs` (`CommandDelivery.cs:26`, `GpsCommandHandler.cs:74`) | otro repositorio | (no verificado) |

### §10.3 Resto del corpus del hermano (por tema)

| Tema | Documento *(hermano)* |
|---|---|
| Visión, mapa de piezas | `docs/00-overview/vision.md`, `system-map.md` |
| C4 contexto/contenedores, runtime, transversales | `docs/01-architecture/01-context.md`, `02-containers.md`, `04-runtime-views.md`, `06-crosscutting.md` |
| Overlays: fundamento y catálogo de pantallas | `docs/01-architecture/07-overlays-dispositivos.md`, `08-pantallas-por-dispositivo.md` |
| Contrato del puente y endpoints | `docs/pieces/integrada/bridge-contract.md`, `docs/05-apis/catalog.md` |
| Una pieza por dominio | `docs/pieces/{camera,gps,integrada,maps,phone,printer,qr,red}/README.md` |
| Build y onboarding | `docs/07-operations/build-and-run.md`, `docs/08-onboarding/developer-setup.md` |
| Derivas conocidas | `Ejemplos_Maui_Devices-docs/GAP-REPORT.md`, `GLOSSARY.md` |
| Guías de desarrollador | `Guias/Guias-Desarrollador/{README,Extender-Comando}.md` (vacíos, 0 bytes) |

---

## §11 Vigencia de la prosa frente al código actual

| Documento | Estado | Motivo verificado |
|---|---|---|
| `Docs/Certificados-SSL/README.md` | vigente salvo §4.3 | XML coincide con §4.2; ver T-6 |
| `Docs/otros/Eventos.md` | vigente | la solución §4 está aplicada en `MainPage.xaml:33,37` |
| `Docs/otros/AsyncRelayCommandOptions.md` | vigente, enlaces rotos | T-8 |
| `Docs/otros/propuesta-rosetta.md` | propuesta histórica | `GoogleUtilities` condicional sigue en `Ejemplo_Maui_Hibrida.csproj:94-96` y en `BSM/CS/ZN.*_Dialog` |
| `Docs/qr-nuget/*` | **desfasado** | T-4 |
| `Docs/web-hibrida/*` | **parcialmente desfasado** | T-1, T-2, T-3 |
| `CHANGELOG.md` | vigente | coincide con el código de `LibApp/UrlCommands/*` |
| `LibApp/Devices/MotorDSL/README.md` | desfasado para la híbrida | T-9 |
| `LibApp/Devices/{Camera,Images}/README.md` | versiones de NuGet viejas | T-10 |
| `QR/{BSN,CS,ZN}.LectorQR/Readme.md` | no corresponde al proyecto | T-5 |
| `Camera/Ejemplo_Docs_Photo/Readme.md` | parcial | habla de «dos ejemplos»; hay cinco proyectos de cámara |

---

## §12 Formato y codificación

| Archivo | Codificación detectada (`file`) | Efecto |
|---|---|---|
| `.github/workflows/Readme.md`, `Pipelinea-Version.md` | ISO-8859 | tildes como `�` en lectores UTF-8 |
| `Camera/Ejemplo_Docs_Photo/Readme.md` | ISO-8859 | idem («Introducci�n») |
| `Red/Ejemplo_Docs_Red/Readme.md` | extended-ASCII | idem («Ninguno � No hay…») |
| `.github/workflows/Analisis/log_2.md` | contiene `U+FFFD` | caracteres ya perdidos |
| resto de `.md` | UTF-8 | — |

Varios `.md` y `.csproj` empiezan con BOM (`﻿`). Los manuales de impresión y la carpeta `Prompts_*` son salida generada por IA y están presentados como tal (`Prompts_generacion_documentacion/Readme.md`).

---

## §13 Trampas de lectura y discrepancias ⚠️

| # | Dónde | Qué dice la prosa | Qué hace el código |
|---|---|---|---|
| T-1 | `Docs/web-hibrida/maui-hibrido.md:286-299`, `lectura-qr.md:155-170`, `llamada.md:25-26` (y análogos) | el VM llama `IsCommand` y luego `DispatchAsync` | `MainViewModel.cs:97,123` usa `Plan(url)` + `ExecuteAsync(plan,url)`; `IsCommand`/`DispatchAsync` sobreviven como envoltorios (`UrlCommandDispatcher.cs:92,98`). Los números de línea citados (`MainViewModel.cs:69-81`, `UrlCommandDispatcher.cs:15`, `MainPage.xaml.cs:18`) ya no coinciden: la URL está en `MainPage.xaml.cs:17`. Las tablas usan `Ejemplo_Maui_Hibrida/UrlCommands/…` sin `LibApp/` (`maui-hibrido.md:656`) |
| T-2 | `Docs/web-hibrida/lectura-qr.md:323-350` §7 | el QR nunca llega al DOM: el handler espera `OnQrCallback` que `QRLectorPage` no declara | resuelto: `QrCommandHandler.cs:34` dejó el `GoToAsync` con callback comentado y espera `destinoPage.ResultadoTask.Task` (`:41`), que `QRLectorPage.xaml.cs:132` completa |
| T-3 | `Docs/web-hibrida/captura-foto.md:287-298` §8.2 | la selfie manda `param=Selfie1` y no hay elemento con ese id | resuelto: `Panel.razor:252` manda `param=imgSelfie1` (existe en `:76`). §8.1 sigue vigente: `CameraCommandHandler.cs:61-62` escribe `.src` y `.value` sobre el mismo id |
| T-4 | `Docs/qr-nuget/README.md:3-5`, `01-analisis-uso-actual.md:12-14`, `04-recomendacion.md:56` | la híbrida usa `BarcodeScanner.Mobile.Maui` 9.0.1; migración «no ejecutada»; prototipo en `Ejemplo_LectorQR_Dialog` | `Ejemplo_Maui_Hibrida.csproj:91` ya usa `BarcodeScanning.Native.Maui` 3.0.4; `QR/Ejemplo_LectorQR_Dialog` no existe (lo reemplazan `QR/{BSM,BSN,CS,ZN}.*`) |
| T-5 | `QR/BSN.LectorQR/Readme.md:5`, `QR/CS.LectorQR/Readme.md:5`, `QR/ZN.LectorQR/Readme.md:5` | NuGet `BarcodeScanner.Mobile.Maui` | copias del Readme de BSM: los `.csproj` usan BSN 3.0.4, `CameraScanner.Maui` 1.8.31 y `CameraMaui` 1.4.5. ZN («ZXing.Net.Maui») **no** referencia `ZXing.Net.Maui` (ver [02_QR](02_QR.md) §8) |
| T-6 | `Docs/Certificados-SSL/README.md:212` | «el config actual del repo todavía incluye `user`» | `network_security_config.xml` no tiene ningún `src="user"` |
| T-7 | `Docs/qr-nuget/README.md:16`, `01-analisis-uso-actual.md:28` | enlace `../propuesta-rosetta.md` | roto: el archivo está en `Docs/otros/propuesta-rosetta.md`. (Los enlaces `../certificados-ssl/README.md` de `Docs/web-hibrida/maui-hibrido.md:488,677` fallaban por mayúsculas; los corrigió `cc80ce1`.) |
| T-8 | `Docs/otros/AsyncRelayCommandOptions.md:4-5,111,317` | enlaces `../GPS/…` y `../Phone/…` | rotos: desde `Docs/otros/` hace falta `../../`; los archivos existen en `GPS/Ejemplo_Maui_GPS/ViewModels/` y `Phone/Ejemplo_Maui_DirectCall/ViewModels/` |
| T-9 | `Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/MotorDSL/README.md:34` | `AddSingleton<PrinterService>()` | la híbrida registra `AddSingleton<IPrinterService, PrinterService>()` (`MauiProgram.cs:83`); el snippet coincide con `Printer/Ejemplo_MotorDSL_Dialog/MauiProgram.cs:63` |
| T-10 | `LibApp/Devices/Camera/README.md:12-13`, `LibApp/Devices/Images/README.md:12` | Camera 6.0.0 / Core 14.0.0 / SkiaSharp 3.119.1 | `Ejemplo_Maui_Hibrida.csproj:100,104-105`: SkiaSharp 3.119.2, Camera 6.1.0, Core 14.2.0 |
| T-11 | `Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/README.md:5-10` | dos NuGets | el `.csproj` suma `MetadataExtractor` 2.9.0 y `SkiaSharp` 3.119.1 |
| T-12 | `Ejemplos_Devices.slnx:62,111` | rutas de documentos | inexistentes (§9); `85384c2` sólo corrigió mayúsculas de otras cuatro |
| T-13 | `CHANGELOG.md:161`, `Tests/Invariantes.cs:9` | `Ejemplos_Maui_Devices.Documentos/…` | el repo hermano es `Ejemplos_Maui_Devices.Documentacion` |
| T-14 | `CHANGELOG.md:355-357` | agrega READMEs de `UrlCommands/Handlers` y `Devices`, y `Utilities/flows/recorrido.yaml` | esos dos README están vacíos; `recorrido.yaml` se borró en `a994257` (hoy `Utilities/end2end/*.yaml`) |
| T-15 | `Docs/web-hibrida/maui-hibrido.md:65,452` | botón nativo «Geo Posicionar» | `Pages/MainPage.xaml:54` dice «Geo Pos» (el `CHANGELOG.md:143` registra la corrección en Maestro) |
| T-16 | `Ejemplo_ws_Blazor/Components/Pages/Panel.razor:171-174` | comentario de `OnSolicitarCoordenadas`: «INYECTA… param={id}» | navega a `/geolocalizacion?coordenadas=coordenadas` **sin** `param` → `Substitution`. Las dos tarjetas dicen «Tomar Coordenadas» (`:18`, `:30`). Desarrollo en [08](08_App-Hibrida-Integrada.md) §5 |
| T-17 | `GpsCommandHandler.cs:75` | la web interpreta el centinela en `Panel.razor:176` | el centinela `0.0` vive en `GeoLocalizacion.razor:32-33` |
| T-18 | *(hermano)* ADR-0009 D4 | «Tomar Coordenadas» entrega por inyección | tras Plan 1 esa tarjeta usa `Substitution`; la inyección quedó en la tarjeta «Solicitar GeoPosicion». El ADR no se revisó después del 2026-07-17 |
| T-19 | *(hermano)* `README.md:26` | «Las 8 decisiones de arquitectura» | hay 9 ADR (0001–0009) |
| T-20 | `README.md:7` | impresora «58H6» | toda la documentación de impresión habla de `58HB6-THERMAL-PRINTER` |

Rasgos didácticos (no son defectos): «En construcción …» en la portada, Readme vacíos de Integrada/Maps, código generado por IA conservado en `ficheros_claude.ia/`, y logs crudos de CI en `.github/workflows/Analisis/`.

---

## §14 Preguntas frecuentes

| Pregunta | Respuesta corta | § |
|---|---|---|
| ¿Dónde está la explicación de cómo funciona la app híbrida? | `Docs/web-hibrida/maui-hibrido.md`, actualizado con `CHANGELOG.md` 2026-07-23 | §5.4, §11 |
| ¿Los documentos de `web-hibrida` describen el código actual? | sólo en parte: modelo previo a `UrlPlan`/`CommandDelivery` y líneas corridas | §13 T-1 a T-3 |
| ¿Dónde se justifica el `network_security_config.xml`? | `Docs/Certificados-SSL/README.md` + ADR-0008 | §5.1, §10.1 |
| ¿Por qué hay ocho proyectos QR y qué librería recomienda el repo? | `Docs/qr-nuget/04-recomendacion.md` (BSN) + ADR-0004 | §5.3, §8 |
| ¿Puedo confiar en el `Readme.md` de cada proyecto QR? | sólo en el de BSM | §13 T-5 |
| ¿Cómo configuro la API key de Google? | `GPS/Ejemplo_Docs_GPS/secret.md` opción 3 | §6 |
| ¿Qué es ADR-0009 y dónde está? | *(hermano)* `docs/04-decisions/0009-…md`; lo citan VM, handler GPS y tests | §10.1 |
| ¿Qué es el «Plan 1» que citan los comentarios? | refactor del puente del 2026-07-23; documento fuente no ubicado | §10.2 |
| ¿Qué cambió en la híbrida y cuándo? | `CHANGELOG.md` raíz | §4.2 |
| ¿Dónde está documentado Phone? | no hay prosa en el repo; `Docs/web-hibrida/llamada.md` y la pieza del hermano | §2, §10.3 |
| ¿Por qué algunos `.md` se ven con `�`? | codificación ISO-8859 | §12 |
| ¿Por qué no veo `web-hibrida` en Visual Studio? | la carpeta virtual de la `.slnx` está vacía | §9 |
| ¿De dónde salieron los manuales de la impresora? | generados con ChatGPT + Copilot; metodología documentada | §6 |
| ¿Por qué se recompila el simulador iOS en x64? | `propuesta-rosetta.md`, `cicd.md`, `Pipelinea-Version.md` | §5.2, §7 |

---

## §15 Fuentes

| Fuente | Uso en este índice |
|---|---|
| `README.md`, `CHANGELOG.md` | §4 |
| `Ejemplos_Devices/Docs/**` (18 archivos) | §5, §11, §13 |
| `Ejemplos_Devices/*/Ejemplo_Docs_*/**`, README de proyectos | §6, §13 |
| `.github/workflows/*.md`, `*.yml` (`PROJECT_FILE`, `on:`) | §3, §7 |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | §3, §9 |
| 25 `*.csproj` (TFM, `PackageReference`) | §3, §13 |
| `Integrada/Ejemplo_Maui_Hibrida/{ViewModels/MainViewModel.cs, Pages/MainPage.xaml(.cs), MauiProgram.cs, LibApp/UrlCommands/**, LibApp/Devices/QRLector/Pages/QRLectorPage.xaml.cs, Platforms/Android/**}` | verificación de §11 y §13 |
| `Integrada/Ejemplo_ws_Blazor/Components/Pages/{Panel,GeoLocalizacion}.razor` | T-3, T-16, T-17 |
| `Utilities/download_sectigo.ps1` | §5.1 |
| *(hermano)* `README.md`, `CHANGELOG.md`, `Analisis/Plan-Armonizacion-Overlays.md`, `Ejemplos_Maui_Devices-docs/docs/04-decisions/0001…0009`, listado de `docs/**` y `Guias/**` | §10 |
| `git log` (`85384c2`, `285f6fb`, `a994257`, `2d625f5`), `file` (codificación) | encabezado, §12, T-14 |
