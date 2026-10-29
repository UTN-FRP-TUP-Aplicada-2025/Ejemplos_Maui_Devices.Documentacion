# 10 — Documentación transversal

> **Propósito**: mapa de la prosa del repo. Dice **dónde está documentado cada porqué** (arquitectura híbrida, certificados SSL, Rosetta, MVVM, migración de QR, API keys…), qué cuenta el `CHANGELOG` por entrada y qué documentos del repo de documentación cita el código. Sirve para ubicar el documento y saber si está vigente. El contenido técnico de cada dominio se explica en su propio índice.
> **Fuente primaria**: `README.md`, `CHANGELOG.md`, `Ejemplos_Devices/Docs/**`, `Ejemplos_Devices/*/Ejemplo_Docs_*/**`, READMEs por proyecto y los comentarios del código que citan `ADR-*`, «Plan 1» y `Analisis/*` (repo de código `Ejemplos_Maui_Devices` @285f6fb). Repo de documentación hermano: `Ejemplos_Maui_Devices.Documentacion` (abajo, «repo doc»).
> **Índices relacionados**: [00_MASTER-INDEX](00_MASTER-INDEX.md) · [01_Camara](01_Camara.md) · [02_QR](02_QR.md) · [03_Impresion-Termica](03_Impresion-Termica.md) · [04_GPS](04_GPS.md) · [05_Mapas](05_Mapas.md) · [06_Telefonia](06_Telefonia.md) · [07_Red-Conectividad](07_Red-Conectividad.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md)

Las rutas se dan **relativas a la raíz del repo de código**, salvo las que empiezan con «repo doc:».

---

## 1. Mapa rápido: «¿dónde está el porqué de…?»

| Tema | Documento en el repo de código | Contraparte en el repo doc | Índice que explica el código |
|---|---|---|---|
| Arquitectura híbrida: WebView + Blazor Interactive Server remoto, «Canal A» (circuito SignalR) y «Canal B» (puente por URL) | `Ejemplos_Devices/Docs/web-hibrida/maui-hibrido.md` | ADR-0003 (puente por URL), `docs/01-architecture/*`, `docs/pieces/integrada/bridge-contract.md` | [08](08_App-Hibrida-Integrada.md) |
| Cada comando del puente (QR, foto/selfie, llamada, sendAPI) | `Docs/web-hibrida/{lectura-qr,captura-foto,llamada,envio-api}.md` | `docs/pieces/integrada/*` | [08](08_App-Hibrida-Integrada.md) |
| Por qué iOS muestra la web pero no es interactiva | `Docs/web-hibrida/maui-hibrido.md` §7 (líneas 501–616) | — | [08](08_App-Hibrida-Integrada.md) |
| `Trust anchor for certification path not found` (Android viejo, ISRG Root X2) | `Ejemplos_Devices/Docs/Certificados-SSL/README.md` + anexos A/B | ADR-0008 | [08](08_App-Hibrida-Integrada.md), [09](09_CI-CD-y-Build.md) (`download_sectigo.ps1`) |
| Simulador iOS en `x86_64` con Rosetta (ML Kit sin slice arm64) | `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md`; `QR/Ejemplo_Docs_QR/cicd.md` | ADR-0006 (CI iOS) | [09](09_CI-CD-y-Build.md), [02](02_QR.md) |
| Qué NuGet de QR usar y cómo migrar | `Ejemplos_Devices/Docs/qr-nuget/` (README + 01…05) | ADR-0004 (BSN recomendada en iOS) | [02](02_QR.md) |
| MVVM didáctico sin toolkit (`AsyncRelayCommand` propio) | `Docs/otros/AsyncRelayCommandOptions.md` | ADR-0002 (servicio tipado + overlay MVVM) | [04](04_GPS.md), [06](06_Telefonia.md) |
| `EventToCommandBehavior` + compiled bindings (eventos del `WebView` que no llegan al VM) | `Docs/otros/Eventos.md` | — | [08](08_App-Hibrida-Integrada.md) |
| Cómo un servicio usa el GPS sin acoplarse al VM | `GPS/Ejemplo_Docs_GPS/servicio.md` | ADR-0002 | [04](04_GPS.md) |
| API keys fuera del repo (`ApiKeys.cs.template`) | `GPS/Ejemplo_Docs_GPS/secret.md` | ADR-0007 | [04](04_GPS.md), [05](05_Mapas.md) |
| Pasar una foto entre pantallas (Stream, `byte[]`, archivo temporal…) | `Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` | `docs/pieces/camera/README.md` | [01](01_Camara.md) |
| Bug de Stream de `MediaPicker` (se resetea en Debug y no en Release) | `Camera/Ejemplo_Photo_MediaPicker/MEDIAPICKER_FIX.md` | — | [01](01_Camara.md) |
| Impresora 58HB6: manuales, ESC/POS y prompts que generaron el ejemplo | `Printer/Ejemplo_Docs_Printer/**` | ADR-0005 (motor DSL) | [03](03_Impresion-Termica.md) |
| Overlays: los cinco invariantes y el catálogo de pantallas | (sólo en tests y comentarios) | repo doc: `Analisis/Plan-Armonizacion-Overlays.md`, `docs/01-architecture/07-*`, `08-*` | [08](08_App-Hibrida-Integrada.md) |
| Un proyecto mínimo por técnica (duplicación deliberada) | (sólo en comentario de `.csproj` de tests) | ADR-0001 | [00](00_MASTER-INDEX.md) |
| Entorno: SDK, workloads, NuGet instalados | `Docs/otros/GetEnviromentVersion.md`; `.github/workflows/Readme.md` | `docs/08-onboarding/developer-setup.md`, `docs/07-operations/build-and-run.md` | [09](09_CI-CD-y-Build.md) |

