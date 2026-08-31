# Índice 10 — Documentación transversal

> **Propósito**: mapear el material de apoyo que cruza varios dominios (certificados SSL/TLS, notas técnicas de `otros/`) y las convenciones de documentación del repo (`CHANGELOG`, `README`).
> **Fuente primaria**: `Ejemplos_Devices/Docs/` + `CHANGELOG.md` + `README.md` (raíz).
> **Entrada ia-db**: [README](../README.md) · [Índice maestro](00_MASTER-INDEX.md)

---

## 1. Mapa de la documentación transversal

```
Ejemplos_Devices/Docs/
├── Certificados-SSL/          # guía TLS/PKI para MAUI Híbrida (WebView Android)
│   ├── README.md
│   ├── Anexo-A-Comandos.md
│   └── Anexo-B-Glosario.md
├── otros/                     # notas técnicas varias
│   ├── AsyncRelayCommandOptions.md
│   ├── Eventos.md
│   ├── GetEnviromentVersion.md
│   └── propuesta-rosetta.md
├── qr-nuget/     ─▶ índice 02 (solo referencia)
└── web-hibrida/  ─▶ índice 08 (solo referencia)

CHANGELOG.md   (raíz)          # Keep a Changelog (es-ES 1.1.0)
README.md      (raíz)          # portada de dominios
```

> **Fuera de alcance de este índice** (solo referencia): `Docs/qr-nuget/` (análisis y migración de NuGets de QR) → **índice 02**; `Docs/web-hibrida/` (captura-foto, envio-api, lectura-qr, llamada, maui-hibrido) → **índice 08**.

---

## 2. `Docs/Certificados-SSL/` — TLS/PKI en Android WebView

Guía didáctica (es-AR) para resolver el error **`Trust anchor for certification path not found`** en una app **.NET MAUI Híbrida** cuyo WebView (Android System WebView = Chromium) abre una URL HTTPS. Caso de estudio: `https://aplicada.somee.com` en un Moto g42 (Android 12/13). Reutilizable para cualquier dominio.

| Archivo | Contenido | Fuente |
|---|---|---|
| `README.md` | Resumen ejecutivo, PKI en 5 min, diagnóstico, solución (`network_security_config.xml` + `res/raw`), mantenimiento, errores comunes | `Docs/Certificados-SSL/README.md` |
| `Anexo-A-Comandos.md` | Comandos de inspección: `openssl s_client -showcerts`, PowerShell `X509Chain`, `adb logcat -s cr_X509Util` | `Docs/Certificados-SSL/Anexo-A-Comandos.md` |
| `Anexo-B-Glosario.md` | Glosario PKI/TLS (AIA, CA, cadena, DER/PEM, trust anchor, SNI, SAN, MITM, …) | `Docs/Certificados-SSL/Anexo-B-Glosario.md` |

### 2.1. Diagnóstico (núcleo de la guía)

