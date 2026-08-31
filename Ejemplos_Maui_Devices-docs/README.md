---
doc_id: DOCS-README-001
doc_type: index
title: Ejemplos_Maui_Devices — índice maestro de documentación
version: 1.0.0
status: draft
origin: reverse-engineered
confidence: high
owner: fernandofilipuzzi
last_review: 2026-07-15
review_cycle_days: 180
audience: [dev, arquitectos, qa, agentes-automaticos]
classification: uso-interno
traces: []
---

# Ejemplos_Maui_Devices — documentación

> **Resumen ejecutivo.** Conjunto documental de `Ejemplos_Maui_Devices`, una colección de **ejemplos didácticos de acceso a dispositivos desde .NET MAUI** (cámara, QR, impresión térmica, GPS, mapas, telefonía, red) más una **app híbrida** que los integra tras un WebView con backend Blazor. Generado conforme al Marco de Documentación de Software: navegable por humanos desde este índice y operable por agentes desde [`docs-manifest.yaml`](docs-manifest.yaml) + el frontmatter de cada documento. Todo el corpus está en `status: draft`.

## Cómo está organizado

```
Ejemplos_Maui_Devices.Documentacion/
├── ia-db/                     ← índices navegables del código (Fase 1; punto de entrada para agentes)
└── Ejemplos_Maui_Devices-docs/  ← este conjunto documental
    ├── README.md              ← este índice maestro
    ├── GLOSSARY.md            ← vocabulario controlado
    ├── docs-manifest.yaml     ← manifiesto máquina-legible (piezas, types, gaps)
    ├── GAP-REPORT.md          ← qué falta, qué se omitió y las derivas detectadas
    └── docs/
        ├── 00-overview/       ← visión + mapa del sistema
        ├── 01-architecture/   ← C4 contexto/contenedores, runtime views, transversales, overlays
        ├── 04-decisions/      ← 9 ADRs (8 reconstruidos + 1 de diseño)
        ├── 05-apis/           ← catálogo de contratos (puente URL + backend)
        ├── 07-operations/     ← runbook de build y ejecución
        ├── 08-onboarding/     ← setup del desarrollador
        └── pieces/            ← doc por dominio de dispositivo
```

## Navegación — «Necesito… → voy a…»

| Necesito… | Voy a… |
|---|---|
| Entender qué es la solución y para quién | [Visión](docs/00-overview/vision.md) |
| Ver todas las piezas y su tipo | [Mapa del sistema](docs/00-overview/system-map.md) |
| Ver cómo encaja todo y por qué | [Contexto C4](docs/01-architecture/01-context.md) → [Contenedores](docs/01-architecture/02-containers.md) |
| Seguir un flujo de punta a punta (imprimir, leer QR, GPS) | [Vistas de runtime](docs/01-architecture/04-runtime-views.md) |
| **Entender el patrón de overlays de dispositivo** (fundamento, cómo agregar uno, anti-patrones) | [Overlays — fundamento](docs/01-architecture/07-overlays-dispositivos.md) |
| **Saber qué ve el usuario ante un fallo** de GPS, red, llamada o impresión | [Overlays — catálogo de pantallas](docs/01-architecture/08-pantallas-por-dispositivo.md) |
| Entender el porqué de una decisión | [ADRs](docs/04-decisions/) |
| Integrar la web con lo nativo | [Catálogo de APIs](docs/05-apis/catalog.md) · [Contrato del puente](docs/pieces/integrada/bridge-contract.md) |
| Compilar y ejecutar los ejemplos | [Runbook](docs/07-operations/build-and-run.md) |
| Arrancar como dev nuevo | [Setup del desarrollador](docs/08-onboarding/developer-setup.md) |
| Saber qué falta o qué revisar | [GAP-REPORT](GAP-REPORT.md) |
| Buscar en el código sin leerlo todo | [ia-db](../ia-db/README.md) |

## Piezas por dominio

| Dominio | Qué demuestra | Documento |
|---|---|---|
| Cámara | Captura de foto: 5 variantes (MediaPicker vs. picker propio, callback vs. Task, normalización EXIF, selfie) | [pieces/camera](docs/pieces/camera/README.md) |
| QR | Escaneo: 4 librerías (BSM·BSN·CS·ZN) × 2 UIs (página/diálogo); recomendación fundada | [pieces/qr](docs/pieces/qr/README.md) |
| Impresión térmica | BT 58 mm: ESC/POS directo vs. motor DSL (JSON→raster) | [pieces/printer](docs/pieces/printer/README.md) |
| GPS | Geolocalización + geocodificación inversa | [pieces/gps](docs/pieces/gps/README.md) |
| Mapas | Control `Map` nativo con pines | [pieces/maps](docs/pieces/maps/README.md) |
| Telefonía | Marcador del sistema vs. llamada directa | [pieces/phone](docs/pieces/phone/README.md) |
| Red | Estado y cambios de conectividad | [pieces/red](docs/pieces/red/README.md) |
| Integrada | App híbrida WebView + puente de comandos por URL + backend Blazor | [pieces/integrada](docs/pieces/integrada/README.md) |

