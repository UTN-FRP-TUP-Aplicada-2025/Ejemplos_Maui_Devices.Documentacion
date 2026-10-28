# ia-db — Base de conocimiento de `Ejemplos_Maui_Devices`

> **Instrucción para IA:** este archivo es el **punto de entrada único** a la base de conocimiento del repositorio `Ejemplos_Maui_Devices`. Antes de explorar código o documentación, leé este README, ubicá el tema en la tabla de navegación y cargá **solo** el índice o los dos índices que correspondan. Cada índice cita sus fuentes con ruta y, cuando aporta, número de línea: andá al archivo fuente únicamente si el índice no alcanza. No recorras el repositorio completo.

---

## Navegación — «Necesito saber… → leo este índice»

| Necesito saber… | Índice |
|-----------------|--------|
| De qué trata el repositorio, con qué stack, qué proyectos hay, qué patrones se repiten, qué trampas conviene conocer | [00_MASTER-INDEX](indexes/00_MASTER-INDEX.md) |
| Cómo se toma una foto o una selfie (MediaPicker o `CameraView`), cómo vuelve el resultado a la página anterior (callback o Task), la normalización EXIF, los permisos | [01_Camara](indexes/01_Camara.md) |
| Qué librería QR usar (sobre todo en iOS), cómo traducir la API entre BSM/BSN/CS/ZN, por qué ML Kit falla en el simulador arm64 | [02_QR](indexes/02_QR.md) |
| Cómo se imprime por Bluetooth en una térmica de 58 mm: ESC/POS con ESCPOS_NET, el motor `MotorDsl.*`, permisos, catálogo `PRN-*` | [03_Impresion-Termica](indexes/03_Impresion-Termica.md) |
| Cómo se lee la ubicación con `Geolocation`, permisos normalizados, overlay del GPS, geocodificación inversa, por qué no compila recién clonado | [04_GPS](indexes/04_GPS.md) |
| Cómo se usa el control `Map` (pines, centrado, `MapType`) y dónde va la clave de Google Maps | [05_Mapas](indexes/05_Mapas.md) |
| La diferencia entre abrir el marcador y llamar directo (`CALL_PHONE`), el `Info.plist`, el overlay de llamadas | [06_Telefonia](indexes/06_Telefonia.md) |
| Cómo consultar el estado de red y detectar cambios con `Connectivity` | [07_Red-Conectividad](indexes/07_Red-Conectividad.md) |
| Cómo la app híbrida hospeda la web Blazor: puente de comandos por URL, `CommandDelivery`, overlays, `LibApp.*`, `Panel.razor`, trust anchors, tests | [08_App-Hibrida-Integrada](indexes/08_App-Hibrida-Integrada.md) |
| Qué ejemplos tienen pipeline, cómo se compila, firma y simula el `.app` de iOS, qué hay en la `.slnx`, versiones por `.csproj`, build local en Android | [09_CI-CD-y-Build](indexes/09_CI-CD-y-Build.md) |
| Dónde está documentado cada porqué, qué prosa sigue vigente frente al código y qué ADR del repo hermano cita el código | [10_Documentacion-Transversal](indexes/10_Documentacion-Transversal.md) |

**Atajos por pregunta frecuente**

| Pregunta | Índice · sección |
|----------|------------------|
| ¿Por qué hay 8 proyectos de QR y cuál recomienda el repo? | [02 §1, §6.3](indexes/02_QR.md) |
| ¿Cómo se agrega un comando nuevo al puente WebView ↔ nativo? | [08 §3.1](indexes/08_App-Hibrida-Integrada.md) |
| ¿Por qué `e.Cancel` tiene que fijarse antes del primer `await`? | [08 §3.2](indexes/08_App-Hibrida-Integrada.md) |
| ¿Qué diferencia hay entre `Injection` y `Substitution` en el GPS? | [08 §5](indexes/08_App-Hibrida-Integrada.md) |
| ¿Por qué el proyecto de GPS no compila recién clonado? | [04 §6](indexes/04_GPS.md) |
| ¿Por qué el CI baja Xcode de Google Drive? | [09 §5](indexes/09_CI-CD-y-Build.md) |
| ¿Qué ejemplos tienen pipeline y cuáles no? | [00 §3](indexes/00_MASTER-INDEX.md) · [09 §4](indexes/09_CI-CD-y-Build.md) |
| ¿Se ejecutan los tests en CI? | [08 §10](indexes/08_App-Hibrida-Integrada.md) · [09 §15](indexes/09_CI-CD-y-Build.md) |
| ¿Qué código de error ve el usuario cuando falta papel? | [03 §6](indexes/03_Impresion-Termica.md) |
| ¿Se espejan bien las orientaciones EXIF 2/4/5/7? | [01 §15](indexes/01_Camara.md) |