| Concepto | Resumen | Fuente |
|---|---|---|
| Síntoma | `cr_X509Util: Failed to validate the certificate chain … Trust anchor for certification path not found` | `README.md §1` |
| Causa raíz | La raíz **`ISRG Root X2`** (Let's Encrypt/ISRG, ECDSA 2020) **no está** en el trust store de Android viejo → cadena abierta | `README.md §1, §3.2` |
| No es | **No es Sectigo** (suposición errada del setup original; de ahí el nombre `download_sectigo.ps1`) | `README.md §1, §8` |
| Cadena real | `[0] aplicada.somee.com → [1] YE2 → [2] Root YE → [3] ISRG Root X2` (raíz auto-firmada, el server NO la envía) | `README.md §3.2`, `Anexo-A §A.1` |
| Solución | Embeber la **raíz** (y por robustez los 2 intermedios) en `res/raw/*.pem` y declararlos como `trust-anchors` por dominio en `network_security_config.xml` | `README.md §4` |

La cadena verificada (con `openssl` y `X509Chain`):

| # | Rol | Subject | Issuer | ¿Lo envía el server? |
|---|---|---|---|---|
| 0 | Hoja | `CN=aplicada.somee.com` | `CN=YE2, O=Let's Encrypt` | Sí |
| 1 | Intermedio | `CN=YE2, O=Let's Encrypt` | `CN=Root YE, O=ISRG` | Sí |
| 2 | Intermedio | `CN=Root YE, O=ISRG` | `CN=ISRG Root X2, O=ISRG` | Sí |
| 3 | **Raíz (anchor)** | `CN=ISRG Root X2` | *(auto-firmado)* | **NO** → debe estar en el cliente |

### 2.2. Herramientas de inspección (Anexo A)

| Herramienta | Qué te dice | Señal clave | Fuente |
|---|---|---|---|
| `openssl s_client -showcerts` | Qué certificados **envía el server** y sus `s:`/`i:` | El `i:` del último cert = la raíz que falta | `Anexo-A §A.1` |
| PowerShell `X509Chain` (`download_sectigo.ps1`) | Si **Windows** valida la cadena | `Cadena válida desde Windows` → el problema es **solo** de Android | `Anexo-A §A.2` |
| `adb logcat -s cr_X509Util:*` | Si el **WebView** acepta la cadena | Desaparece `Trust anchor … not found` cuando está resuelto | `Anexo-A §A.3` |

### 2.3. Mantenimiento y trade-offs

| Estrategia | Qué embebés | Pro / Contra | Fuente |
|---|---|---|---|
| **Solo la raíz** (recomendada) | `isrg_root_x2.pem` | Mínimo mantenimiento (raíz válida hasta 2040); depende de que el server siga enviando intermedios | `README.md §6` |
| **Raíz + intermedios** (la elegida) | `isrg_root_x2.pem`, `root_ye.pem`, `ye2.pem` | Robusta ante cadenas incompletas; **rompe cuando Let's Encrypt rota los intermedios** y hay que regenerar los `.pem` | `README.md §6` |

### 2.4. Archivos de la app que la guía toca (proyecto `Integrada/Ejemplo_Maui_Hibrida/`)

```
Platforms/Android/
├── AndroidManifest.xml                         # android:networkSecurityConfig="@xml/network_security_config"
└── Resources/
    ├── xml/network_security_config.xml         # <domain-config> aplicada.somee.com + <trust-anchors>
    └── raw/{isrg_root_x2,root_ye,ye2}.pem       # raíz + 2 intermedios (PEM)
```

Fuentes de `trust-anchors` (`<certificates src="…">`), `README.md §4.1`:

| `src` | Significado | Uso recomendado |
|---|---|---|
| `"system"` | Raíces del SO | **Siempre** incluir |
| `"user"` | CAs instaladas por el usuario | **Evitar en producción** (riesgo MITM) |
| `"@raw/<nombre>"` | Certificado embebido en `res/raw/<nombre>.(pem\|der)` | Aportar la raíz que falta (nombre **sin** extensión) |

Reglas críticas de `res/raw` (rompieron el build original): nombres solo `[a-z0-9_]` minúscula; sin duplicar base con distinta extensión (`.cer` + `.pem` → *Duplicate resources*); en `@raw/<nombre>` va **sin** extensión (`README.md §7`). En producción, evitar `src="user"` por riesgo MITM (`README.md §4.3`).

> Cruce: el proyecto sobre el que aplica (`Ejemplo_Maui_Hibrida`, WebView) pertenece al **índice 08 (web-híbrida)**; aquí se documenta solo el material SSL/PKI transversal.

---

## 3. `Docs/otros/` — notas técnicas varias

| Archivo | Tema | Aplica a | Fuente |
|---|---|---|---|
| `AsyncRelayCommandOptions.md` | Clase didáctica `AsyncRelayCommand` (MVVM sin `CommunityToolkit.Mvvm`): anti-reentrancia, `IsRunning`, límites (no captura excepciones, `CancellationToken` siempre `None`), comparación con CommunityToolkit | `GPS/…/ViewModels/AsyncRelayCommand.cs`, `Phone/Ejemplo_Maui_DirectCall/…` | `Docs/otros/AsyncRelayCommandOptions.md` |
| `Eventos.md` | `WebView` + `EventToCommandBehavior` con **compiled bindings**: por qué el `Command` queda `null` y por qué `e` llega `null`; 3 soluciones (x:Reference, x:DataType en behavior, code-behind) + `EventArgsConverter` | `Integrada/Ejemplo_Maui_Hibrida` | `Docs/otros/Eventos.md` |
| `GetEnviromentVersion.md` | Prerequisitos de entorno: 7-Zip (requerido por SDK Android), NuGets usados (OneSignal, SkiaSharp, CommunityToolkit.Maui.Camera…), `dotnet sdk check`, `dotnet workload list/update` | entorno de desarrollo | `Docs/otros/GetEnviromentVersion.md` |
| `propuesta-rosetta.md` | Propuesta CI: compilar el simulador iOS en `x86_64`/Rosetta para sortear que Google ML Kit no publica slice de simulador arm64 (afecta `BarcodeScanner.Mobile.Maui`) | job CD iOS (índice 09) | `Docs/otros/propuesta-rosetta.md` |

### 3.1. Puntos clave por documento

**`AsyncRelayCommandOptions.md`** — implementación minimalista (~50 líneas, sin NuGet):
- Resuelve: **espera de finalización** y **anti-reentrancia** (`IsRunning` deshabilita el botón mientras corre); envuelve un `Func<Task>` en `ICommand`.
- Límites conocidos: `Execute` es `async void` → **no captura excepciones** (el `try/catch` va en el ViewModel); no expone `ExecuteAsync` awaitable; el `CancellationToken` siempre vale `None` (cancelación vía coordinador externo, p. ej. `GpsCoordinator`).
- Regla práctica: para cancelación fina / manejo de errores estructurado, migrar a `CommunityToolkit.Mvvm` (`README.md §5` de ese doc lo compara punto a punto).

**`Eventos.md`** — `WebView` + `EventToCommandBehavior` con compiled bindings (`x:DataType`):
- Problema 1: el `Command` queda `null` porque los `Behavior` no heredan el `BindingContext` de forma fiable → soluciones: `Source={x:Reference page}` (recomendada), `x:DataType` en el behavior, o code-behind.
- Problema 2: el comando se invoca pero `e` llega `null` — en CommunityToolkit.Maui el `EventToCommandBehavior` **no pasa los `EventArgs` por defecto**; hay que configurar un `EventArgsConverter` o manejar el evento en code-behind.

**`GetEnviromentVersion.md`** — prerequisitos y verificación del entorno de desarrollo (7-Zip para el SDK Android; NuGets base; `dotnet sdk check`, `dotnet workload list/update`). Empareja con `Ejemplos_Devices/scripts/GetEnviromentVersion.bat` (índice 09).

**`propuesta-rosetta.md`** — 3 cambios de YAML (sin tocar código) para el build de simulador afectado por ML Kit: RID → `iossimulator-x64`, `xcodebuild -downloadPlatform iOS -architectureVariant universal`, y `softwareupdate --install-rosetta`. Incluye causa raíz (ML Kit no publica slice de simulador arm64), riesgos (rendimiento bajo Rosetta, disco, deprecación de Rosetta 2) y rollback.

### 3.2. Cruces con otros índices

- **`propuesta-rosetta.md`** es la justificación técnica del RID `iossimulator-x64` de los workflows **`BSM`** y de los steps `ROSETTA`/`-architectureVariant universal` → detalle de aplicación en el **índice 09**. Los 3 cambios propuestos (RID → x64, runtime universal, `softwareupdate --install-rosetta`) ya están incorporados en `cd-ios-qr.BSM.*.yml`.
- **`GetEnviromentVersion.md`** documenta el entorno que el script `Ejemplos_Devices/scripts/GetEnviromentVersion.bat` inspecciona → **índice 09**.
- **`AsyncRelayCommandOptions.md`** y **`Eventos.md`** documentan patrones de código MVVM/WebView de los ejemplos (dominios GPS/Phone/Híbrida) → material de referencia para los índices de esos dominios.

---

## 4. `CHANGELOG.md` (raíz) — convención

- **Formato**: [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/) en **es-ES** (`CHANGELOG.md:1-4`).
- **Alcance declarado**: "Cambios notables de los ejemplos de dispositivos MAUI (`Ejemplos_Maui_Devices`)".
- **Estructura de entrada**: encabezado `## [AAAA-MM-DD] — <título>` seguido de subsecciones **`### Agregado` / `### Cambiado`** (categorías Keep a Changelog: Added/Changed/…).
- **Estilo**: viñetas con rutas y símbolos concretos en `código`, agrupadas por área (p. ej. `LibApp/…`, `DTOs/Print/…`, versiones de paquetes `MotorDsl.* 1.0.12 → 1.0.13`), incluyendo cambios de CI (p. ej. la recategorización del workflow de la híbrida a `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml`; ver índice 09).
- **Variantes de subsección observadas**: además de `### Agregado` / `### Cambiado`, se usan `### Corregido` y `### Activado`; varias entradas abren con una línea **`Alcance:`** que delimita qué se tocó (y aclara, cuando aplica, «No se tocó código de la app»).

> Entrada más reciente al momento del índice: `## [2026-07-23] — Panel con ambos modos de GPS y normalización de namespaces en LibApp`. Las dos entradas más recientes (`2026-07-23` ×2) cubren el **puente de comandos por URL** de la app híbrida (Plan 1: clasificación separada de ejecución; y el panel ejercitando los dos modos de `CommandDelivery` + namespaces `LibApp.*`) → efecto indexado en el **índice 08 §4.3 y §7.1**; las tres anteriores (`2026-07-18` ×3) cubren la **técnica de simulación end2end** (flujo Maestro rehecho + activación del `push` del workflow, robustez del boot del simulador, pre-warm de la grabación) → **índice 09 §2 y §4.2**. **No se copian entradas aquí**; consultar `CHANGELOG.md` para el detalle vigente.

---

## 5. `README.md` (raíz) — portada de dominios

Portada mínima "Ejemplos de uso de dispositivos desde MAUI .NET" que lista los dominios con enlace a un `Ejemplo_Docs_<Dominio>/Readme.md` por área (`README.md:1-25`):

| Dominio | Enlace en el README | Estado |
|---|---|---|
| Thermal Printer BlueTooth | `Ejemplos_Devices/Printer/Ejemplo_Docs_Printer/Readme.md` | con GIF de ejemplo |
| Lectura de QR | `Ejemplos_Devices/QR/Ejemplo_Docs_QR/Readme.md` | "En construcción …" |
| Cámara | `Ejemplos_Devices/Camera/Ejemplo_Docs_Photo/Readme.md` | "En construcción …" |
| Red y Conectividad | `Ejemplos_Devices/Red/Ejemplo_Docs_Red/Readme.md` | "En construcción …" |
| GPS | `Ejemplos_Devices/GPS/Ejemplo_Docs_GPS/Readme.md` | "En construcción …" |
| Maps | `Ejemplos_Devices/Maps/Ejemplo_Docs_Maps/Readme.md` | "En construcción …" |

Convención de documentación observada: cada dominio tiene una carpeta `Ejemplo_Docs_<Dominio>/` con su `Readme.md`; el README raíz es solo el índice de entrada (muchos dominios aún "En construcción"). La documentación técnica profunda vive en `Ejemplos_Devices/Docs/` (este índice) y en los Readme por-ejemplo.

---

## 6. Trazabilidad rápida

| Afirmación | Fuente project-relative |
|---|---|
| Error/causa/solución TLS Android | `Ejemplos_Devices/Docs/Certificados-SSL/README.md` |
| Comandos de inspección de cadena | `Ejemplos_Devices/Docs/Certificados-SSL/Anexo-A-Comandos.md` |
| Glosario PKI/TLS | `Ejemplos_Devices/Docs/Certificados-SSL/Anexo-B-Glosario.md` |
| `AsyncRelayCommand` (MVVM) | `Ejemplos_Devices/Docs/otros/AsyncRelayCommandOptions.md` |
| WebView + EventToCommandBehavior | `Ejemplos_Devices/Docs/otros/Eventos.md` |
| Prerequisitos de entorno | `Ejemplos_Devices/Docs/otros/GetEnviromentVersion.md` |
| Propuesta Rosetta/x64 (CI) | `Ejemplos_Devices/Docs/otros/propuesta-rosetta.md` |
| Formato del changelog | `CHANGELOG.md:1-4` |
| Portada de dominios | `README.md:1-25` |

**Referencias externas a este índice**: `Docs/qr-nuget/` → índice 02 · `Docs/web-hibrida/` → índice 08 · pipeline/CI (rosetta, GetEnviromentVersion.bat) → índice 09.