## Decisiones de arquitectura (ADRs)

| ADR | Decisión |
|---|---|
| [0001](docs/04-decisions/0001-un-proyecto-por-tecnica.md) | Un proyecto MAUI mínimo por técnica |
| [0002](docs/04-decisions/0002-servicio-tipado-overlay-mvvm.md) | Patrón servicio tipado + overlay MVVM por dispositivo |
| [0003](docs/04-decisions/0003-puente-webview-comandos-url.md) | Puente WebView ↔ nativo por comandos en la URL |
| [0004](docs/04-decisions/0004-qr-bsn-recomendada-ios.md) | QR: BSN (Apple Vision) recomendada frente a BSM (ML Kit) en iOS |
| [0005](docs/04-decisions/0005-impresion-motor-dsl.md) | Impresión: motor DSL (JSON→raster) sobre ESC/POS directo |
| [0006](docs/04-decisions/0006-ci-ios-adhoc-android-local.md) | CI de iOS con firma ad-hoc; ejecución local en Android físico |
| [0007](docs/04-decisions/0007-secretos-fuera-del-repo.md) | Secretos de build fuera del repo (plantilla + secrets de CI) |
| [0008](docs/04-decisions/0008-trust-anchors-embebidos-webview.md) | TLS en WebView Android: trust anchors embebidos |
| [0009](docs/04-decisions/0009-ciclo-vida-overlay-gps-y-cancelacion-navegacion.md) | Cierre determinista del overlay GPS (Opción A) y cancelación robusta del puente de navegación |

## Estado y trazabilidad

- **Origen** (solo lectura): `../../Ejemplos_Maui_Devices/` · corpus anclado a `@24d611d`; el origen avanzó a `@fd6a1ed` durante la documentación (ver [GAP-REPORT §1](GAP-REPORT.md)).
- **Todo el corpus está en `draft`**: lo generó un agente por ingeniería inversa (`origin: reverse-engineered`); la promoción a `approved` es humana (Marco §12.4).
- **Lo inferido lleva `confidence`**; las contradicciones entre fuentes se **reportan**, no se resuelven en silencio (ver [GAP-REPORT §3](GAP-REPORT.md)).
- ⚠️ **Requiere acción humana**: hay una API key de Google Maps hardcodeada en el origen — ver [GAP-REPORT §3.1](GAP-REPORT.md).

## Pruebas End2End (dedo virtual)

La solución incluye una técnica de **prueba end2end sobre la UI real** que corre en CI (macOS) y deja **video + logs** como evidencia. No vive en `docs/` porque las utilidades de simulación quedan fuera del alcance indexado de la ia-db (ver [ia-db §nota v1.2](../ia-db/README.md)); se resume aquí como puntero.

**Piezas** (en el repo de código `../../Ejemplos_Maui_Devices/`):

| Pieza | Ruta | Rol |
|---|---|---|
| Workflow CI | `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` | Compila la app para el **simulador iOS** (ad-hoc), instala **Maestro** y llama al script |
| Script | `Utilities/simular_ui.sh` | Bootea el simulador, instala la `.app`, **graba video nativo** (`simctl io recordVideo`) y ejecuta el flujo Maestro; deja `recorrido.mp4` + `debug_logs/` |
| Flujo | `Utilities/end2end/<PACKAGE_NAME>.yaml` | "Dedo virtual" Maestro: toca controles **por texto visible** (`tapOn`), con `appId: ${APP_ID}` inyectado |
| (Legado) | `Utilities/simular.sh` | Enfoque pasivo previo: screenshots → GIF (`ffmpeg`). Reemplazado por el de video |

**Cómo funciona.** Maestro actúa por **texto/accessibility label**, no por `AutomationId`, así que un mismo YAML es portable (se puede autoría/validar en Android y correr en el simulador iOS). Convenciones: `launchApp` → `waitForAnimationToEnd` (8 s inicial por splash/WebView) → `tapOn: "<texto>"` por cada acción → `back`. Fallback por coordenada `tapOn: { point: "x%, y%" }`. **Límite conocido:** el simulador iOS no tiene cámara → las vistas de cámara/QR se ven negras; es esperado y sirve igual como evidencia de navegación.

**Arranque del simulador en el runner (operación).** El boot del simulador es la parte frágil del pipeline. Aprendido en CI (2026-07):