---

## Resumen ejecutivo

| | |
|---|---|
| **Proyecto** | `Ejemplos_Maui_Devices`: colección **didáctica** de ejemplos .NET MAUI de la cátedra |
| **Tipo** | Repositorio de ejemplos + una app integradora; no es un producto ni una librería |
| **Stack** | C# · .NET 10 · .NET MAUI · XAML · Blazor Server · xUnit |
| **TFM** | `net10.0-android` (+ `net10.0-ios` al compilar en macOS); mínimos Android 25 (21 en DirectCall) e iOS 15.0 |
| **Proyectos** | 25 `.csproj`: 22 apps MAUI, 1 web Blazor, 1 de tests, 1 artefacto documental; 23 en `Ejemplos_Devices.slnx` |
| **CI** | 18 workflows de GitHub Actions, sólo iOS (simulador en `macos-15`); Android se compila y despliega en local |
| **Backend desplegado** | `https://aplicada.somee.com` |
| **Repo de documentación** | `Ejemplos_Maui_Devices.Documentacion` (aloja esta ia-db, los ADR y los planes de análisis) |
| **Origen indexado** | commit `cc80ce1` (2026-10-28), árbol limpio |

**Función principal.** Mostrar, con un ejemplo mínimo por técnica, cómo una app MAUI usa las capacidades del dispositivo (cámara, QR, impresión térmica Bluetooth, GPS, mapas, teléfono y red) y cómo esas piezas se integran en una app híbrida donde **la pantalla la pone una web remota y el nativo ejecuta las capacidades**.

**Arquitectura en una línea.** Siete dominios de ejemplos aislados, cada uno con servicio → resultado tipado → overlay, que convergen en `Ejemplo_Maui_Hibrida`: un `WebView` sobre Blazor Server remoto con un puente de comandos por URL que intercepta la navegación, ejecuta el comando nativo y devuelve el resultado al DOM.

---

## Estructura del repositorio indexado

```
Ejemplos_Maui_Devices/
├── README.md · CHANGELOG.md · .gitignore · vs.bat
├── .github/workflows/          18 pipelines iOS + Readme, bitácora de versiones y logs de análisis
├── Utilities/                  simulación (simular.sh, simular_ui.sh), flujos Maestro end2end, download_sectigo.ps1
└── Ejemplos_Devices/           la solución (Ejemplos_Devices.slnx)
    ├── Camera/ QR/ Printer/    dominios de dispositivo: un ejemplo por técnica + Ejemplo_Docs_*
    ├── GPS/ Maps/ Phone/ Red/
    ├── Integrada/              la app híbrida, su backend Blazor y los tests
    ├── Docs/                   prosa transversal (Certificados-SSL, otros, qr-nuget, web-hibrida)
    └── scripts/                .bat de arranque para Windows / Android
```

---

## Estructura de esta base

```
ia-db/
├── README.md                          ← este archivo (punto de entrada único)
└── indexes/
    ├── 00_MASTER-INDEX.md             visión general, catálogo, patrones, decisiones, trampas
    ├── 01_Camara.md … 07_Red-Conectividad.md    un índice por dominio de dispositivo
    ├── 08_App-Hibrida-Integrada.md    la app que los integra
    ├── 09_CI-CD-y-Build.md            pipelines, solución, .csproj, firma, simulación
    └── 10_Documentacion-Transversal.md          mapa de la prosa del repositorio
```

