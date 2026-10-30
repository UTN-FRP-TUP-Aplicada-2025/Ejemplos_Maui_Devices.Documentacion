# 10 — Documentación transversal

> **Propósito:** mapa de toda la prosa disponible —la que vive junto al código y la del repo de documentación— para saber **qué documento responde cada porqué** sin releerla. Una línea por documento; no se resume su contenido.
> **Fuentes primarias:** repo de código (`README.md`, `CHANGELOG.md`, `Ejemplos_Devices/Docs/`, `Ejemplos_Devices/*/Ejemplo_Docs_*/`), HEAD `cc80ce1`; repo de documentación (`Ejemplos_Maui_Devices-docs/`, `Analisis/`, `Guias/`, `CHANGELOG.md`, `PROMPTs/`).
> Convención: la prosa del repo de documentación se enlaza relativa a este índice (`../../…`); la del código se cita por ruta desde la raíz del repo de código.

---

## 1. Por qué pregunta → qué documento

| Pregunta de porqué | Documento |
|---|---|
| ¿Por qué hay un proyecto por técnica y no una app que lo haga todo? | [ADR-0001](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0001-un-proyecto-por-tecnica.md) · [vision](../../Ejemplos_Maui_Devices-docs/docs/00-overview/vision.md) |
| ¿Por qué cada dispositivo es servicio tipado + overlay MVVM? | [ADR-0002](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0002-servicio-tipado-overlay-mvvm.md) · [07-overlays-dispositivos](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/07-overlays-dispositivos.md) |
| ¿Qué ve el usuario en cada estado de un overlay? | [08-pantallas-por-dispositivo](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/08-pantallas-por-dispositivo.md) |
| ¿Cómo se llegó a los cinco invariantes de los overlays? | [Plan-Armonizacion-Overlays](../../Analisis/Plan-Armonizacion-Overlays.md) |
| ¿Por qué la web habla con lo nativo navegando a URLs? | [ADR-0003](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0003-puente-webview-comandos-url.md) · [bridge-contract](../../Ejemplos_Maui_Devices-docs/docs/pieces/integrada/bridge-contract.md) |
| ¿Por qué se cancela siempre la navegación y se cierra así el overlay GPS? | [ADR-0009](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0009-ciclo-vida-overlay-gps-y-cancelacion-navegacion.md) |
| ¿Cómo funciona cada comando del puente (foto, QR, llamada, relay REST)? | `Ejemplos_Devices/Docs/web-hibrida/{captura-foto,lectura-qr,llamada,envio-api}.md` |
| ¿Por qué Blazor Interactive Server remoto dentro del WebView y qué problemas tiene en iOS? | `Ejemplos_Devices/Docs/web-hibrida/maui-hibrido.md` · [pieza integrada](../../Ejemplos_Maui_Devices-docs/docs/pieces/integrada/README.md) |
| ¿Por qué BSN y no BSM para QR en iOS? | [ADR-0004](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0004-qr-bsn-recomendada-ios.md) · `Ejemplos_Devices/Docs/qr-nuget/04-recomendacion.md` |
| ¿Por qué el simulador de BSM compila en x64 con Rosetta? | `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md` · `Ejemplos_Devices/Docs/qr-nuget/03-matriz-rid.md` |
| ¿Por qué imprimir con el motor DSL y no ESC/POS directo? | [ADR-0005](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0005-impresion-motor-dsl.md) · [pieza printer](../../Ejemplos_Maui_Devices-docs/docs/pieces/printer/README.md) |
| ¿Por qué iOS solo en CI y Android solo local? | [ADR-0006](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0006-ci-ios-adhoc-android-local.md) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) |
| ¿Por qué la clave de Google Maps no está en el repo? | [ADR-0007](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0007-secretos-fuera-del-repo.md) · `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/secret.md` |
| ¿Por qué la app embebe certificados para el WebView Android? | [ADR-0008](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0008-trust-anchors-embebidos-webview.md) · `Ejemplos_Devices/Docs/Certificados-SSL/README.md` |
| ¿Por qué `AsyncRelayCommand` y no handlers async sueltos? | `Ejemplos_Devices/Docs/otros/AsyncRelayCommandOptions.md` |
| ¿Por qué `EventToCommandBehavior` sobre el `WebView`? | `Ejemplos_Devices/Docs/otros/Eventos.md` |
| ¿Cómo pasar una foto recién tomada a otra pantalla? | `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` |
| ¿Cómo hace el servicio GPS para que el overlay reaccione? | `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/servicio.md` |
| ¿Qué permisos y convenciones MVVM son transversales? | [06-crosscutting](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/06-crosscutting.md) |
| ¿Qué falta documentar o está desactualizado? | [GAP-REPORT](../../Ejemplos_Maui_Devices-docs/GAP-REPORT.md) · [docs-manifest](../../Ejemplos_Maui_Devices-docs/docs-manifest.yaml) |
| ¿Qué cambió y cuándo (código / documentación)? | `CHANGELOG.md` · [CHANGELOG de la documentación](../../CHANGELOG.md) |

