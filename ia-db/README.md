# ia-db — Base de conocimiento de `Ejemplos_Maui_Devices`

> **Instrucción para IA**: este archivo es el **punto de entrada único** a la base de conocimiento del repositorio de código `Ejemplos_Maui_Devices`. Antes de abrir código o documentación, leé este README, buscá el tema en la tabla de navegación o en los atajos y cargá **sólo** el índice que corresponda, o dos como mucho. Cada afirmación de los índices cita su fuente con una ruta relativa a la raíz del repo de código y, cuando aporta, con número de línea. Abrí el archivo fuente sólo si el índice no alcanza. No recorras el repositorio completo. ⚠️ marca trampas de lectura y discrepancias reales; «no verificado» marca lo que no se pudo comprobar.

---

## Navegación: «Necesito saber… → leo este índice»

| Necesito saber… | Índice |
|---|---|
| Qué es el repo, stack, catálogo de los 24 proyectos, patrones transversales, decisiones | [00_MASTER-INDEX](indexes/00_MASTER-INDEX.md) |
| Cómo se toma una foto y vuelve a la pantalla anterior; EXIF; selfie; permisos de cámara | [01_Camara](indexes/01_Camara.md) |
| Qué librería de QR usar, la variante `_Dialog`, cómo traducir la API, ML Kit en el simulador | [02_QR](indexes/02_QR.md) |
| Cómo se imprime en la térmica Bluetooth: ESC/POS, motor DSL, overlay, catálogo `PRN-*` | [03_Impresion-Termica](indexes/03_Impresion-Termica.md) |
| Cómo se obtiene la ubicación; permisos; geocodificación inversa; API keys | [04_GPS](indexes/04_GPS.md) |
| Cómo se usa el control `Map`: centrado, pines, tipo de mapa, clave de Google | [05_Mapas](indexes/05_Mapas.md) |
| Abrir el marcador o llamar directo; `CALL_PHONE`; `CallResult` y overlay | [06_Telefonia](indexes/06_Telefonia.md) |
| `Connectivity`, `NetworkAccess`, el esqueleto de red | [07_Red-Conectividad](indexes/07_Red-Conectividad.md) |
| La app híbrida: `WebView` + Blazor remoto, puente por URL, overlays, backend, tests | [08_App-Hibrida-Integrada](indexes/08_App-Hibrida-Integrada.md) |
| Workflows iOS, Xcode, firma, artefactos, simulación, Maestro, `.slnx`, TFMs, `.gitignore` | [09_CI-CD-y-Build](indexes/09_CI-CD-y-Build.md) |
| Dónde está escrito un porqué; README, CHANGELOG, `Docs/`, ADR; documentos desactualizados | [10_Documentacion-Transversal](indexes/10_Documentacion-Transversal.md) |

### Atajos por pregunta frecuente

