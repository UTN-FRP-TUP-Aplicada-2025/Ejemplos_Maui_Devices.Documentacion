# 10 — Documentación transversal

> **Propósito:** los documentos que no pertenecen a un solo dominio — la arquitectura web↔híbrida, los certificados SSL, la propuesta Rosetta, los apuntes de MVVM/eventos y el CHANGELOG.
> **Fuente primaria:** `Ejemplos_Devices/Docs/` (18 documentos, ~4.300 líneas), `CHANGELOG.md`, `README.md`, `.gitignore`.
> **Índices relacionados:** todos — este índice es el mapa de la prosa; el detalle técnico está en los índices [01](01_Camara.md)–[09](09_CI-CD-y-Build.md).

---

## 1. Mapa de `Ejemplos_Devices/Docs/`

| Carpeta | Documentos | Tema | Índice que lo cubre |
|---------|-----------:|------|---------------------|
| `web-hibrida/` | 5 · 1.850 l. | La arquitectura de la app híbrida, comando por comando | [08](08_App-Hibrida-Integrada.md) |
| `qr-nuget/` | 6 · 766 l. | Evaluación de librerías de escaneo | [02](02_QR.md) |
| `Certificados-SSL/` | 3 · 624 l. | `Trust anchor for certification path not found` en el WebView de Android | este índice §3 |
| `otros/` | 4 · 1.031 l. | `AsyncRelayCommand`, eventos + behaviors, Rosetta, entorno | este índice §4 y §5 |

## 2. `web-hibrida/` — la arquitectura documentada

Cinco documentos que describen la **secuencia de llamadas y la lógica** de cada comando del puente. Todos remiten a `maui-hibrido.md` como documento raíz y llaman **«Canal B»** al puente nativo por URL.

| Documento | Líneas | Alcance |
|-----------|-------:|---------|
| `maui-hibrido.md` | 677 | El principio de funcionamiento: MAUI hospedando una web **Blazor Interactive Server remota** (`https://aplicada.somee.com`), cómo se consigue la interactividad (el **circuito SignalR**), el puente por URL con el flujo GPS en detalle, y **el diagnóstico del fallo de iOS** (la página se ve pero los botones no responden) |
| `lectura-qr.md` | 393 | El comando `qr=qr` |
| `captura-foto.md` | 326 | Los comandos `photo=photo` y `selfie=selfie`: cámara nativa, normalización y devolución al DOM sin recargar |
| `envio-api.md` | 244 | El comando `sendAPI=sendAPI`: relay REST con `HttpClient` nativo |
| `llamada.md` | 222 | El comando `phone=phone` y su overlay de estado |

Los cuatro documentos de comando comparten la misma estructura: panorama del rol dentro del Canal B → secuencia de llamadas → lógica asociada → contraste con los otros comandos.

⚠️ La solución `Ejemplos_Devices.slnx` declara la carpeta `/Docs/web-hibrida/` **vacía**, sin enlazar ninguno de los 5 documentos ([índice 09 §7](09_CI-CD-y-Build.md)).

## 3. `Certificados-SSL/` — el caso `Trust anchor`

Guía didáctica en tres piezas (`README.md` 353 l., `Anexo-A-Comandos.md` 223 l., `Anexo-B-Glosario.md` 48 l.) sobre un problema real y reproducible.