## 2. ADR

Todos en `Ejemplos_Maui_Devices-docs/docs/04-decisions/`; estado en la línea 6 del frontmatter; decisión en la sección «Decisión» (`:25`, salvo ADR-0009 `:39`).

| N.º | Título | Estado | Decisión en una línea | Cita |
|---|---|---|---|---|
| 0001 | Un proyecto MAUI mínimo por técnica de integración | draft | cada técnica en su propio proyecto; la híbrida es la única integradora | [0001](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0001-un-proyecto-por-tecnica.md) |
| 0002 | Patrón por dispositivo — servicio tipado + overlay MVVM | draft | interfaz `I*Service` en DI + overlay con ViewModel que maneja permisos, progreso y resultado | [0002](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0002-servicio-tipado-overlay-mvvm.md) |
| 0003 | Puente WebView ↔ nativo por comandos en la URL | draft | la web navega a URLs convencionales; el primer `IUrlCommandHandler` que coincide despacha | [0003](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0003-puente-webview-comandos-url.md) |
| 0004 | QR — BSN (Apple Vision) recomendada frente a BSM (ML Kit) en iOS | draft | BSN recomendada; BSM queda como comparativo con CI adaptado | [0004](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0004-qr-bsn-recomendada-ios.md) |
| 0005 | Impresión térmica — motor DSL (JSON → raster) sobre ESC/POS directo | draft | `MotorDsl.*` compone, rasteriza y el servicio Bluetooth transporta; ThermalPrinter es línea base | [0005](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0005-impresion-motor-dsl.md) |
| 0006 | CI de iOS con firma ad-hoc en macOS; ejecución local en Android físico | draft | Android local por `.bat`; iOS solo en CI `macos-15` con firma ad-hoc y simulador | [0006](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0006-ci-ios-adhoc-android-local.md) |
| 0007 | Secretos de build fuera del repositorio | draft | `ApiKeys.cs` gitignoreado, generado desde plantilla (local) o desde secrets (CI) | [0007](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0007-secretos-fuera-del-repo.md) |
| 0008 | TLS en WebView Android — trust anchors embebidos | draft | raíz + dos intermedios en `res/raw` declarados por dominio en `network_security_config.xml` | [0008](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0008-trust-anchors-embebidos-webview.md) |
| 0009 | Cierre determinista del overlay GPS y cancelación robusta del puente | **accepted** | siempre `e.Cancel = true` ante URL-comando + guard de reentrada + cierre determinista del overlay | [0009](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0009-ciclo-vida-overlay-gps-y-cancelacion-navegacion.md) |

## 3. Prosa del repo de código