| Pregunta | Índice · sección |
|---|---|
| ¿Qué demuestra cada proyecto y cuáles tienen workflow o están en la solución? | [00 §3](indexes/00_MASTER-INDEX.md) |
| ¿Qué aporta cada dominio a la app híbrida? | [00 §5.3](indexes/00_MASTER-INDEX.md) |
| ¿Cómo se decide el mensaje que ve el usuario ante un error? | [00 §6](indexes/00_MASTER-INDEX.md) · [03 §7](indexes/03_Impresion-Termica.md) · [08 §8.3](indexes/08_App-Hibrida-Integrada.md) |
| ¿Qué proyecto de cámara copiar para una app real? | [01 §12](indexes/01_Camara.md) |
| ¿Por qué hay 8 proyectos de QR y cuál conviene? | [02 §1](indexes/02_QR.md) · [02 §10](indexes/02_QR.md) |
| ¿«ZN» usa ZXing.Net.Maui? | [02 §2](indexes/02_QR.md) |
| ¿Por qué falla el simulador iOS arm64 con ML Kit? | [02 §8](indexes/02_QR.md) · [09 §7](indexes/09_CI-CD-y-Build.md) |
| ¿Cuáles son los códigos `PRN-*` y qué ve el usuario? | [03 §7.2](indexes/03_Impresion-Termica.md) |
| ¿Se puede imprimir en iOS? | [03 §6.3](indexes/03_Impresion-Termica.md) |
| ¿Por qué `Ejemplo_Maui_GPS` no compila recién clonado? | [04 §9](indexes/04_GPS.md) |
| ¿Dónde va la key de Google Maps y si la necesita iOS? | [05 §8.1](indexes/05_Mapas.md) · [05 §8.2](indexes/05_Mapas.md) |
| ¿DirectCall llama directo en iPhone? | [06 §5.2](indexes/06_Telefonia.md) · [06 §9](indexes/06_Telefonia.md) |
| ¿Por qué la app de Connectivity no muestra nada? | [07 §5](indexes/07_Red-Conectividad.md) |
| ¿Por qué la híbrida cancela la navegación y fija `e.Cancel` antes de cualquier `await`? | [08 §4.2](indexes/08_App-Hibrida-Integrada.md) |
| ¿Injection o Substitution para el GPS en la web? | [08 §6.2](indexes/08_App-Hibrida-Integrada.md) · [04 §10](indexes/04_GPS.md) |
| ¿Cómo agrego un comando nuevo al puente? | [08 §14](indexes/08_App-Hibrida-Integrada.md) |
| ¿Por qué un push no dispara el pipeline de mi ejemplo? | [09 §2.1](indexes/09_CI-CD-y-Build.md) |
| ¿Por qué el CI instala su propio Xcode? | [09 §5](indexes/09_CI-CD-y-Build.md) |
| ¿Qué ejemplos no tienen pipeline? ¿Qué falta en el `.slnx`? | [09 §3](indexes/09_CI-CD-y-Build.md) · [09 §12](indexes/09_CI-CD-y-Build.md) |
| ¿Qué ADR o análisis cita el código y existen? | [10 §10](indexes/10_Documentacion-Transversal.md) |
| ¿Qué documentos están desactualizados o tienen enlaces rotos? | [10 §12](indexes/10_Documentacion-Transversal.md) |

---

## Resumen ejecutivo

| | |
|---|---|
| **Proyecto** | `Ejemplos_Maui_Devices`: colección **didáctica** de ejemplos .NET MAUI de la cátedra |
| **Tipo** | Repositorio de ejemplos más una app integradora; no es un producto ni una librería |
| **Stack** | C# · .NET 10 · .NET MAUI (XAML) · Blazor Interactive Server · xUnit |
| **TFM** | `net10.0-android`, más `net10.0-ios` sólo al compilar en macOS (más `net10.0-windows10.0.19041.0` en Mapas, sólo en Windows) · mínimos Android 25 (21 en DirectCall), iOS 15.0 |
| **Proyectos** | 24: 22 apps MAUI, 1 web Blazor y 1 de tests. La solución `Ejemplos_Devices.slnx` tiene 23 (falta el de tests) |
| **Dominios** | Cámara (5) · QR (8) · Impresión térmica (3) · GPS (1) · Mapas (1) · Teléfono (2) · Red (1) · Integrada (3) |
| **CI** | 18 workflows de GitHub Actions, todos iOS al simulador en `macos-15` con firma ad-hoc; 17 compilan un proyecto real (uno es huérfano) y sólo el de la híbrida tiene `push` activo |
| **Repo de documentación** | `Ejemplos_Maui_Devices.Documentacion`: esta ia-db, ADR-0001…0009, `Analisis/`, `Guias/`, corpus `Ejemplos_Maui_Devices-docs/` |
| **Backend desplegado** | `https://aplicada.somee.com` (lo abre la híbrida; `Ejemplo_ws_Blazor`) |
| **Origen indexado** | commit `285f6fb` (2026-07-23), árbol limpio |