En la columna «contraparte» se relaciona cada documento con un ADR **por su título**. Que el contenido de los dos coincida **no está verificado**: ver §11.

---

## 2. Inventario de la prosa del repo de código

```
README.md                         24 l.  portada: 6 enlaces a Ejemplo_Docs_* (§3)
CHANGELOG.md                     371 l.  Keep a Changelog, entradas por fecha (§4)
.github/workflows/{Readme,Pipelinea-Version}.md, Analisis/log_{1,2}.md   → índice 09
Ejemplos_Devices/
├── Docs/                                 documentación transversal (§5–§8)
│   ├── Certificados-SSL/  README (353) · Anexo-A-Comandos (223) · Anexo-B-Glosario (48)
│   ├── otros/             AsyncRelayCommandOptions (412) · Eventos (312)
│   │                      GetEnviromentVersion (115) · propuesta-rosetta (192)
│   ├── qr-nuget/          README (52) · 01…05 (158/174/47/77/258)
│   └── web-hibrida/       maui-hibrido (677) · lectura-qr (393) · captura-foto (326)
│                          llamada (222) · envio-api (244)
├── Camera/Ejemplo_Docs_Photo/    Readme (81) · Transferencia_Foto_Entre_Pantallas (762)
├── GPS/Ejemplo_Docs_GPS/         Readme (2, sólo un link) · secret (115) · servicio (129)
├── Integrada/Ejemplo_Docs_Integrada/  Readme (0 bytes)
├── Maps/Ejemplo_Docs_Maps/       Readme (0 bytes)
├── Printer/Ejemplo_Docs_Printer/ Readme (21) + Manuales/ (4 manuales + Readme)
│                                 Prompts_Generacion_Apk_ejemplo/ (+ ficheros_claude.ia/*.cs, .csproj, .XAML)
│                                 Prompts_generacion_documentacion/ · Borradores/ · Pruebas_realizadas/
│                                 Ejemplo_MotorDSL/flowchat.{bmp,png}
├── QR/Ejemplo_Docs_QR/           Readme (115) · cicd (263)
└── Red/Ejemplo_Docs_Red/         Readme (64)
```

READMEs dentro de los proyectos (detalle en el índice de cada dominio):

| Archivo | Contenido | Índice |
|---|---|---|
| `Camera/Ejemplo_Photo_MediaPicker/MEDIAPICKER_FIX.md` | causa (logcat de Glide) y estrategias de carga de imagen | [01](01_Camara.md) |
| `Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion/README.md` | guía rápida: NuGets y registro de servicios | [01](01_Camara.md) |
| `Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/{Camera,Images,MotorDSL}/README.md` | guías rápidas de cada módulo de `LibApp` | [08](08_App-Hibrida-Integrada.md) |
| `Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/README.md`, `LibApp/UrlCommands/Handlers/Readme.md` | **vacíos** (0 líneas) | — |
| `Maps/Ejemplo_Maui_Mapas/Readme.md` | link a Microsoft Learn + nombre del NuGet `Microsoft.Maui.Controls.Maps` | [05](05_Mapas.md) |
| `Printer/Ejemplo_MotorDSL_Dialog/README.md` | fragmento de `MauiProgram` (`AddPrintServices`, `AddMotorDslEngine`) | [03](03_Impresion-Termica.md) |
| `QR/{BSM,BSN,CS,ZN}.LectorQR/Readme.md` | **los cuatro son idénticos** (mismo md5), ⚠️ ver §12 | [02](02_QR.md) |

⚠️ **La solución no lista igual lo que hay en disco.** `Ejemplos_Devices/Ejemplos_Devices.slnx:15-19` agrupa la guía SSL en la carpeta virtual `/Docs/certificado-ssl/` (singular) con rutas `Docs/certificados-ssl/…` (minúscula), y la carpeta real es `Docs/Certificados-SSL/`: en Windows abre igual, en un sistema que distingue mayúsculas no. Además, `slnx:34` declara `/Docs/web-hibrida/` **vacía**: los cinco `.md` de la carpeta existen pero no aparecen en el Explorador de soluciones.

---

## 3. README raíz (`README.md`)

Es una portada con seis secciones y nada más: no describe el stack, ni cómo compilar, ni la app híbrida.

| Línea | Enlace | Estado declarado |
|---|---|---|
| 5–9 | Thermal Printer BlueTooth → `Printer/Ejemplo_Docs_Printer/Readme.md` + gif de prueba | completo («Impresora térmica 58H6») |
| 11–12 | Lectura de QR → `QR/Ejemplo_Docs_QR/Readme.md` | «En construcción …» |
| 14–15 | Cámara → `Camera/Ejemplo_Docs_Photo/Readme.md` | «En construcción …» |
| 17–18 | Red y Conectividad → `Red/Ejemplo_Docs_Red/Readme.md` | «En construcción …» |
| 20–21 | GPS → `GPS/Ejemplo_Docs_GPS/Readme.md` | «En construcción …» |
| 23–24 | Maps → `Maps/Ejemplo_Docs_Maps/Readme.md` | «En construcción …» |

- Las carpetas «en construcción» son **deliberadas**: el repo es material de cátedra que crece por tema.
- ⚠️ La portada no enlaza `Phone/`, `Integrada/` ni nada de `Ejemplos_Devices/Docs/`. Lo más elaborado del repo (web-hibrida, SSL, qr-nuget) **sólo se encuentra navegando** o desde la `.slnx`.
- ⚠️ `README.md:7` dice «58H6»; los manuales hablan del modelo **58HB6** (`Printer/Ejemplo_Docs_Printer/Manuales/Readme.md:1`).
- ⚠️ El enlace de Maps (`README.md:23`) apunta a un archivo de 0 bytes.