| Ruta | Líneas | Qué responde |
|---|---|---|
| `README.md` | 24 | portada: enlaces a los `Ejemplo_Docs_*` de Printer, QR, Cámara, Red, GPS y Maps (`:5-23`) |
| `CHANGELOG.md` | 371 | historia de cambios de código por fecha (última entrada `[2026-07-23]`, `:6`) |
| `Ejemplos_Devices/Docs/Certificados-SSL/README.md` | 353 | resolver «Trust anchor for certification path not found» en Android |
| `Ejemplos_Devices/Docs/Certificados-SSL/Anexo-A-Comandos.md` | 223 | comandos de inspección TLS |
| `Ejemplos_Devices/Docs/Certificados-SSL/Anexo-B-Glosario.md` | 48 | glosario PKI/TLS |
| `Ejemplos_Devices/Docs/otros/AsyncRelayCommandOptions.md` | 412 | `AsyncRelayCommand` y sus opciones |
| `Ejemplos_Devices/Docs/otros/Eventos.md` | 312 | eventos del `WebView` + `EventToCommandBehavior` |
| `Ejemplos_Devices/Docs/otros/GetEnviromentVersion.md` | 115 | instalar y verificar SDK/workloads la primera vez |
| `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md` | 192 | compilar el simulador iOS en x86_64 (Rosetta) |
| `Ejemplos_Devices/Docs/qr-nuget/README.md` | 52 | entrada a la evaluación de NuGets de QR |
| `Ejemplos_Devices/Docs/qr-nuget/01-analisis-uso-actual.md` … `05-plan-migracion.md` | 158/174/47/77/258 | uso actual de BSM, alternativas, matriz de RID, recomendación, migración BSM→BSN |
| `Ejemplos_Devices/Docs/web-hibrida/maui-hibrido.md` | 677 | arquitectura de la híbrida con Blazor Interactive Server remoto |
| `Ejemplos_Devices/Docs/web-hibrida/captura-foto.md` | 326 | comando de foto/selfie |
| `Ejemplos_Devices/Docs/web-hibrida/lectura-qr.md` | 393 | comando de QR |
| `Ejemplos_Devices/Docs/web-hibrida/llamada.md` | 222 | comando de llamada |
| `Ejemplos_Devices/Docs/web-hibrida/envio-api.md` | 244 | relay REST (`sendAPI`) |
| `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/Readme.md` | 81 | notas de los ejemplos de cámara |
| `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/Transferencia_Foto_Entre_Pantallas.md` | 762 | pasar la foto entre pantallas |
| `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/Readme.md` | 2 | solo un enlace a Microsoft Learn (geolocalización) |
| `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/secret.md` | 115 | manejo de API keys en MAUI |
| `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/servicio.md` | 129 | alternativas para que el servicio GPS alimente al overlay |
| `Ejemplos_Devices/Integrada/Ejemplo_Docs_Integrada/Readme.md` | 0 | vacío |
| `Ejemplos_Devices/Maps/Ejemplo_Docs_Maps/Readme.md` | 0 | vacío (el enlace del `README.md` raíz llega aquí) |
| `Ejemplos_Devices/Maps/Ejemplo_Maui_Mapas/Readme.md` | 4 | enlace a Microsoft Learn (control Map) |
| `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/Readme.md` | 21 | índice de la documentación de impresión (manuales, prompts, pruebas; 15 `.md` en la carpeta) |
| `Ejemplos_Devices/QR/Ejemplo_Docs_QR/Readme.md` | 115 | uso XAML de los lectores |
| `Ejemplos_Devices/QR/Ejemplo_Docs_QR/cicd.md` | 263 | errores de compilación del CI de QR |
| `Ejemplos_Devices/Red/Ejemplo_Docs_Red/Readme.md` | 64 | notas de conectividad |
| `.github/workflows/Readme.md`, `Pipelinea-Version.md`, `Analisis/log_{1,2}.md` | 53/14/2110/102 | entorno local de referencia, bitácora del yml y logs de CI → [09_CI-CD-y-Build](09_CI-CD-y-Build.md) |

## 4. Prosa del repo de documentación

### 4.1 `Ejemplos_Maui_Devices-docs/` (todo `status: draft`)

| Documento | Responde |
|---|---|
| [README](../../Ejemplos_Maui_Devices-docs/README.md) | índice maestro del corpus |
| [docs-manifest.yaml](../../Ejemplos_Maui_Devices-docs/docs-manifest.yaml) | qué documentos debería haber por pieza |
| [GAP-REPORT](../../Ejemplos_Maui_Devices-docs/GAP-REPORT.md) | qué falta, derivas, bugs latentes, hallazgo de seguridad S-01 |
| [GLOSSARY](../../Ejemplos_Maui_Devices-docs/GLOSSARY.md) | vocabulario controlado |
| [vision](../../Ejemplos_Maui_Devices-docs/docs/00-overview/vision.md) | visión y alcance |
| [system-map](../../Ejemplos_Maui_Devices-docs/docs/00-overview/system-map.md) | inventario de piezas |
| [01-context](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/01-context.md) | C4-L1 |
| [02-containers](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/02-containers.md) | C4-L2 |
| [04-runtime-views](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/04-runtime-views.md) | flujos de punta a punta |
| [06-crosscutting](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/06-crosscutting.md) | seguridad, permisos, convenciones MVVM |
| [07-overlays-dispositivos](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/07-overlays-dispositivos.md) | fundamento del patrón overlay |
| [08-pantallas-por-dispositivo](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/08-pantallas-por-dispositivo.md) | catálogo de pantallas por estado |
| [ADR 0001–0009](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/) | §2 |
| [catalog](../../Ejemplos_Maui_Devices-docs/docs/05-apis/catalog.md) | APIs y contratos (backend Blazor) |
| [build-and-run](../../Ejemplos_Maui_Devices-docs/docs/07-operations/build-and-run.md) | runbook local + CI |
| [developer-setup](../../Ejemplos_Maui_Devices-docs/docs/08-onboarding/developer-setup.md) | primer día del desarrollador |
| [pieces/camera](../../Ejemplos_Maui_Devices-docs/docs/pieces/camera/README.md) | 5 variantes de captura comparadas |
| [pieces/gps](../../Ejemplos_Maui_Devices-docs/docs/pieces/gps/README.md) | geolocalización y geocodificación inversa |
| [pieces/integrada](../../Ejemplos_Maui_Devices-docs/docs/pieces/integrada/README.md) | app híbrida + backend Blazor |
| [pieces/integrada/bridge-contract](../../Ejemplos_Maui_Devices-docs/docs/pieces/integrada/bridge-contract.md) | contrato de comandos y endpoints |
| [pieces/maps](../../Ejemplos_Maui_Devices-docs/docs/pieces/maps/README.md) | mapa nativo con pines |
| [pieces/phone](../../Ejemplos_Maui_Devices-docs/docs/pieces/phone/README.md) | marcador vs llamada directa |
| [pieces/printer](../../Ejemplos_Maui_Devices-docs/docs/pieces/printer/README.md) | impresión térmica Bluetooth 58 mm |
| [pieces/qr](../../Ejemplos_Maui_Devices-docs/docs/pieces/qr/README.md) | 4 librerías × 2 UIs |
| [pieces/red](../../Ejemplos_Maui_Devices-docs/docs/pieces/red/README.md) | estado y cambios de conectividad |