**Función principal.** Mostrar, con un ejemplo mínimo por técnica, cómo una app MAUI usa las capacidades del dispositivo, y cómo esas piezas se integran en una app híbrida donde **la pantalla la pone una web remota y el lado nativo ejecuta las capacidades**.

**Arquitectura en una línea.** Siete dominios aislados que repiten el patrón servicio → resultado tipado → overlay y convergen en `Ejemplo_Maui_Hibrida`: un `WebView` sobre Blazor Server remoto con un puente que intercepta la navegación por URL, ejecuta el comando nativo y devuelve el resultado al DOM.

---

## Estructura del repositorio indexado

```
Ejemplos_Maui_Devices/
├── README.md · CHANGELOG.md · .gitignore · vs.bat
├── .github/workflows/     18 pipelines cd-ios-* + Readme.md, Pipelinea-Version.md, Analisis/   → 09
├── Utilities/             simular.sh, simular_ui.sh, end2end/ (Maestro), download_sectigo.ps1 → 09
└── Ejemplos_Devices/
    ├── Ejemplos_Devices.slnx                                              → 09 §12
    ├── Camera/ QR/ Printer/ GPS/ Maps/ Phone/ Red/   un ejemplo por técnica → 01–07
    ├── Integrada/       app híbrida + backend Blazor + tests             → 08
    ├── Docs/            Certificados-SSL, otros, qr-nuget, web-hibrida   → 10
    └── scripts/         .bat de build+deploy Android (Windows)           → 09 §11
```

## Estructura de esta base

```
ia-db/
├── README.md                          ← este archivo (entrada única)
└── indexes/
    ├── 00_MASTER-INDEX.md             visión, stack, catálogo, patrones, decisiones
    ├── 01_Camara.md                   Ejemplos_Devices/Camera/**
    ├── 02_QR.md                       Ejemplos_Devices/QR/** + Docs/qr-nuget
    ├── 03_Impresion-Termica.md        Ejemplos_Devices/Printer/**
    ├── 04_GPS.md                      Ejemplos_Devices/GPS/** (API keys, geocodificación)
    ├── 05_Mapas.md                    Ejemplos_Devices/Maps/**
    ├── 06_Telefonia.md                Ejemplos_Devices/Phone/**
    ├── 07_Red-Conectividad.md         Ejemplos_Devices/Red/**
    ├── 08_App-Hibrida-Integrada.md    Ejemplos_Devices/Integrada/** + Docs/web-hibrida
    ├── 09_CI-CD-y-Build.md            workflows, Utilities, scripts, .slnx, .gitignore
    └── 10_Documentacion-Transversal.md  README, CHANGELOG, Docs/**, Ejemplo_Docs_*, repo de documentación
```

---

## Restricciones para el agente que consuma esta base

- **No cargues todos los índices.** Usá la navegación o los atajos y traé uno o dos.
- **No confundas los dos repos.** El código está en `Ejemplos_Maui_Devices`. Los ADR, `Analisis/`, `Guias/` y esta base están en `Ejemplos_Maui_Devices.Documentacion`. Los comentarios del código citan documentos del segundo, y uno lo hace con el nombre de repo equivocado («`.Documentos`», [10 §10](indexes/10_Documentacion-Transversal.md)).
- **Si la evidencia contradice un índice, prevalece la evidencia.** Corregí el índice y dejalo anotado en el manifiesto.
- **No completes por inferencia.** Respetá las marcas «no verificado» y ⚠️.
- **No modifiques el repo de código** al actualizar esta base. Sólo se escribe dentro de `ia-db/`.
- **El repo es didáctico.** El código comentado, la duplicación entre ejemplos y los proyectos «en construcción» son deliberados. No los «arregles» ni los reportes como defectos.
- **Los números de línea valen para `285f6fb`.** Si el repo de código avanzó, revalidalos antes de citarlos.