- **Síntoma:** el step de grabación se colgaba hasta agotar el `timeout-minutes` (30) con el simulador trabado en `Waiting on BackBoard`, que **nunca** completaba —ni tras `erase`—.
- **Causa:** en un runner **headless** con Xcode instalado a mano, el boot por `xcrun simctl boot`/`bootstatus -b` no levanta el stack gráfico (BackBoard/SpringBoard) y espera para siempre.
- **Fix (en `simular_ui.sh` + workflow):**
  1. **Bootear levantando `Simulator.app`** (`open -a "$(xcode-select -p)/Applications/Simulator.app" --args -CurrentDeviceUDID <uuid>`). Los runners macOS de GitHub corren sesión GUI (Aqua), así que es automatizado; Maestro sigue manejando la app por `simctl`.
  2. **Timeout + reintento limpio**: `bootstatus -b` acotado a 240 s; si se cuelga, `shutdown` + `erase` + reabrir GUI + 300 s. Peor caso ~9 min en vez de 30 (helper `run_with_timeout` con `perl alarm`).
  3. **Precalentado**: se abre la GUI y se bootea en el step "Verificando simulador instalado" (fire-and-forget), para que el simulador arranque **durante el build** y no se pague el boot en frío. `simular_ui.sh` hace short-circuit si ya está `Booted`.
  4. **Diagnóstico**: si el boot falla tras el reintento, vuelca las últimas líneas de `~/Library/Logs/CoreSimulator/CoreSimulator.log` para no quedar a ciegas.

**Captura del video con el recorrido completo (operación).** Aprendido en CI (2026-07):

- **Síntoma:** el simulador booteaba y grababa (~1:50), pero el video **terminaba sobre el arranque de la app** y no mostraba las interacciones.
- **Causa:** el arranque en frío (Release + WebView remoto `aplicada.somee.com`) es lento y se comía la grabación; además el flujo tocaba `Geo Pos` tras un wait fijo de 8 s, antes de que la UI existiera, y **fallaba en el primer `tapOn`** → las interacciones nunca ocurrían.
- **Fix (flujo + `simular_ui.sh`):**
  1. **Espera activa** en el flujo: `extendedWaitUntil: { visible: "Geo Pos", timeout: 120000 }` en vez del wait fijo. Maestro no interactúa hasta que el control nativo está presente.
  2. **Pre-warm** en el script: se lanza la app y se espera su carga **antes** de arrancar `recordVideo`; el flujo usa `launchApp: { stopApp: false }` para **no reiniciarla** (un solo arranque en frío, no grabado). El video queda enfocado en el recorrido, no en el splash.

**Herramientas y posibilidades para *generar* el flujo por exploración** (ver el prompt [`Crear-Flow-End2End-App.md`](../PROMPTs/Implementaciones/Crear-Flow-End2End-App.md)):
- **A — Barrido caja-negra (Android USB):** `adb` + `maestro hierarchy`/`uiautomator dump` para recorrer la UI y derivar el *storyboard*.
- **B — Grabación de navegación:** `maestro record`/`maestro studio` capturan la sesión → se normaliza a las convenciones y se replaya en iOS.
- **C — Derivación estática:** leer los `Text=` de las páginas MAUI y emitir el flujo sin dispositivo (provisional hasta validar).

> **Deuda técnica.** Los YAML de `Utilities/end2end/` nombran apps distintas pero `…gps.yaml` y `…qr.dialog.yaml` siguen con el **contenido de la híbrida**; `simular_ui.sh` resuelve el flujo como `end2end/${PACKAGE_NAME}.yaml`, así que cada app debería tener su propio recorrido. `…integrada.hibrida.yaml` ya fue rehecho a partir de la UI real (Estrategia B, validación en dispositivo pendiente).
>
> **Textos reales de MainPage (híbrida)** — la barra inferior declara 4 botones nativos: `Volver` · `Geo Pos` · `Llamar` · `Leer QR`. Ojo: el de GPS dice **"Geo Pos"**, no "Geo Posicionar" (texto que usaba el flujo previo y no matcheaba). El "Volver" de `QRLectorPage` es solo ícono (`arrow_back`, sin texto) → requiere coordenada o gesto `back`.
>
> **Verificado en dispositivo real** (Moto G42, Android, 1080px, vía `adb uiautomator dump`, 2026-07): los textos `Volver`/`Geo Pos`/`Llamar` coinciden y son tappables. En ese ancho la fila de 4 botones no entra y `Leer QR` queda recortado a la derecha; el flujo corre sobre el simulador iOS del CI (iPhone 17 Pro Max, más ancho), donde entra. Si algún runner lo recortara, el YAML trae un fallback por coordenada comentado.

## Para agentes

1. Leer [`docs-manifest.yaml`](docs-manifest.yaml): piezas, `type`, `required_docs` y `gaps` declarados.
2. Leer el frontmatter de los `.md` bajo `docs/` (IDs estables `PIECE-*`, `ARCH-*`, `ADR-*`, `API-*`, `OPS-*`).
3. Recuperar contexto de código desde la [ia-db](../ia-db/README.md) antes de releer fuentes.
4. El gap documental se calcula de `docs-manifest.yaml` + frontmatter + [GAP-REPORT](GAP-REPORT.md); no requiere leer prosa.