### 4.2 Resto del repo de documentación

| Ruta | Responde |
|---|---|
| [README](../../README.md) | puertas de entrada (docs para el porqué, ia-db para el dónde) |
| [CHANGELOG](../../CHANGELOG.md) | historia de la documentación y de la ia-db |
| [Analisis/Plan-Armonizacion-Overlays](../../Analisis/Plan-Armonizacion-Overlays.md) | plan (ejecutado, 2026-07-17) para llevar los cuatro overlays de la híbrida a los mismos invariantes |
| `Guias/Guias-Desarrollador/README.md`, `Extender-Comando.md` | **vacíos** (0 líneas) |

### 4.3 `PROMPTs/` (solo listado: tool-prompts, no documentación)

| Carpeta | Archivos |
|---|---|
| `PROMPTs/Comportamientos/` | `Analisis-Comportamiento-Software-01.md`, `-02.md` (+ carpeta con 7 casos y una imagen), `-03.md` (+ `Primera`/`Segunda`/`Tercera.md`), `Analizar-Fundanalidad-Comportamiento-Libreria-Nuget-PrintTermal.md`, `…-Aplicacion.md` |
| `PROMPTs/Documentaciones/` | `Documentacion-Actualizar-Ejemplos_Maui_Devices.md`, `Indexado-Actualizar.md` |
| `PROMPTs/Guias/` | `Crear-Extender-Comando.md` |
| `PROMPTs/Implementaciones/` | `Analizar-Implementacion-Print-PoC-En-Proyecto.md`, `Crear-Flow-End2End-App.md`, `Crear-Flow-end2end-Ejemplo_Maui_Hibrida.md`, `Crear-Prompt-Analizar-Implementacion-End2End.md` |
| `PROMPTs/Indexado/` | `Crear-Indexado.md`, `Actualizar-Indexado.md` |

## 5. Huecos y derivas de la prosa (verificados a HEAD)

| Hecho | Cita |
|---|---|
| El README del repo de documentación habla de «las 8 decisiones de arquitectura»; hay 9 ADR | `README.md` del repo de documentación (tabla «Atajos») |
| `Guias/` existe pero sus dos archivos están vacíos, aunque hay un tool-prompt para generar `Extender-Comando` | `Guias/Guias-Desarrollador/*.md` · `PROMPTs/Guias/Crear-Extender-Comando.md` |
| `Docs/web-hibrida/` tiene 5 documentos pero el `.slnx` muestra la carpeta vacía; dos entradas `File` del `.slnx` apuntan a rutas inexistentes | `Ejemplos_Devices/Ejemplos_Devices.slnx:34,62,111` (detalle en [09 §9](09_CI-CD-y-Build.md)) |
| El `README.md` raíz no enlaza Integrada, Phone ni `Docs/`; su enlace de Maps lleva a un Readme vacío | `README.md:5-23` · `Ejemplos_Devices/Maps/Ejemplo_Docs_Maps/Readme.md` |

## Ver también

- [00_MASTER-INDEX](00_MASTER-INDEX.md) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) · [02_QR](02_QR.md)
- [README de la documentación](../../Ejemplos_Maui_Devices-docs/README.md) · [GAP-REPORT](../../Ejemplos_Maui_Devices-docs/GAP-REPORT.md)