---

## 4. CHANGELOG (`CHANGELOG.md`): resumen por entrada

El formato es Keep a Changelog (`CHANGELOG.md:4`). **No hay números de versión**: cada entrada lleva fecha y título. Casi todo el historial registrado trata de `Integrada/` (app híbrida + backend Blazor + tests + simulación end2end). De los ejemplos aislados sólo aparece `Ejemplo_MotorDSL_Dialog`.

| Líneas | Fecha | Tema | Qué deja (en una línea) | Índice |
|---|---|---|---|---|
| 6–28 | 2026-07-23 | Panel con ambos modos de GPS + namespaces `LibApp.*` | tarjeta «Solicitar GeoPosicion» (`Injection`) convive con la que el CHANGELOG llama «Tomar Coordenadas» (`Substitution`; ver D-13); `ApiRelayService` y `PrintCommandHandler` pasan a `LibApp.*` | [08](08_App-Hibrida-Integrada.md) |
| 30–81 | 2026-07-23 | Puente por URL: clasificación separada de ejecución («Plan 1») | `CommandDelivery` (`None`/`Injection`/`Substitution`), `UrlPlan`, página `/geolocalizacion`, GPS re-navega siempre (centinela `0.0/0.0`), cancelación como OR, *default interface members* en `IUrlCommandHandler` | [08](08_App-Hibrida-Integrada.md) |
| 83–101 | 2026-07-18 | Video end2end completo | Maestro `extendedWaitUntil` "Geo Pos" 120 s; *pre-warm* antes de `recordVideo` | [09](09_CI-CD-y-Build.md) |
| 103–129 | 2026-07-18 | Arranque del simulador iOS en CI | boot abriendo `Simulator.app` (el boot headless se colgaba en «Waiting on BackBoard»); timeout 240 s + retry 300 s | [09](09_CI-CD-y-Build.md) |
| 131–157 | 2026-07-18 | Flujo end2end rehecho + CI activado | textos verificados en Moto G42 (`Volver`·`Geo Pos`·`Llamar`·`Leer QR`); se reactiva el `push` del workflow de la híbrida | [09](09_CI-CD-y-Build.md) |
| 159–238 | 2026-07-17 | Armonización de overlays + primer proyecto de tests | `Ejemplo_Maui_Hibrida.Tests` (116 xUnit, linkeo de fuentes), invariantes I-1…I-5, interfaces `I*Service`, catálogos `GPS-*`/`TEL-*`, GPS deja de ser mudo; workflow `cd-ios-gps…` → `cd-ios-Integrada…` | [08](08_App-Hibrida-Integrada.md), [09](09_CI-CD-y-Build.md) |
| 240–302 | 2026-07-16 | UX de impresión | catálogo `PRN-*` (14 códigos), `DocumentResult` tipado, `BluetoothOff` alcanzable, alias/MAC para homónimas, «Elegir otra» con `forzarSelector` | [03](03_Impresion-Termica.md), [08](08_App-Hibrida-Integrada.md) |
| 304–336 | 2026-07-15 | WebView, overlay de impresión y permisos BT | fin de la doble navegación en pull-to-refresh (`NavigateTo=null`); permisos Bluetooth en el manifest; se quita Windows de la híbrida; `MotorDsl.*` 1.0.13 en `Ejemplo_MotorDSL_Dialog` | [08](08_App-Hibrida-Integrada.md), [03](03_Impresion-Termica.md) |
| 338–371 | 2026-07-13 | Impresión MotorDSL + reorganización a `LibApp/` | `PrintCommandHandler` (`action=print`), `LibApp/Devices/MotorDSL/`, backend `TikectsController`, ~40 archivos movidos a `LibApp/` | [08](08_App-Hibrida-Integrada.md), [03](03_Impresion-Termica.md) |

Convenciones de forma (útiles para escribir una entrada nueva):
- Encabezado `## [AAAA-MM-DD] — <título temático>` y, en la mayoría, una línea `Alcance:` debajo con las rutas afectadas (`CHANGELOG.md:8`, `:32`, `:85`…).
- Subsecciones estándar `### Agregado` (6), `### Cambiado` (7), `### Corregido` (6), `### Eliminado` (1), más dos propias: `### Notas` (2) y `### Activado` (1, `:152`, la reactivación del CI de la híbrida).
- Las entradas explican **el defecto que motivó el cambio**, no sólo el cambio (p. ej. `:34-37`, los dos defectos del modelo anterior del puente).

Constantes del historial:
- **`MotorDsl.*` no se modifica desde este repo**: sigue en 1.0.13 (`CHANGELOG.md:163`, `:244`). Es un NuGet externo.
- Varias entradas dejan **pendiente la prueba en dispositivo real** (`:234-236`, `:302`). Toda espera del overlay **sigue sin poder cancelarse** (`:237-238`).
- Los documentos externos que cita el CHANGELOG se verifican en §10.

---

## 5. `Ejemplos_Devices/Docs/Certificados-SSL/`

Es una guía didáctica (es-AR) para el error `Trust anchor for certification path not found` del WebView de la híbrida en Android. El caso de estudio es `aplicada.somee.com` en un Moto g42 «Android 12/13» (`README.md:6`).

