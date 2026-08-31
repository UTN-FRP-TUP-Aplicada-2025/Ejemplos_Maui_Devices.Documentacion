# Tool-Prompt — Diseñar un Flujo End2End de una App (dedo virtual Maestro)

> **Invocación**:
> - `Lee y ejecuta Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Implementaciones/Crear-Flow-End2End-App.md`
> - Parámetros (opcionales; si faltan, el prompt los infiere o los pregunta):
>   - `PROYECTO` — carpeta del proyecto MAUI (p.ej. `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida`)
>   - `PACKAGE_NAME` — bundle/applicationId (p.ej. `com.ejemplos.devices.integrada.hibrida`)
>   - `ESTRATEGIA` — `A` (barrido caja-negra), `B` (grabación de navegación), `C` (derivación estática) o `PREGUNTAR`

---

# Contexto

Este proyecto ya tiene una técnica de prueba **end2end sobre la UI real** de la app, montada en CI. Antes de diseñar nada, entendé las piezas existentes:

- **Base de conocimiento (leer primero, no releer todo el repo):**
  - `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/ia-db/README.md` → índice **09 (CI/CD y build)** y **08 (App híbrida integrada)**.
  - `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/Ejemplos_Maui_Devices-docs/README.md` → sección **End2End / dedo virtual**.
- **Piezas de la técnica actual (fuente de verdad):**
  1. App bajo prueba: un proyecto MAUI (referencia: `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida`).
  2. Workflow CI: `.github/workflows/cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` — compila para el **simulador iOS** (ad-hoc, sin firma real), instala Maestro y llama al script de simulación.
  3. Script de simulación: `Utilities/simular_ui.sh` — bootea el simulador, instala la `.app`, arranca captura de logs + **grabación de video nativa** (`xcrun simctl io … recordVideo`), y ejecuta el **flujo Maestro**. Deja `recorrido.mp4` + `debug_logs/` como evidencia.
  4. Flujo Maestro: `Utilities/end2end/<PACKAGE_NAME>.yaml` — el "dedo virtual" que recorre la app.

## Cómo funciona el "dedo virtual" (resumen operativo)

- El flujo es un YAML de **Maestro** que toca controles **por su TEXTO VISIBLE** (`tapOn: "…"`), sin depender de `AutomationId` ni de inspeccionar el WebView: en MAUI el texto de un botón nativo se expone como *accessibility label*.
- El `appId` se inyecta desde el script: `maestro test -e APP_ID=<bundle>`, y el YAML lo consume como `appId: ${APP_ID}`.
- Convenciones observadas en el flujo de referencia:
  - `- launchApp`
  - `- waitForAnimationToEnd: { timeout: <ms> }` tras el arranque y tras cada transición (splash/WebView tarda; usar 8000 ms al inicio, 3000–5000 ms entre pasos).
  - `- tapOn: "<texto visible>"` para cada acción.
  - Fallback si el texto no matchea: coordenada relativa `- tapOn: { point: "16%, 95%" }`.
  - `- back` para volver (best-effort en iOS).
  - **Limitación del simulador iOS:** no hay cámara real → vistas de cámara/QR se ven negras. Es **esperado** y sirve igual como evidencia de que la navegación funciona.

> **Deuda técnica detectada (verificar y respetar al escribir el nuevo flujo):** hoy los tres YAML de `Utilities/end2end/` (`…gps.yaml`, `…qr.dialog.yaml`, `…integrada.hibrida.yaml`) tienen **contenido idéntico** (el recorrido de la híbrida). El nombre del archivo **debe** ser `<PACKAGE_NAME>.yaml` porque `simular_ui.sh` lo resuelve como `end2end/${PACKAGE_NAME}.yaml`; no reutilizar el contenido de otro sin adaptarlo a la app real.

---

# Objetivo

Producir un **flujo Maestro end2end** (`Utilities/end2end/<PACKAGE_NAME>.yaml`) para la app indicada, **descubriendo la UI de la app por exploración propia** en lugar de escribirlo a ciegas, de modo que:

