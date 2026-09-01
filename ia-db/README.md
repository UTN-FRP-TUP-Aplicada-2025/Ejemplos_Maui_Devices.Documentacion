# ia-db — Base de conocimiento de `Ejemplos_Maui_Devices`

> **Instrucción para IA:** este archivo es el **punto de entrada único** a la base de conocimiento del repositorio `Ejemplos_Maui_Devices`. Antes de explorar código o documentación, leé este README, ubicá el tema en la tabla de navegación y cargá **solo** el índice o los dos índices que correspondan. Cada índice referencia sus fuentes con ruta y, cuando aporta, número de línea: recurrí al archivo fuente únicamente cuando el índice resulte insuficiente. No recorras el repositorio completo.

---

## Navegación — «Necesito saber… → leo este índice»

| Necesito saber… | Índice |
|-----------------|--------|
| De qué trata el repositorio, con qué stack, qué proyectos hay, qué patrones se repiten | [00_MASTER-INDEX](indexes/00_MASTER-INDEX.md) |
| Cómo se captura una foto y cómo vuelve a la pantalla anterior; normalización EXIF; selfie | [01_Camara](indexes/01_Camara.md) |
| Qué librería de QR usar y cómo se traduce su API; el problema de ML Kit en el simulador iOS | [02_QR](indexes/02_QR.md) |
| Cómo se imprime en la térmica Bluetooth; motor DSL; catálogo de errores de impresión | [03_Impresion-Termica](indexes/03_Impresion-Termica.md) |
| Cómo se obtiene la ubicación; permisos; geocodificación inversa; manejo de API keys | [04_GPS](indexes/04_GPS.md) |
| Cómo se usa el control `Map`: pines, centrado, tipos de mapa, clave de Google | [05_Mapas](indexes/05_Mapas.md) |
| Cómo se abre el marcador o se llama directo; permiso `CALL_PHONE` | [06_Telefonia](indexes/06_Telefonia.md) |
| Estados de `Connectivity` y detección de cambios de red | [07_Red-Conectividad](indexes/07_Red-Conectividad.md) |
| Cómo la web pide capacidades nativas: puente de comandos por URL, overlays, backend Blazor, tests | [08_App-Hibrida-Integrada](indexes/08_App-Hibrida-Integrada.md) |
| Cómo compila y se simula cada ejemplo en CI; firma, artefactos, Maestro | [09_CI-CD-y-Build](indexes/09_CI-CD-y-Build.md) |
| Dónde está documentado un porqué: arquitectura híbrida, certificados SSL, Rosetta, MVVM, CHANGELOG | [10_Documentacion-Transversal](indexes/10_Documentacion-Transversal.md) |

**Atajos por pregunta frecuente**

| Pregunta | Índice · sección |
|----------|------------------|
| ¿Por qué hay 8 proyectos de QR? | [02 §1–§2](indexes/02_QR.md) |
| ¿Por qué la app cancela la navegación del WebView? | [08 §4](indexes/08_App-Hibrida-Integrada.md) |
| ¿Por qué el proyecto de GPS no compila recién clonado? | [04 §7](indexes/04_GPS.md) |
| ¿Por qué el CI instala su propio Xcode? | [09 §4](indexes/09_CI-CD-y-Build.md) |
| ¿Cómo se decide el mensaje que ve el usuario ante un error? | [00 §6.3](indexes/00_MASTER-INDEX.md) · [03 §6](indexes/03_Impresion-Termica.md) |
| ¿Qué ejemplos no tienen pipeline? | [09 §2](indexes/09_CI-CD-y-Build.md) |

---

## Resumen ejecutivo

| | |
|---|---|
| **Proyecto** | `Ejemplos_Maui_Devices` — colección **didáctica** de ejemplos .NET MAUI de la cátedra |
| **Tipo** | Repositorio de ejemplos + una app integradora; no es un producto ni una librería |
| **Stack** | C# · .NET 10 · .NET MAUI · XAML · Blazor Server · xUnit |
| **TFM** | `net10.0-android` (+ `net10.0-ios` en macOS); Android API 25, iOS 15.0 |
| **Proyectos** | 24 (22 apps MAUI, 1 web Blazor, 1 de tests); 23 en `Ejemplos_Devices.slnx` |
| **CI** | 18 workflows de GitHub Actions (build + simulador iOS en `macos-15`) |
| **Repo de documentación** | `Ejemplos_Maui_Devices.Documentacion` (aloja esta ia-db, el CHANGELOG documental y los ADR) |
| **Backend desplegado** | `https://aplicada.somee.com` |
| **Origen indexado** | commit `285f6fb` (2026-07-23), árbol limpio |

