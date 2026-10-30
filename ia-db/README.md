# ia-db — Base de conocimiento de `Ejemplos_Maui_Devices`

> **Instrucción para IA:** este archivo es el **punto de entrada único** a la base de conocimiento del repositorio de código `Ejemplos_Maui_Devices`. Antes de explorar código o documentación, leé este README, ubicá el tema en la tabla de navegación y cargá **sólo** el índice (o los dos) que correspondan. Cada afirmación de los índices lleva su fuente como `ruta:línea` (relativa a la raíz del repo de código) o como enlace a la prosa del repo de documentación: recurrí al archivo fuente únicamente cuando el índice no alcance. No recorras el repositorio completo.

---

## Navegación — «Necesito saber… → leo este índice»

| Necesito saber… | Índice |
|---|---|
| De qué trata, stack, catálogo de proyectos, patrones que se repiten, avisos transversales | [00_MASTER-INDEX](indexes/00_MASTER-INDEX.md) |
| Cómo se toma una foto y vuelve a la pantalla que la pidió (callback vs Task); normalización EXIF; selfie | [01_Camara](indexes/01_Camara.md) |
| Por qué hay 8 proyectos de QR, qué librería usa cada uno, la variante `_Dialog`, ML Kit en el simulador iOS | [02_QR](indexes/02_QR.md) |
| Impresión térmica Bluetooth: ESC/POS, motor DSL, catálogo de errores `PRN-*`, permisos | [03_Impresion-Termica](indexes/03_Impresion-Termica.md) |
| Ubicación, permisos, geocodificación inversa, `ApiKeys.cs`, overlay de GPS | [04_GPS](indexes/04_GPS.md) |
| Control `Map`: pines, centrado, tipos, clave de Google | [05_Mapas](indexes/05_Mapas.md) |
| Marcador vs llamada directa, `CALL_PHONE`, overlay de llamada | [06_Telefonia](indexes/06_Telefonia.md) |
| `Connectivity`, estados de red y cambios | [07_Red-Conectividad](indexes/07_Red-Conectividad.md) |
| App híbrida: WebView + Blazor Server, puente de comandos por URL, overlays, backend, trust anchors, tests | [08_App-Hibrida-Integrada](indexes/08_App-Hibrida-Integrada.md) |
| Workflows de CI iOS, plantilla del pipeline, Xcode, Maestro, `.slnx`, build local | [09_CI-CD-y-Build](indexes/09_CI-CD-y-Build.md) |
| Dónde está documentado un porqué: ADR, piezas, guías, análisis, prosa del código | [10_Documentacion-Transversal](indexes/10_Documentacion-Transversal.md) |

**Atajos por pregunta frecuente**

| Pregunta | Índice · sección |
|---|---|
| ¿Por qué hay 8 proyectos de QR y cuál conviene? | [02 §1](indexes/02_QR.md) · [02 §8](indexes/02_QR.md) |
| ¿Cómo viaja un comando de la web al nativo y vuelve? | [08 §3](indexes/08_App-Hibrida-Integrada.md) |
| ¿Cómo agrego un comando al puente? | [08 §11](indexes/08_App-Hibrida-Integrada.md) |
| ¿Por qué el proyecto de GPS no compila recién clonado? | [04 §5](indexes/04_GPS.md) |
| ¿Cómo se decide el mensaje que ve el usuario ante un error de impresión? | [03 §6](indexes/03_Impresion-Termica.md) |
| ¿Qué proyectos no tienen pipeline? ¿Por qué el CI instala su propio Xcode? | [09 §2](indexes/09_CI-CD-y-Build.md) · [09 §3](indexes/09_CI-CD-y-Build.md) |
| ¿Qué dice cada ADR? | [10 §2](indexes/10_Documentacion-Transversal.md) |

---

## Resumen ejecutivo

| | |
|---|---|
| **Proyecto** | `Ejemplos_Maui_Devices` — colección **didáctica** de ejemplos .NET MAUI de acceso a dispositivos |
| **Tipo** | Repositorio de ejemplos + una app integradora; no es un producto ni una librería |
| **Stack** | C# · .NET 10 · .NET MAUI · XAML · Blazor Server · xUnit |
| **TFM** | `net10.0-android` (+ `net10.0-ios` en macOS); mínimos Android 25 / iOS 15.0 |
| **Proyectos** | 24 reales (22 apps MAUI, 1 web Blazor, 1 de tests); 23 en `Ejemplos_Devices.slnx` (falta el de tests) |
| **CI** | 18 workflows `cd-ios-*` de GitHub Actions (`macos-15`, simulador iOS); Android se compila local |
| **Backend de la híbrida** | `https://aplicada.somee.com` (URL inicial del WebView) |
| **Repo de documentación** | `Ejemplos_Maui_Devices.Documentacion`: aloja esta ia-db, la prosa (`Ejemplos_Maui_Devices-docs/`, `Guias/`, `Analisis/`) y su CHANGELOG |
| **Origen indexado** | HEAD local `cc80ce1` (2026-10-28) — ver manifiesto |