| Documento | Qué tiene |
|---|---|
| `README.md` §1 (l. 17) | TL;DR: la raíz es **ISRG Root X2** (Let's Encrypt, ECDSA 2020) y falta en Android viejos; **no es Sectigo** |
| §2 (l. 43) | PKI en 5 minutos: cadena, trust anchor, por qué el server no manda la raíz, DER vs PEM |
| §3 (l. 107) | diagnóstico: mensaje `cr_X509Util` y cadena real |
| §4 (l. 153) | solución: `network_security_config.xml` + `res/raw/*.pem` + enganche en el manifest |
| §5 (l. 232) | procedimiento reproducible (OpenSSL / PowerShell `X509Chain` / adb) |
| §6 (l. 301) | mantenimiento: sólo raíz (hasta 2040) vs raíz + intermedios (se rompe cuando LE rota `YE2`/`Root YE`) |
| §7 (l. 316) | reglas de `res/raw`: `[a-z0-9_]`, sin dos extensiones para el mismo nombre base, `@raw/` sin extensión |
| §8 (l. 327) | errores del setup original: dominio `gobdigital.com.ar`, `@raw` inexistente, Sectigo, mayúsculas, `src="user"` |
| `Anexo-A-Comandos.md` | `openssl s_client -showcerts`, script `X509Chain`, `adb logcat` filtrado |
| `Anexo-B-Glosario.md` | glosario PKI/TLS en tabla |

Lo que se aplicó en el código (verificado):

| Afirmación de la guía | Código |
|---|---|
| `domain-config` para `aplicada.somee.com` con `system` + 3 `@raw` | `Integrada/Ejemplo_Maui_Hibrida/Platforms/Android/Resources/xml/network_security_config.xml:14-22` |
| tres `.pem` en minúscula | `…/Platforms/Android/Resources/raw/{isrg_root_x2,root_ye,ye2}.pem` (en `git ls-files`) |
| el manifest apunta al config | `…/Platforms/Android/AndroidManifest.xml:9` |
| el script viejo con nombre engañoso | `Utilities/download_sectigo.ps1` (sigue versionado; ver [09](09_CI-CD-y-Build.md)) |

⚠️ `Certificados-SSL/README.md:212` dice que «el config actual del repo **todavía incluye `user`**». **Es falso @285f6fb**: `network_security_config.xml` sólo tiene `system` y los tres `@raw` (líneas 4-6 y 16-21). El párrafo describe un estado anterior.

---

## 6. `Ejemplos_Devices/Docs/otros/`

| Documento | Porqué que documenta | Código que lo encarna |
|---|---|---|
| `AsyncRelayCommandOptions.md` | MVVM **sin** `CommunityToolkit.Mvvm`: `AsyncRelayCommand` propio (el problema de partida: `async void` traga excepciones, l. 30), anti-reentrancia, `IsRunning`, `canExecute`, comparación con el del toolkit (§5) | `GPS/Ejemplo_Maui_GPS/ViewModels/AsyncRelayCommand.cs`, `Phone/Ejemplo_Maui_DirectCall/ViewModels/AsyncRelayCommand.cs` (copias deliberadas) |
| `Eventos.md` | `EventToCommandBehavior` + compiled bindings (`x:DataType`): el Behavior no hereda el `BindingContext` y los eventos del `WebView` no llegan, **sin ninguna excepción** (l. 11). Tres soluciones, y una segunda trampa (`e` llega `null`, §7) | la solución 1 (`Source={x:Reference page}`) y la §7.4 B (`EventArgsConverter`: `NavigatingConverter`/`NavigatedConverter`) están aplicadas en `Integrada/Ejemplo_Maui_Hibrida/Pages/MainPage.xaml:31-37` |
| `GetEnviromentVersion.md` | notas de entorno Windows: 7zip para el SDK Android, `dotnet --version` 10.0.300, comandos para listar SDK/workloads | acompaña a `Ejemplos_Devices/scripts/GetEnviromentVersion.bat` (ver [09](09_CI-CD-y-Build.md)) |
| `propuesta-rosetta.md` | el build de simulador `iossimulator-arm64` falla al enlazar porque **ML Kit no trae slice arm64 de simulador**: device y simulador comparten CPU arm64, pero el Mach-O los distingue por `LC_BUILD_VERSION` y `ld` rechaza mezclarlos (l. 35); el IPA `ios-arm64` nunca estuvo afectado (l. 17, 128). Advierte su «fecha de caducidad»: Apple retira Rosetta 2 (l. 149). Propuesta: RID `iossimulator-x64` + runtime x86_64 + Rosetta 2, en 3 cambios (sólo el 1 arregla el build) | aplicada en los workflows BSM: `.github/workflows/cd-ios-qr.BSM.LectorQR.yml:45` (`iossimulator-x64`); el resto sigue en `arm64` (detalle en [09](09_CI-CD-y-Build.md)) |

⚠️ Hay enlaces rotos por la profundidad de las carpetas:
- `AsyncRelayCommandOptions.md:4-5` enlaza `../GPS/...` y `../Phone/...` desde `Docs/otros/`. Eso resuelve a `Docs/GPS/…`, que no existe; el camino correcto empieza con `../../`.
- `propuesta-rosetta.md:6` cita como origen `Analisis/log.qr.build.ios.md`, que **no existe** ni en este repo ni en el repo doc (en `.github/workflows/Analisis/` sólo hay `log_1.md` y `log_2.md`): **no verificado**.

---

## 7. `Ejemplos_Devices/Docs/qr-nuget/`

Es una evaluación read-only fechada el 2026-06-27 (`README.md:7`): cómo salir de `BarcodeScanner.Mobile.Maui` 9.0.1 (ML Kit también en iOS → workaround Rosetta) hacia APIs nativas de Apple.

| Doc | Contenido |
|---|---|
| `README.md` | hallazgo central + ranking: 🥇 BarcodeScanning.Native.Maui · 🥈 CameraScanner.Maui · 🥉 ZXing.Net.Maui · fork CameraMaui · comerciales |
| `01-analisis-uso-actual.md` | versión y TFMs, registro (`AddBarcodeScannerHandler`), superficie de API, secuencia iOS, permisos, acoplamiento (concentrado en `QRLectorPage`) |
| `02-alternativas-nuget.md` | una ficha por alternativa (2.1–2.5) |
| `03-matriz-rid.md` | matriz por arquitectura (Android arm/x86, iOS device/sim arm64/x64, Mac Catalyst, Windows) |
| `04-recomendacion.md` | criterio doble (nativo iOS + confianza del autor) y ruta de migración «no ejecutada» (§4.4) |
| `05-plan-migracion.md` | diffs por archivo; nombres de API verificados contra un volcado del DLL 3.0.4 (`BarcodeSymbologies`, `BarcodeFormats.QRCode`, `UseBarcodeScanning()`) |

Cómo quedó el código después (verificado en los `.csproj`): la recomendación se materializó **como colección comparativa** y no como una sustitución. Hay un par de proyectos por librería, `QR/{BSM,BSN,CS,ZN}.LectorQR[_Dialog]`, y la híbrida ya usa `BarcodeScanning.Native.Maui` 3.0.4 (`Integrada/Ejemplo_Maui_Hibrida/Ejemplo_Maui_Hibrida.csproj:91`). Detalle en [02](02_QR.md).

⚠️ Hay documentos desfasados respecto del código:
- `04-recomendacion.md:60` y `05-plan-migracion.md:41,237` nombran `Ejemplo_LectorQR_Dialog`, un proyecto que **ya no existe** con ese nombre (hoy son `QR/*.LectorQR_Dialog`). Sólo sobrevive en `.github/workflows/cd-ios-qr.Ejemplo_LectorQR_Dialog.yml:40-41`, que todavía lo apunta (ver [09](09_CI-CD-y-Build.md)); el log de `QR/Ejemplo_Docs_QR/cicd.md` también es de ese proyecto.
- `05-plan-migracion.md` (§5.0) manda quitar `AdamE.Google.iOS.GoogleUtilities`. La híbrida lo **conserva** bajo condición `iossimulator-x64` (`Ejemplo_Maui_Hibrida.csproj:94-96`), aunque ya no use ML Kit en iOS.
- `README.md:16` y `01-analisis-uso-actual.md:28` enlazan `../propuesta-rosetta.md`. El archivo está en `Docs/otros/`, así que el enlace está **roto**.

---

## 8. `Ejemplos_Devices/Docs/web-hibrida/`

Es el corpus que explica **por qué la app híbrida está hecha así**. `maui-hibrido.md` es el documento raíz (§1–§9: panorama, Canal A, contenedor, Canal B, plataformas, diagnóstico iOS, dependencias, archivos). Las cuatro piezas por comando siguen la misma plantilla: panorama → componentes → convención de URL → secuencia (mermaid) → decisiones → variante → observaciones → referencia de archivos.

| Doc | Alcance | Secciones con trampas |
|---|---|---|
| `maui-hibrido.md` | esqueleto (carga remota + circuito Blazor = Canal A), contenedor MAUI, Canal B con el **flujo GPS** en detalle, asimetría Android/iOS (§6, l. 464), diagnóstico de iOS no interactivo (§7, l. 501), dependencias (§8) | §6–§7 |
| `lectura-qr.md` | `qr=qr`: intercepta, abre cámara, inyecta en el DOM | §7 «⚠️ Discrepancia en el cableado actual del callback» (l. 323) |
| `captura-foto.md` | `photo=photo` / `selfie=selfie` + normalización | §8 «⚠️ Observaciones sobre el cableado `param` ↔ DOM» (l. 259) |
| `llamada.md` | `phone=phone` + overlay de estado | §7 Observación (l. 197) |
| `envio-api.md` | `sendAPI=sendAPI` (relay REST con `HttpClient` nativo) y el combo «Llamar y reportar» | §7 (l. 202) |

Las tesis de diseño que **sólo** están escritas acá:
- La web es **remota y opaca**: se usa el `WebView` estándar y no `BlazorWebView` (§8).
- El arreglo de TLS es **sólo de Android**. En iOS no hace falta ni es posible: el fallo de iOS es el circuito SignalR en `WKWebView` (ATS, cookies `SameSite`, *forwarded headers*, hosting free de somee), no el certificado (§7.2–§7.5).

⚠️ `maui-hibrido.md` es **anterior al «Plan 1»** (CHANGELOG 2026-07-23):
- El flujo GPS que describe re-navega a `/panel?Latitud=…&Longitud=…` (l. 349, 382-385, 432). Hoy el botón web «Tomar Coordenadas» va a `/geolocalizacion?coordenadas=coordenadas` (`Integrada/Ejemplo_ws_Blazor/Components/Pages/Panel.razor:174`), y el modo con inyección está en otra tarjeta (`Panel.razor:181`). Para el modelo vigente leé [08](08_App-Hibrida-Integrada.md).
- Las anclas de línea envejecieron. Ejemplo: cita MAUI 10.0.80 en `csproj:105` (l. 632), y hoy está en `Ejemplo_Maui_Hibrida.csproj:111`.
- Enlaza `../certificados-ssl/README.md` (l. 488, 677) en minúscula. La carpeta es `Certificados-SSL`: el enlace se rompe en sistemas que distinguen mayúsculas.

---

## 9. Carpetas `Ejemplo_Docs_*` (una por dominio)

| Carpeta | Contenido real | Índice |
|---|---|---|
| `Camera/Ejemplo_Docs_Photo/` | `Readme.md`: los dos enfoques (diálogo nativo `MediaPicker` vs diálogo propio con `CommunityToolkit.Maui.Camera`, motivado por fallas del nativo en algunos Android) + permisos. `Transferencia_Foto_Entre_Pantallas.md`: guía conceptual (6 técnicas; cómo lo hacen WhatsApp, Instagram, Telegram, Signal y X; downsampling/EXIF) | [01](01_Camara.md) |
| `GPS/Ejemplo_Docs_GPS/` | `Readme.md`: un link a Learn. `secret.md`: 5 opciones para API keys, se eligió la **opción 3** (`Services/ApiKeys.cs` ignorado + `.template` versionado). `servicio.md`: 5 alternativas para que `TestService` use el GPS sin inyectar el VM (se recomienda la 1) | [04](04_GPS.md) |
| `Integrada/Ejemplo_Docs_Integrada/` | `Readme.md` de **0 bytes**: la documentación real de la híbrida está en `Docs/web-hibrida/` | [08](08_App-Hibrida-Integrada.md) |
| `Maps/Ejemplo_Docs_Maps/` | `Readme.md` de **0 bytes** | [05](05_Mapas.md) |
| `Printer/Ejemplo_Docs_Printer/` | portada (`Readme.md`) → `Manuales/` (usuario, técnico ESC/POS, integración ESCPOS_NET, borrador; ~2.900 l. los cuatro, más un `Readme.md` de 223), `Prompts_Generacion_Apk_ejemplo/` (cómo se generó el primer ejemplo con claude.ai + Copilot; incluye el código generado en `ficheros_claude.ia/`, que **no es un proyecto de la solución**), `Prompts_generacion_documentacion/` (ChatGPT → Copilot → manuales), `Borradores/`, `Pruebas_realizadas/` (gif + foto del ticket, Android 13), `Ejemplo_MotorDSL/flowchat.*` | [03](03_Impresion-Termica.md) |
| `QR/Ejemplo_Docs_QR/` | `Readme.md`: setup de `BarcodeScanner.Mobile.Maui` 9.0.1 + `CommunityToolkit.Maui.Camera`, permisos iOS, trucos de compilación iOS (`MtouchExtraArgs`, `NoWarn MT5209…`). `cicd.md`: log del error de linker de ML Kit en `iossimulator-arm64` (el mismo que motiva `propuesta-rosetta.md`) | [02](02_QR.md), [09](09_CI-CD-y-Build.md) |
| `Red/Ejemplo_Docs_Red/` | `Readme.md`: apunte de `Connectivity` / `NetworkAccess` (incluye `ConstrainedInternet` = portal cautivo) | [07](07_Red-Conectividad.md) |

Que `secret.md` está aplicado se verificó: `.gitignore:421` (`**/Services/ApiKeys.cs`) y `GPS/Ejemplo_Maui_GPS/Services/ApiKeys.cs.template` versionado.

⚠️ Hay codificación mixta: `Camera/Ejemplo_Docs_Photo/Readme.md` está en ISO-8859 y `Red/Ejemplo_Docs_Red/Readme.md` en una codificación extendida que no es UTF-8, así que los acentos se ven mal en un visor UTF-8. Lo mismo pasa en `.github/workflows/{Readme,Pipelinea-Version}.md` (ver [09](09_CI-CD-y-Build.md)).

---

## 10. Referencias del código a documentos externos (verificadas)

| Cita en el código | Dónde | ¿Existe? |
|---|---|---|
| `ADR-0001` («cada ejemplo autocontenido y copiable») | `Integrada/Ejemplo_Maui_Hibrida.Tests/Ejemplo_Maui_Hibrida.Tests.csproj:34-35` | ✅ repo doc: `Ejemplos_Maui_Devices-docs/docs/04-decisions/0001-un-proyecto-por-tecnica.md` |
| `ADR-0009` D1 (guard de reentrada), D2 (`InvariantCulture`), «Opción A» (cierre determinista del overlay GPS) | `…Tests.csproj:73,80`; `…Tests/GpsCommandHandlerTests.cs:13`; `…Tests/GpsOverlayTests.cs:20`; `…Tests/NavigatingReentrancyTests.cs:14`; `Ejemplo_Maui_Hibrida/LibApp/Devices/GPS/Services/GpsService.cs:37`; `…/GPS/ViewModels/GpsOverlayViewModel.cs:42`; `…/LibApp/UrlCommands/Handlers/GpsCommandHandler.cs:12,58`; `Ejemplo_Maui_Hibrida/ViewModels/MainViewModel.cs:36,89` | ✅ repo doc: `docs/04-decisions/0009-ciclo-vida-overlay-gps-y-cancelacion-navegacion.md` (único ADR con `status: accepted`; los 0001–0008 están en `draft`) |
| `Ejemplos_Maui_Devices.Documentos/Analisis/Plan-Armonizacion-Overlays.md` §2 | `…Tests/Invariantes.cs:9`; `CHANGELOG.md:161` | ⚠️ **existe con otro nombre de repo**: `Ejemplos_Maui_Devices.Documentacion/Analisis/Plan-Armonizacion-Overlays.md` (su §2 es «Los cinco invariantes»). No hay carpeta `…Documentos` |
| «Plan 1» §3, §4 (ejes A y B), §5, §7 | `LibApp/UrlCommands/CommandDelivery.cs:3,30`; `IUrlCommandHandler.cs:13,25`; `UrlCommandDispatcher.cs:8,36,47,78`; `Handlers/GpsCommandHandler.cs:12,37,77-78`; `…Tests/UrlCommandDispatcherTests.cs:6,9,49,76,80`; `…Tests/GpsCommandHandlerTests.cs:88,124` | ❌ **no verificado**: no hay ningún documento «Plan 1» en el repo doc. Sólo aparece en el título del commit `0ef370b` que cita el `CHANGELOG.md:112` del repo doc. Lo que hace se reconstruye desde `CHANGELOG.md:30-81` |
| `PrintThermal_Motor_Maui.Documentacion/Analisis/Analisis-UX-UI.md` | `CHANGELOG.md:242` | ✅ pero **fuera de este par de repos**: `workspace-dev/Libs/NugetOrg/PrintThermal_Motor_Maui.Documentacion/Analisis/Analisis-UX-UI.md` (documentación de la librería MotorDsl) |
| `Analisis/log.qr.build.ios.md` | `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md:6` | ❌ no se encontró |
| `GDA.APP/APPGDA/Services/NotionsLocationsService.cs:139-150` (origen del patrón `Substitution`) | `Integrada/Ejemplo_Maui_Hibrida/LibApp/UrlCommands/CommandDelivery.cs:25-26`; `…/Handlers/GpsCommandHandler.cs:40,74` (cita `:145-149`) | ❌ repo externo, **no verificado** desde este par de repos |
| `GDA.Core.API.Client` (DTOs `PrintActa/*`) y `GDA.Core.API` (`PrintController.PrintActaById`) | `…/LibApp/Devices/MotorDSL/DTOs/Print/{N,PrintDocument,PrintNode,PrintStyle}.cs`; `Integrada/Ejemplo_ws_Blazor/Controllers/TikectsController.cs:9-10` | ❌ repos externos, **no verificado** |

⚠️ Trampa de lectura con «Tomar Coordenadas»: **los dos botones GPS de `Panel.razor` se llaman igual** (`:18` en la tarjeta «Solicitar coordenadas» y `:30` en «Solicitar GeoPosicion»). El encabezado de `GpsCommandHandler.cs:12-19` asocia «camino web "Tomar Coordenadas"» a `Injection` (cierto para el botón `:30`, que navega con `param=`, `:181`), mientras `CHANGELOG.md:75-76` y `:17-18` llaman «Tomar Coordenadas» a la tarjeta que usa `Substitution` contra `/geolocalizacion` (`:174`). Además, el comentario de `OnSolicitarCoordenadas` (`Panel.razor:172-173`) dice que la app «INYECTA» el resultado, pero esa navegación no lleva `param` y cae en `Substitution`: es un comentario copiado de `OnSolicitarGeoposicion` (`:179-180`). A su vez, ADR-0009 D4 (l. 113) prescribe «entrega por inyección JS (no por re-navegación)», y **ni el corpus `Ejemplos_Maui_Devices-docs/` ni `Analisis/` mencionan `Substitution` ni `CommandDelivery`** (sólo el `CHANGELOG.md` del repo doc y esta ia-db): el ADR no se actualizó tras el Plan 1. Para el comportamiento vigente leé [08](08_App-Hibrida-Integrada.md).

---

## 11. El repo de documentación: qué hay y para qué

Repo doc `Ejemplos_Maui_Devices.Documentacion/` (sin código; su `README.md` propone dos puertas: el corpus narrativo y esta ia-db; ⚠️ su atajo «Las 8 decisiones» de `README.md:26` quedó viejo: hay 9 ADR):

```
Ejemplos_Maui_Devices-docs/   corpus narrativo: README, GLOSSARY, GAP-REPORT, docs-manifest.yaml
  docs/00-overview/           vision.md · system-map.md
  docs/01-architecture/       01-context (C4-L1) · 02-containers (C4-L2) · 04-runtime-views
                              06-crosscutting · 07-overlays-dispositivos · 08-pantallas-por-dispositivo
  docs/04-decisions/          ADR-0001 … ADR-0009 (tabla abajo)
  docs/05-apis/catalog.md     catálogo de APIs y contratos
  docs/07-operations/         build-and-run.md (runbook)
  docs/08-onboarding/         developer-setup.md
  docs/pieces/<dominio>/      camera, gps, integrada (+ bridge-contract), maps, phone, printer, qr, red
Analisis/                     Plan-Armonizacion-Overlays.md (estado «ejecutado», 2026-07-17)
Guias/Guias-Desarrollador/    README.md y Extender-Comando.md: ambos de 0 bytes
PROMPTs/                      tool-prompts (Indexado, Documentaciones, Comportamientos, Guias, Implementaciones)
CHANGELOG.md                  historial del propio repo doc (no del código)
```

| ADR | Título (resumido) | Doc del repo de código que trata lo mismo |
|---|---|---|
| 0001 | un proyecto MAUI mínimo por técnica | — |
| 0002 | servicio tipado + overlay MVVM | `Docs/otros/AsyncRelayCommandOptions.md`, `GPS/Ejemplo_Docs_GPS/servicio.md` |
| 0003 | puente WebView ↔ nativo por comandos en la URL | `Docs/web-hibrida/*` |
| 0004 | QR: BSN (Apple Vision) por sobre BSM (ML Kit) en iOS | `Docs/qr-nuget/*` |
| 0005 | impresión térmica: motor DSL sobre ESC/POS directo | `Printer/Ejemplo_Docs_Printer/Manuales/*` |
| 0006 | CI iOS con firma ad-hoc; Android local en físico | `Docs/otros/propuesta-rosetta.md`, `QR/Ejemplo_Docs_QR/cicd.md` |
| 0007 | secretos fuera del repo (plantilla `ApiKeys` + secrets de CI) | `GPS/Ejemplo_Docs_GPS/secret.md` |
| 0008 | TLS en WebView Android: trust anchors embebidos | `Docs/Certificados-SSL/*` |
| 0009 | cierre determinista del overlay GPS + cancelación robusta del puente | — (sólo comentarios del código, §10) |

La correspondencia se armó leyendo **sólo los títulos**: no se cotejó el contenido de cada ADR contra su documento par.

---

## 12. Discrepancias consolidadas (⚠️)

| # | Dónde | Qué dice | Qué hay |
|---|---|---|---|
| D-1 | `Docs/Certificados-SSL/README.md:212` | el config «todavía incluye `user`» | sin `user` (`network_security_config.xml:4-6,16-21`) |
| D-2 | `Docs/qr-nuget/04-recomendacion.md:60`, `05-plan-migracion.md:41,237` | `Ejemplo_LectorQR_Dialog` | no existe; hoy `QR/{BSM,BSN,CS,ZN}.LectorQR_Dialog` (sólo lo sigue apuntando `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml:40-41`) |
| D-3 | `Docs/qr-nuget/05-plan-migracion.md` §5.0 | quitar `AdamE.Google.iOS.GoogleUtilities` | la híbrida lo conserva condicionado (`Ejemplo_Maui_Hibrida.csproj:94-96`) |
| D-4 | `Docs/qr-nuget/README.md:16`, `01-analisis-uso-actual.md:28` | `../propuesta-rosetta.md` | está en `Docs/otros/` (enlace roto) |
| D-5 | `Docs/otros/AsyncRelayCommandOptions.md:4-5` | `../GPS/…`, `../Phone/…` | falta un nivel (`../../`) |
| D-6 | `Docs/web-hibrida/maui-hibrido.md:349-432` | GPS → `/panel?Latitud=…` | modelo `Substitution`/`Injection` con `/geolocalizacion` (Plan 1) |
| D-7 | `Docs/web-hibrida/maui-hibrido.md:488,677` | `../certificados-ssl/` | carpeta `Certificados-SSL` |
| D-8 | `Ejemplos_Devices.slnx:15-19,34` | `Docs/certificados-ssl/…`; `web-hibrida` vacía | `Docs/Certificados-SSL/`; 5 archivos sin listar |
| D-9 | `GPS/Ejemplo_Docs_GPS/servicio.md:11,32,129` | `MainPageViewModel.cs#L60-L82` con `AplicarResultado` | `AplicarResultado` no existe en `GPS/`; `MainPageViewModel.cs` tiene 116 l. |
| D-10 | `QR/{BSM,BSN,CS,ZN}.LectorQR/Readme.md` | «Librería nuget `BarcodeScanner.Mobile.Maui`» | sólo BSM la usa; BSN → `BarcodeScanning.Native.Maui`, CS → `CameraScanner.Maui` 1.8.31, ZN → `CameraMaui` 1.4.5 (ver [02](02_QR.md)) |
| D-11 | `…Tests/Invariantes.cs:9`, `CHANGELOG.md:161` | repo `Ejemplos_Maui_Devices.Documentos` | se llama `Ejemplos_Maui_Devices.Documentacion` |
| D-12 | código, «Plan 1 §n» | documento citado por sección | no existe documento «Plan 1» |
| D-13 | `GpsCommandHandler.cs:12-19` vs `CHANGELOG.md:17-18,75-76` | «Tomar Coordenadas» = `Injection` (código) / = `Substitution` (CHANGELOG) | ambos botones se rotulan «Tomar Coordenadas» (`Panel.razor:18,30`): uno es `Substitution` (`:174`), el otro `Injection` (`:181`) |
| D-14 | ADR-0009 D4 (repo doc) | GPS sólo por inyección | coexisten `Injection` y `Substitution` (`CHANGELOG.md:41-45`) |
| D-15 | `README.md:7` | «58H6» | modelo 58HB6 |
| D-16 | `Ejemplo_ws_Blazor/Components/Pages/Panel.razor:172-173` | `OnSolicitarCoordenadas` «INYECTA el resultado en #contenidoCoordenada» | navega sin `param` a `/geolocalizacion` (`:174`) → `Substitution`; comentario copiado de `:179-180` |
| D-17 | repo doc: `README.md:26` | «Las 8 decisiones de arquitectura» | hay 9 ADR (`Ejemplos_Maui_Devices-docs/README.md:34`: «8 reconstruidos + 1 de diseño») |

Ninguna de estas discrepancias afecta la compilación. Todas son de prosa: describen un estado anterior del código o tienen enlaces que no resuelven.

---

## 13. Preguntas frecuentes

**¿Dónde leo cómo funciona la app híbrida?** El porqué está en `Ejemplos_Devices/Docs/web-hibrida/maui-hibrido.md` y en los cuatro documentos por comando. El modelo de entrega vigente (`CommandDelivery`) sólo está en `CHANGELOG.md:30-81` y en [08](08_App-Hibrida-Integrada.md).

**¿Por qué el WebView de Android tiene certificados embebidos?** Por ISRG Root X2 en Android viejo: `Docs/Certificados-SSL/README.md` §1, §3 y §4, más ADR-0008 en el repo doc.

**¿Por qué algunos workflows de QR compilan el simulador en `x64`?** Porque ML Kit no trae slice arm64 de simulador: `Docs/otros/propuesta-rosetta.md` y `QR/Ejemplo_Docs_QR/cicd.md`. La salida de fondo es migrar de librería (`Docs/qr-nuget/`).

**¿Qué versión tiene el repo?** No tiene número de versión: el `CHANGELOG.md` es por fecha (§4).

**¿Los ADR viven en este repo?** No. Viven en el repo doc (`Ejemplos_Maui_Devices-docs/docs/04-decisions/`) y el código los cita por número (§10).

**¿Puedo seguir los line numbers que citan los `.md` de `Docs/`?** Con cuidado. Varios son anteriores a la reorganización a `LibApp/` y al Plan 1 (D-6, D-9). Verificá contra el código o contra el índice del dominio.

**¿`Ejemplo_Docs_Integrada` y `Ejemplo_Docs_Maps` tienen algo?** No: son archivos de 0 bytes. Es material «en construcción» deliberado.