**Función principal.** Mostrar, con un ejemplo mínimo por técnica, cómo una app MAUI usa las capacidades del dispositivo — cámara, QR, impresión térmica Bluetooth, GPS, mapas, teléfono y red — y cómo esas piezas se integran en una app híbrida donde **la pantalla la pone una web remota y el nativo ejecuta las capacidades**.

**Arquitectura en una línea.** Siete dominios de ejemplos aislados, cada uno con servicio → resultado tipado → overlay, que convergen en `Ejemplo_Maui_Hibrida`: un `WebView` sobre Blazor Server remoto con un puente de comandos por URL que intercepta la navegación, ejecuta el comando nativo y devuelve el resultado al DOM.

---

## Estructura del repositorio indexado

```
Ejemplos_Maui_Devices/
├── README.md · CHANGELOG.md · .gitignore · vs.bat
├── .github/workflows/          18 pipelines CD iOS + bitácora de versiones
├── Utilities/                  scripts de simulación y flujos Maestro end2end
└── Ejemplos_Devices/           la solución (.slnx)
    ├── Camera/ QR/ Printer/    los dominios de dispositivo, un ejemplo por técnica
    ├── GPS/ Maps/ Phone/ Red/
    ├── Integrada/              la app híbrida + su backend Blazor + los tests
    ├── Docs/                   documentación transversal (arquitectura, SSL, QR, MVVM)
    └── scripts/                .bat de arranque para Windows
```

---

## Estructura de esta base

```
ia-db/
├── README.md                       ← este archivo (punto de entrada único)
└── indexes/
    ├── 00_MASTER-INDEX.md          Visión general, catálogo, patrones, decisiones
    ├── 01_Camara.md … 07_Red-Conectividad.md    Un índice por dominio de dispositivo
    ├── 08_App-Hibrida-Integrada.md La app que los integra
    ├── 09_CI-CD-y-Build.md         Pipelines, firma, simulación
    └── 10_Documentacion-Transversal.md          Mapa de la prosa del repositorio
```

---

## Restricciones para el agente que consuma esta base

- **No cargues todos los índices.** Usá la tabla de navegación y traé uno o dos.
- **No confundas los dos repositorios.** El código está en `Ejemplos_Maui_Devices`; los ADR, los planes de análisis y esta base están en `Ejemplos_Maui_Devices.Documentacion`. Los comentarios del código citan documentos del segundo.
- **Si la evidencia contradice un índice, prevalece la evidencia**: corregí el índice y anotalo en el manifiesto.
- **No completes por inferencia.** Lo no verificado está marcado como tal; los avisos ⚠️ señalan trampas de lectura y discrepancias reales entre lo que un comentario dice y lo que el código hace.
- **No modifiques el repositorio de código** al actualizar esta base: solo se escribe dentro de `ia-db/`.
- Recordá que el repositorio es **didáctico**: código comentado, duplicación entre ejemplos y proyectos «en construcción» son deliberados, no defectos a corregir.

---

## Manifiesto de generación

- Generado por : `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Iniciar-Indexado.md`
- Invocado por : `/APLICADA/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Indexado/Crear-Indexado.md`
- Alcance      : `Ejemplos_Maui_Devices` — modo proyecto (11 dominios: cámara, QR, impresión térmica, GPS, mapas, telefonía, red, app híbrida integrada, CI/CD, documentación transversal, más el índice maestro)
- Fuentes      : `Ejemplos_Devices/` (Camera, QR, Printer, GPS, Maps, Phone, Red, Integrada, Docs, scripts, `Ejemplos_Devices.slnx`), `.github/workflows/`, `Utilities/`, `README.md`, `CHANGELOG.md`, `.gitignore`, `vs.bat`
- Exclusiones  : `.git`, `bin`, `obj`, `.vs`, `Platforms/*/Resources`, `wwwroot/lib`, binarios e imágenes, artefactos de build, y lo ignorado por el `.gitignore` del proyecto
- Estado del origen : commit `285f6fb` (2026-07-23), árbol de trabajo limpio
- Generado     : 2026-08-31 · Versión: 1.0
- Actualizar   : `/IA/PROMPTs/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md`

> **Nota de regeneración (2026-08-31).** Esta base **reemplaza** a la ia-db v1.7 (2026-08-05), reconstruida desde cero por decisión del usuario. La versión anterior queda recuperable en el historial de git del repositorio de documentación. Motivo del reemplazo: v1.7 había indexado dos cambios locales sin commitear —el resubdividido de regiones de `Panel.razor` y el archivo *untracked* `Ejemplos_Devices.slnLaunch`— que desde entonces fueron revertidos; ninguna de las dos cosas existe en el árbol actual y esta base no las registra. El historial de notas de sincronización v1.1–v1.7 no se conserva acá.