| | |
|---|---|
| **Síntoma** | El WebView (Android System WebView = Chromium) no abre `https://aplicada.somee.com`. En el log: `cr_X509Util … CertPathValidatorException: Trust anchor for certification path not found` |
| **Caso de estudio** | Moto g42 (Android 12/13) |
| **Causa raíz** | La raíz de la cadena es **`ISRG Root X2`** (Let's Encrypt / ISRG), una raíz **ECDSA de 2020** que no está en el almacén de confianza de Android en dispositivos viejos: el validador no puede cerrar la cadena |
| **No es** | **No es Sectigo** — «esa fue una suposición equivocada del setup original (incluso el script se llamaba `download_sectigo.ps1`)» |
| **Solución** | Embeber la raíz y los dos intermedios como `.pem` en `res/raw` y declararlos como `trust-anchors` para ese dominio en `network_security_config.xml` |
| **Archivos** | `Ejemplo_Maui_Hibrida/Platforms/Android/Resources/xml/network_security_config.xml`, `…/raw/*.pem`, `AndroidManifest.xml` |

El `Anexo-A` trae los comandos de inspección con salidas de ejemplo (`openssl`, PowerShell + `X509Chain`, `adb`); el `Anexo-B`, el glosario PKI/TLS.

> El script `Utilities/download_sectigo.ps1` **conserva el nombre equivocado** que el propio documento señala. Es la huella de la hipótesis descartada.

## 4. `otros/propuesta-rosetta.md` — la propuesta que el repo terminó ejecutando

192 líneas. Estado declarado: «Propuesta para reevaluar en el workspace de pruebas de pipelines». Ámbito: **solo YAML del workflow**, sin tocar código ni NuGets.

El job de iOS construye **dos binarios distintos** y solo uno falla:

| Build | RID | Propósito | ¿Falla? |
|-------|-----|-----------|---------|
| Simulador | `iossimulator-arm64` | Smoke test + GIF de evidencia | ❌ **Sí** |
| Release / IPA | `ios-arm64` | Entregable para iPhone físico | ✅ No |

**La causa raíz, explicada en términos de Mach-O:** el runner es Apple Silicon, así que el RID del simulador es `iossimulator-arm64`. `BarcodeScanner.Mobile.Maui` 9.0.1 usa el SDK nativo de **Google ML Kit**, que publica `arm64` **solo para device** y `x86_64` para simulador. En Apple Silicon device y simulador comparten CPU arm64, pero el Mach-O los distingue por el flag `LC_BUILD_VERSION` (`PLATFORM_IOS` vs `PLATFORM_IOSSIMULATOR`), y `ld` rechaza mezclarlos.

Es el documento fundacional de dos decisiones que sí se ejecutaron: la variante `iossimulator-x64` + Rosetta de los dos workflows de BSM ([índice 09 §4.3](09_CI-CD-y-Build.md)) y, después, la evaluación de alternativas que buscó **eliminar** esa dependencia ([índice 02 §2](02_QR.md)). El estudio de NuGets lo cita explícitamente como la solución con fecha de caducidad, porque Apple está retirando Rosetta.

## 5. `otros/` — los apuntes de técnica

| Documento | Líneas | Contenido |
|-----------|-------:|-----------|
| `AsyncRelayCommandOptions.md` | 412 | Qué resuelve `AsyncRelayCommand`: `ICommand.Execute` devuelve `void`, así que un `async void` deja excepciones sin capturar y no permite deshabilitar el botón mientras corre. Documenta la **implementación didáctica sin dependencias** que aparece duplicada en `GPS/Ejemplo_Maui_GPS/ViewModels/` y `Phone/Ejemplo_Maui_DirectCall/ViewModels/`, «pensada para mostrar MVVM sin incorporar `CommunityToolkit.Mvvm`» |
| `Eventos.md` | 312 | **El problema del `EventToCommandBehavior` con compiled bindings**: la app inicia, el `WebView` muestra la URL, pero `Navigating`/`Navigated` **nunca llegan al ViewModel y no hay ninguna excepción**. La causa es la interacción de `x:DataType` con los `Behavior`, que no heredan `BindingContext` — el mismo mecanismo que documenta `WebViewBridgeBehavior` en su código ([índice 08 §5](08_App-Hibrida-Integrada.md)) |
| `GetEnviromentVersion.md` | 115 | Instalación del entorno la primera vez (7zip requerido por el SDK de Android, NuGet, …); acompaña a `scripts/GetEnviromentVersion.bat` |
| `propuesta-rosetta.md` | 192 | §4 |

Los dos primeros son **el porqué de código que está duplicado a propósito**: `AsyncRelayCommand` en dos proyectos y la mecánica de behaviors en la híbrida.

## 6. Documentos de raíz

### `README.md`

25 líneas. Portada con seis entradas que apuntan a la carpeta `Ejemplo_Docs_*` de cada dominio: Thermal Printer (con el GIF `ejemplo_print_thermal.gif`), Lectura de QR, Cámara, Red y Conectividad, GPS y Maps. Las cinco últimas dicen **«En construcción …»** — lo que refleja el estado de esas carpetas `Ejemplo_Docs_*`, no el de los ejemplos, que sí están desarrollados.

### `CHANGELOG.md`

Basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) 1.1.0, en español. **9 entradas**, de `2026-07-13` a `2026-07-23` (la más reciente).