**Función principal.** Mostrar, con un ejemplo mínimo por técnica, cómo una app MAUI usa cámara, QR, impresora térmica Bluetooth, GPS, mapas, teléfono y red, y cómo esas capacidades se integran en una app híbrida donde **la pantalla la pone una web remota y el nativo ejecuta**.

**Arquitectura en una línea.** Siete dominios aislados (servicio → resultado tipado → overlay) que convergen en `Ejemplo_Maui_Hibrida`: un `WebView` sobre Blazor Server remoto con un puente de comandos por URL que intercepta la navegación, ejecuta la capacidad nativa y devuelve el resultado a la web.

---

## Estructura del repositorio indexado

```
Ejemplos_Maui_Devices/
├── README.md · CHANGELOG.md · .gitignore · vs.bat
├── .github/workflows/      18 pipelines cd-ios-* + notas de versión de pipeline
├── Utilities/              simulación iOS y flujos Maestro end2end
└── Ejemplos_Devices/       solución .slnx
    ├── Camera/ QR/ Printer/ GPS/ Maps/ Phone/ Red/   un ejemplo por técnica
    ├── Integrada/          app híbrida + backend Blazor + tests
    ├── Docs/               prosa transversal (SSL, QR, web híbrida, otros)
    └── scripts/            .bat de arranque Windows
```

## Estructura de esta base

```
ia-db/
├── README.md                          ← este archivo (entrada única)
└── indexes/
    ├── 00_MASTER-INDEX.md             visión general, catálogo, patrones, avisos
    ├── 01_Camara.md … 07_Red-Conectividad.md   un índice por dominio de dispositivo
    ├── 08_App-Hibrida-Integrada.md    la app que los integra
    ├── 09_CI-CD-y-Build.md            pipelines, solución, build
    └── 10_Documentacion-Transversal.md  mapa de la prosa y los ADR
```

---

## Restricciones para el agente que consuma esta base

- **No cargues todos los índices.** Usá la navegación y traé uno o dos.
- **No confundas los dos repositorios.** El código está en `Ejemplos_Maui_Devices`; los ADR, las piezas, las guías, los análisis y esta base, en `Ejemplos_Maui_Devices.Documentacion`.
- **La prosa no se duplica acá**: los índices la enlazan. Si la prosa y el código discrepan, manda el código (las divergencias conocidas están marcadas ⚠️).
- **Si la evidencia contradice un índice, prevalece la evidencia**: corregí el índice y anotalo en el manifiesto.
- **No completes por inferencia**; lo no verificado está marcado «(no verificado)».
- **No copies credenciales**: los índices nombran secretos y mecanismos, nunca valores.
- **No modifiques el repo de código** al actualizar esta base: sólo se escribe dentro de `ia-db/`.
- El repo es **didáctico**: duplicación entre ejemplos, código comentado y proyectos «en construcción» son deliberados.

---

## Manifiesto de generación

- Generado por : `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Iniciar-Indexado.md` (Profile `/IA/PROMPTs/IA.Prompts/PromptFramework/Profiles/Knowledge-Indexing.md`)
- Invocado por : `/APLICADA/2026/EJEMPLOS/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Indexado/Crear-Indexado.md`
- Alcance      : `Ejemplos_Maui_Devices` — modo proyecto; 11 índices (maestro + cámara, QR, impresión térmica, GPS, mapas, telefonía, red, app híbrida, CI/CD, documentación transversal)
- Fuentes      : `Ejemplos_Devices/` (Camera, QR, Printer, GPS, Maps, Phone, Red, Integrada, Docs, scripts, `Ejemplos_Devices.slnx`), `.github/workflows/`, `Utilities/`, `README.md`, `CHANGELOG.md`, `.gitignore`, `vs.bat`; prosa enlazada del repo de documentación (`Ejemplos_Maui_Devices-docs/`, `Guias/`, `Analisis/`)
- Exclusiones  : `.git`, `bin`, `obj`, `.vs`, `wwwroot/lib`, binarios, imágenes y fuentes, `ia-db`, y lo ignorado por el `.gitignore` del proyecto
- Estado del origen : HEAD local `cc80ce1` (2026-10-28), árbol limpio. **Ese `main` local está 2 commits adelante de `origin/main`** (`85384c2`, `cc80ce1`: correcciones de mayúsculas de rutas en el `.slnx` y en `Docs/web-hibrida/maui-hibrido.md`), sin subir al momento de indexar
- Generado     : 2026-10-29 · Versión: 2.0
- Reemplaza    : las dos regeneraciones v1.0 del 2026-10-28 (hpdev e i7infra), conservadas en las ramas `rescate/hpdev-ia-db-2026-10-28` y `rescate/i7infra-ia-db-2026-10-28`
- Actualizar   : `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md`