---

## Restricciones para el agente que consuma esta base

- **No cargues todos los índices.** Usá la tabla de navegación y traé uno o dos.
- **No confundas los dos repositorios.** El código está en `Ejemplos_Maui_Devices`; los ADR, los planes de análisis y esta base están en `Ejemplos_Maui_Devices.Documentacion`. Los comentarios del código citan documentos del segundo.
- **Si la evidencia contradice un índice, prevalece la evidencia**: corregí el índice y anotalo en el manifiesto.
- **No completes por inferencia.** Lo no verificado está marcado «(no verificado)»; los avisos ⚠️ señalan trampas de lectura y discrepancias reales entre lo que dice un comentario o un Readme y lo que hace el código.
- **Las citas `ruta:línea` valen para el origen indexado.** Si el repo avanzó, comprobá la línea antes de usarla.
- **No modifiques el repositorio de código** al actualizar esta base: solo se escribe dentro de `ia-db/`.
- Recordá que el repositorio es **didáctico**: código comentado, duplicación entre ejemplos y proyectos «en construcción» son deliberados, no defectos a corregir.

---

## Manifiesto de generación

- Generado por : `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Iniciar-Indexado.md`
- Invocado por : `/APLICADA/2026/EJEMPLOS/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Indexado/Crear-Indexado.md`
- Profile      : `/IA/PROMPTs/IA.Prompts/PromptFramework/Profiles/Knowledge-Indexing.md`
- Alcance      : `/APLICADA/2026/EJEMPLOS/Ejemplos_Maui_Devices`, en modo proyecto. 11 índices: el maestro más cámara, QR, impresión térmica, GPS, mapas, telefonía, red, app híbrida integrada, CI/CD y build, y documentación transversal. La taxonomía se deriva de las carpetas de `Ejemplos_Devices/`
- Fuentes      : lo versionado (`git ls-files`) en `Ejemplos_Devices/` (Camera, QR, Printer, GPS, Maps, Phone, Red, Integrada, Docs, scripts, `Ejemplos_Devices.slnx`), `.github/workflows/`, `Utilities/`, `README.md`, `CHANGELOG.md`, `.gitignore` y `vs.bat`. Como contexto de solo lectura, los ADR de `Ejemplos_Maui_Devices.Documentacion/Ejemplos_Maui_Devices-docs/docs/04-decisions/`
- Exclusiones  : `.git`, `bin`, `obj`, `.vs`, `wwwroot/lib`, `Platforms/*/Resources`, binarios e imágenes, artefactos de build, la propia `ia-db` y lo ignorado por el `.gitignore` del proyecto
- Estado del origen : commit `cc80ce1` (2026-10-28), árbol de trabajo limpio. El contenido es el de `285f6fb` (2026-07-23) más `85384c2` y `cc80ce1`, que sólo corrigen mayúsculas en 4 rutas de la `.slnx` y en 2 enlaces de `Docs/web-hibrida/maui-hibrido.md`
- Método       : un escritor por dominio, que leyó las fuentes sin consultar la base anterior, y un verificador adversarial por índice, que contrastó cada cita contra el código y corrigió en el lugar. El maestro y este README se consolidaron a partir de los dos
- Generado     : 2026-10-28 · Versión: 1.0
- Actualizar   : `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md`

> **Nota de regeneración (2026-10-28).** Esta base **reemplaza** a la v1.0 del 2026-08-31 (commit `14b22fb` del repo de documentación), reconstruida desde cero por decisión del usuario aunque el código no había cambiado. La versión anterior se puede recuperar del historial de git. Diferencias de fondo: el inventario cuenta los 25 `.csproj` (incluido el artefacto documental `Thermalprintermaui`, en net8); se registran el workflow huérfano `cd-ios-qr.Ejemplo_LectorQR_Dialog.yml` y las rutas rotas de la `.slnx`, cuatro de las cuales (las de mayúsculas) se corrigieron en el código durante esta misma sesión.