1. Sea consumible tal cual por `simular_ui.sh` en el CI de iOS (mismas convenciones que el flujo de referencia).
2. Recorra las pantallas/acciones relevantes (el "story board" de la app) y deje evidencia en video.
3. Quede documentado el **cómo se descubrió** (qué controles, qué textos, qué fallbacks).

---

# Solicitudes

## 1. Diagnóstico previo (obligatorio)

- Cargá los índices ia-db 08/09 y la sección End2End de la doc; no releas el repo completo.
- Confirmá `PROYECTO` y `PACKAGE_NAME` (leé el `.csproj`/`Info.plist`/`AndroidManifest` si no se pasaron).
- Verificá el flujo de referencia y las convenciones vigentes (`appId: ${APP_ID}`, timeouts, `tapOn` por texto).

## 2. Elegir estrategia de descubrimiento

Presentá al usuario las estrategias y **pedile que elija** (salvo que `ESTRATEGIA` venga fijada). Todas producen el **mismo formato** de salida (YAML Maestro portable), porque `tapOn` por texto es **cross-plataforma**: se puede autoría/validar en Android y correr el mismo YAML en el simulador iOS del CI.

### Estrategia A — Barrido caja-negra sobre teléfono Android (USB) → *storyboard*
El agente se conecta a un teléfono en modo desarrollador y **explora la UI sin ver el código**, generando la "historia" (storyboard) de pantallas.

- **Herramientas:** `adb` (dispositivo USB autorizado), `maestro` (dispositivo Android detectado por `maestro`), volcado de jerarquía de vistas.
- **Procedimiento:**
  1. `adb devices` → confirmar dispositivo autorizado. Instalar/lanzar el APK: `adb install -r <apk>` / `adb shell monkey -p <PACKAGE_NAME> -c android.intent.category.LAUNCHER 1`.
  2. Volcar la pantalla actual y extraer **textos táctiles**:
     - `maestro hierarchy` (jerarquía accesible que ve Maestro), o
     - `adb exec-out uiautomator dump /dev/tty` → parsear nodos `clickable="true"` y su `text`/`content-desc`.
     - Screenshot de apoyo: `adb exec-out screencap -p > paso_N.png`.
  3. Para cada control clicable con texto, generar un paso `tapOn: "<texto>"`, tocar, esperar animación, re-volcar. Repetir en **BFS acotado** (profundidad y nº de pantallas máximos para no divergir), registrando `back` al retroceder.
  4. Con el recorrido, armar el **storyboard**: lista ordenada de pantallas → acción → resultado esperado, y traducirlo al YAML.
- **Fortalezas:** no necesita el código; refleja lo que el usuario realmente ve. **Límites:** requiere hardware conectado; textos dinámicos o iconos sin label necesitan fallback por coordenada.

### Estrategia B — Grabación de la navegación real (Android) → *replay* en el simulador
El agente (o el usuario) **navega la app una vez** y la sesión se **graba** como flujo Maestro; luego ese mismo YAML se ejecuta en el simulador iOS vía `simular_ui.sh`.

- **Herramientas:** `maestro record` / `maestro studio` (captura interactiva de taps → pasos de flujo), Android físico o emulador.
- **Procedimiento:**
  1. Lanzar `maestro studio` con la app abierta; ejecutar el recorrido "feliz" (los caminos que importan para la evidencia).
  2. Exportar el flujo grabado; **normalizarlo** a las convenciones del repo: `appId: ${APP_ID}`, insertar `waitForAnimationToEnd` en cada transición, preferir `tapOn: "<texto>"` sobre coordenadas absolutas, quitar pasos específicos de Android que no apliquen a iOS.
  3. Anotar los pasos con comentarios (qué dispara cada acción; qué se ve en el simulador iOS, p.ej. cámara negra).