---

## Manifiesto de generación

- Generado por : `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Iniciar-Indexado.md`
- Invocado por : `/APLICADA/2026/EJEMPLOS/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Indexado/Crear-Indexado.md`
- Repo de código : `/APLICADA/2026/EJEMPLOS/Ejemplos_Maui_Devices` · Repo de esta base: `/APLICADA/2026/EJEMPLOS/Ejemplos_Maui_Devices.Documentacion` (rutas desde la raíz del workspace)
- Alcance      : `Ejemplos_Maui_Devices`, modo proyecto. 11 índices: maestro (00) más cámara, QR, impresión térmica, GPS, mapas, telefonía, red, app híbrida integrada, CI/CD y build, y documentación transversal (01–10)
- Fuentes      : `Ejemplos_Devices/` (Camera, QR, Printer, GPS, Maps, Phone, Red, Integrada, Docs, scripts, `Ejemplos_Devices.slnx`), `.github/workflows/` (con `Analisis/`), `Utilities/`, `README.md`, `CHANGELOG.md`, `.gitignore`, `vs.bat`; y, para verificar citas, el repo `Ejemplos_Maui_Devices.Documentacion` (`Ejemplos_Maui_Devices-docs/`, `Analisis/`, `Guias/`)
- Exclusiones  : `.git`, `bin`, `obj`, `.vs`, `wwwroot/lib`, `Platforms/*/Resources`, binarios e imágenes, artefactos de build, lo ignorado por el `.gitignore` del proyecto y la propia `ia-db`
- Estado del origen : commit `285f6fb` (2026-07-23), árbol de trabajo limpio
- Mapa índice → fuentes : el árbol «Estructura de esta base» de arriba, más el campo **Fuente primaria** de cada índice
- Generado     : 2026-10-27 · Versión: 1.0 (regeneración completa; reemplaza a la v1.0 del 2026-08-31, ver nota)
- Actualizar   : `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md` (o el prompt local `/APLICADA/2026/EJEMPLOS/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Indexado/Actualizar-Indexado.md`)

> **Nota de regeneración (2026-10-27).** Esta base reemplaza a la ia-db v1.0 del 2026-08-31 (commit `14b22fb` del repo de documentación). El usuario invocó `Crear-Indexado` después de que los repos se mudaran de `/APLICADA/` a `/APLICADA/2026/EJEMPLOS/`, y la base se regeneró desde cero. El código indexado es el mismo commit `285f6fb`. La versión anterior queda en el historial de git.
>
> **Qué cambió de fondo respecto de la v1.0**:
> - Los índices 01–10 pasaron de 1475 a 3409 líneas (el maestro, de 177 a 253), con números de línea verificados uno por uno, marcas ⚠️ y «no verificado», y la cabecera estándar (Propósito / Fuente primaria / Índices relacionados).
> - El maestro suma, por proyecto, el workflow y la pertenencia al `.slnx`; los tres modelos de mensaje de error (§6) y las dos generaciones de overlay (§5.2). Conserva de la v1.0 la tabla «qué aporta cada dominio a la híbrida» (§5.3) y los dispositivos de referencia (§2).
> - Corrección de fondo a la v1.0: el `IUiDispatcher` de la híbrida lo usa sólo el overlay de Red (`NetworkOverlayViewModel`), no el de GPS como decía su §7.
> - Sin duplicación entre índices: el detalle de los modos de entrega del GPS vive en 08 §6.2, los secretos de CI en 09 §10, las citas del código a documentos externos en 10 §10 y el catálogo `PRN-*` en 03 §7.2; los demás índices los referencian.
> - Corrección al maestro de la v1.0: hay 7 carpetas `Ejemplo_Docs_*`, no 8 (Teléfono no tiene). La v1.0 ya registraba el huérfano de QR, el `push` activo sólo en la híbrida y que «ZN» usa `CameraMaui`; esta versión lo mantiene.