Convenciones observables:
- Encabezado `## [AAAA-MM-DD] — <título temático>`, no versiones semánticas.
- Una línea `Alcance:` inmediatamente debajo del título, con las rutas afectadas.
- Subsecciones estándar (`### Agregado`, `### Cambiado`, `### Corregido`, `### Eliminado`) más dos propias: **`### Notas`** y **`### Activado`** (esta última se usó para activar el `push` del CI de la híbrida).
- Las entradas explican **el defecto que motivó el cambio**, no solo el cambio: p. ej. la del Plan 1 enumera los dos defectos del modelo anterior del puente.

Recorrido temático de las 9 entradas: impresión térmica y reorganización a `LibApp/` → fix de navegación del WebView y permisos Bluetooth → UX de impresión con códigos de error → armonización de overlays y primer proyecto de tests → tres entradas de CI/end2end → puente de comandos (Plan 1) → panel con ambos modos de GPS y namespaces `LibApp`.

### `.gitignore`

421 líneas: el `.gitignore` estándar de Visual Studio más una sección propia al final — **«API keys locales (no subir al repo)»** con `**/Services/ApiKeys.cs`, que es lo que hace que el proyecto de GPS necesite crear ese archivo desde su `.template` ([índice 04 §7](04_GPS.md)).

## 7. Documentación fuera de este repositorio

El código referencia documentos que **no viven en el repositorio de código** sino en `Ejemplos_Maui_Devices.Documentacion` (donde está esta ia-db):

| Referencia en el código | Dónde |
|-------------------------|-------|
| `Analisis/Plan-Armonizacion-Overlays.md` §2 | `Ejemplo_Maui_Hibrida.Tests/Invariantes.cs:9` |
| **ADR-0001** (cada ejemplo autocontenido y copiable) | `Ejemplo_Maui_Hibrida.Tests.csproj` |
| **ADR-0009** (guard de reentrada, cultura de coordenadas) | `MainViewModel.cs`, `GpsCommandHandler.cs`, `Ejemplo_Maui_Hibrida.Tests.csproj` |
| **Plan 1** §3/§4/§5 (ejes A y B, invariante de continuación) | todo `LibApp/UrlCommands/` |

Al leer esos comentarios hay que buscar el documento en el repositorio de documentación, no en el de código.

## 8. Observaciones

- **La documentación por dominio está desbalanceada.** `Ejemplo_Docs_Printer/` tiene ~3.100 líneas de manuales y `Docs/web-hibrida/` ~1.850; en el otro extremo, `GPS/Ejemplo_Docs_GPS/Readme.md` es **un link** y `Maps/Ejemplo_Docs_Maps/Readme.md` está **vacío**.
- Los documentos de análisis (`qr-nuget/`, `propuesta-rosetta.md`) declaran su fecha y su estado («evaluación read-only», «propuesta para reevaluar»), y varios ya fueron superados por los hechos: la propuesta Rosetta se ejecutó y luego se buscó revertirla; el estudio de NuGets nombra un proyecto (`Ejemplo_LectorQR_Dialog`) que ya no existe.
- El repositorio conserva deliberadamente **el rastro del error**: el nombre de `download_sectigo.ps1`, los bloques de código comentados con la versión anterior, los workflows del proyecto retirado. Es material didáctico, no descuido.

## 9. Fuentes

| Ruta | Contenido |
|------|-----------|
| `Docs/web-hibrida/maui-hibrido.md` | Principio de funcionamiento del híbrido, circuito SignalR, Canal B, diagnóstico de iOS |
| `Docs/web-hibrida/{captura-foto,lectura-qr,llamada,envio-api}.md` | Secuencia y lógica de cada comando |
| `Docs/Certificados-SSL/` | El caso `Trust anchor`, comandos de inspección y glosario PKI/TLS |
| `Docs/otros/propuesta-rosetta.md` | La causa raíz Mach-O del fallo de MLKit y la propuesta x64/Rosetta |
| `Docs/otros/{AsyncRelayCommandOptions,Eventos}.md` | MVVM sin toolkit y el problema de behaviors con compiled bindings |
| `CHANGELOG.md` | 9 entradas fechadas, de 2026-07-13 a 2026-07-23 |
| `README.md`, `.gitignore` | Portada del repositorio y la sección de API keys locales |