- **Fortalezas:** captura intención de usuario real, rápido. **Límites:** la grabación puede fijar coordenadas frágiles → hay que endurecerla a `tapOn` por texto.

### Estrategia C — Derivación estática desde el código MAUI (sin dispositivo)
Cuando no hay hardware/emulador, derivar el flujo **leyendo la UI declarada**.

- **Herramientas:** lectura de `MainPage.xaml`/páginas MAUI, `Text="…"`/`AutomationId` de los botones, y el catálogo de overlays (doc 07/08).
- **Procedimiento:** extraer los textos visibles de los controles de entrada (botones de la pantalla principal y navegación), ordenarlos por el "camino feliz", y emitir el YAML con `tapOn` por esos textos + `waitForAnimationToEnd`. Marcar como **provisional** hasta validarlo en dispositivo/simulador.
- **Fortalezas:** cero dependencias, reproducible en cualquier entorno. **Límites:** no ve estados en runtime; los textos deben coincidir exactamente con el label accesible.

### Recomendación de selección
- Hay teléfono/emulador Android disponible + se quiere realismo → **A** (o **B** si se prefiere grabar).
- Solo entorno de repo, sin dispositivo → **C**, y validar luego.
- Máxima robustez → **C para el esqueleto** + **A/B para verificar textos reales** (híbrido).

## 3. Generar el flujo

- Escribir `Ejemplos_Maui/Ejemplos_Maui_Devices/Utilities/end2end/<PACKAGE_NAME>.yaml` siguiendo **exactamente** las convenciones del flujo de referencia:

```yaml
# ─────────────────────────────────────────────────────────────────────────────
# <PACKAGE_NAME>.yaml — "Dedo virtual" para <PROYECTO>
# Descubierto por Estrategia <A|B|C>. Toca controles por su TEXTO VISIBLE.
# appId inyectado desde el script:  maestro test -e APP_ID=<bundle>
# Fallback por coordenada si un tapOn por texto fallara: - tapOn: { point: "x%, y%" }
# ─────────────────────────────────────────────────────────────────────────────
appId: ${APP_ID}
---
- launchApp
- waitForAnimationToEnd: { timeout: 8000 }   # splash + carga inicial

# <acción 1>
- tapOn: "<texto visible>"
- waitForAnimationToEnd: { timeout: 5000 }

# … repetir por cada paso del storyboard …

- back
- waitForAnimationToEnd: { timeout: 3000 }
```

- Reglas de calidad del flujo:
  - Preferir `tapOn` por **texto** > por **id** > por **coordenada** (última opción, comentada con el motivo).
  - Un `waitForAnimationToEnd` tras cada transición; timeout inicial alto (WebView/splash).
  - Comentar cada paso: qué dispara y qué se espera ver (incluida la limitación de cámara en simulador).
  - No romper el pipeline: el flujo es evidencia; los taps que puedan no encontrar cámara real son aceptables.

## 4. Validar

- Si hay dispositivo/emulador: `maestro test -e APP_ID=<PACKAGE_NAME> <ruta_yaml>` y revisar que cada paso resuelva (o su fallback).
- Sin dispositivo: al menos validar YAML (lint) y que los textos existan en la UI declarada (Estrategia C).
- Dejar registro del resultado de validación (qué pasos pasaron, cuáles usaron fallback).

## 5. Realimentar la documentación

- Si se descubrieron pasos, textos o límites nuevos, actualizar la sección End2End de `Ejemplos_Maui_Devices-docs/README.md` y, si corresponde, el índice ia-db 09. No duplicar; referenciar.

---

# Salidas esperadas

1. `Utilities/end2end/<PACKAGE_NAME>.yaml` — el flujo (o su versión provisional, marcada).
2. El **storyboard** (lista pantalla→acción→resultado) en el reporte.
3. Nota de qué **estrategia** se usó y resultado de la **validación**.
4. Realimentación documental si aplica.

---

# Framework

## Rule Set

- `/IA/IA.Prompts/PromptFramework/RuleSets/RuleSet-Lean.md`
