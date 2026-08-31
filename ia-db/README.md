# ia-db — Ejemplos_Maui_Devices

> **Para el agente IA**: este archivo es el **punto de entrada único** a la base de conocimiento (`ia-db`) del proyecto `Ejemplos_Maui_Devices`. Antes de releer el código, consultá la tabla de navegación de abajo, cargá **solo** los índices relevantes desde [`indexes/`](indexes/) y amplía a los archivos fuente únicamente si el índice resulta insuficiente. No recorras el repositorio completo cuando existe información indexada.
>
> El código fuente indexado vive en el repositorio hermano `../../Ejemplos_Maui_Devices/`; las rutas de fuentes en los índices son **relativas a la raíz de ese proyecto** (p.ej. `Ejemplos_Devices/Camera/...`).

---

## Navegación — «Necesitas saber… → Lee este índice»

| Necesitás saber… | Leé este índice |
|------------------|-----------------|
| Qué es el repo, su stack y dónde está cada ejemplo | [00 — Índice maestro](indexes/00_MASTER-INDEX.md) |
| Capturar una foto (MediaPicker vs. picker propio, callback vs. Task, EXIF, selfie) | [01 — Cámara](indexes/01_Camara.md) |
| Escanear QR y comparar librerías (BSM · BSN · CS · ZN; página vs. diálogo) | [02 — Lectura de QR](indexes/02_QR.md) |
| Imprimir en térmica Bluetooth 58 mm (ESC/POS directo vs. motor `MotorDsl.*`) | [03 — Impresión térmica](indexes/03_Impresion-Termica.md) |
| Obtener ubicación y geocodificar (permisos, seguimiento) | [04 — GPS](indexes/04_GPS.md) |
| Mostrar un mapa con pines | [05 — Mapas](indexes/05_Mapas.md) |
| Llamar por teléfono (marcador vs. llamada directa) | [06 — Telefonía](indexes/06_Telefonia.md) |
| Detectar estado y cambios de red | [07 — Red y conectividad](indexes/07_Red-Conectividad.md) |
| Arquitectura híbrida WebView + puente de comandos por URL + backend Blazor | [08 — App híbrida integrada](indexes/08_App-Hibrida-Integrada.md) |
| Compilar/publicar en CI (iOS) y arrancar ejemplos localmente | [09 — CI/CD y build](indexes/09_CI-CD-y-Build.md) |
| SSL/certificados, notas transversales y convención de CHANGELOG | [10 — Documentación transversal](indexes/10_Documentacion-Transversal.md) |

---

## Resumen ejecutivo

| Dato | Valor |
|------|-------|
| Proyecto | `Ejemplos_Maui_Devices` |
| Tipo | Colección de ejemplos didácticos de acceso a dispositivos desde .NET MAUI |
| Stack | .NET 10 · .NET MAUI 10.x · MVVM (`CommunityToolkit.Mvvm`) |
| Targets | `net10.0-android`, `net10.0-ios` (iOS en CI/macOS) |
| Dominios | Cámara · QR · Impresión térmica · GPS · Mapas · Telefonía · Red · App híbrida integrada |
| Repo de código | `../../Ejemplos_Maui_Devices/` (hermano de esta carpeta `.Documentacion`) |
| Versión ia-db | 1.7 |

**Función principal.** Cada dispositivo de plataforma se presenta como uno o varios proyectos MAUI mínimos y comparables que contrastan enfoques de implementación; una app híbrida (`Ejemplo_Maui_Hibrida`) consolida todos los dispositivos detrás de un `WebView` con un puente de comandos por URL y un backend Blazor de apoyo.

**Arquitectura en una línea.** Ejemplos aislados por dispositivo (servicio tipado + overlay MVVM) reutilizados por una app híbrida WebView↔nativo, construidos con pipelines CI de iOS y scripts de arranque para Android.

---

## Estructura de la ia-db

```
ia-db/
├── README.md                       ← Este archivo (punto de entrada único)
└── indexes/
    ├── 00_MASTER-INDEX.md          ← Visión general, stack, catálogo de ejemplos
    ├── 01_Camara.md
    ├── 02_QR.md
    ├── 03_Impresion-Termica.md
    ├── 04_GPS.md
    ├── 05_Mapas.md
    ├── 06_Telefonia.md
    ├── 07_Red-Conectividad.md
    ├── 08_App-Hibrida-Integrada.md
    ├── 09_CI-CD-y-Build.md
    └── 10_Documentacion-Transversal.md
```

---

## Restricciones para un agente que consuma esta ia-db

- **Recuperar antes que releer.** Cargá los índices necesarios, no el repositorio completo.
- **No duplicar conocimiento.** Los índices se referencian entre sí; no copies contenido de un índice a otro.
- **El código manda.** Si la evidencia del código contradice un índice, prevalece el código: corregí el índice (ver *Actualizar*) y eliminá referencias obsoletas.
- **Trazabilidad.** Toda afirmación de un índice cita su fuente por ruta relativa al proyecto; no completes por inferencia lo no verificable.
- **No modificar el proyecto indexado.** Esta ia-db es de solo lectura respecto de `../../Ejemplos_Maui_Devices/`.

---

## Conjunto documental derivado

Sobre esta ia-db se generó el conjunto documental completo (Marco de Documentación de Software) en la carpeta hermana [`../Ejemplos_Maui_Devices-docs/`](../Ejemplos_Maui_Devices-docs/README.md): visión, mapa del sistema, arquitectura C4, vistas de runtime, 8 ADRs, catálogo de APIs, runbook, onboarding y un documento por dominio de dispositivo, más el [reporte de gap y derivas](../Ejemplos_Maui_Devices-docs/GAP-REPORT.md). Punto de entrada humano: [`../Ejemplos_Maui_Devices-docs/README.md`](../Ejemplos_Maui_Devices-docs/README.md); punto de entrada máquina: [`../Ejemplos_Maui_Devices-docs/docs-manifest.yaml`](../Ejemplos_Maui_Devices-docs/docs-manifest.yaml).

> **Nota de sincronización (`@fd6a1ed`):** el origen avanzó del commit `24d611d` (que indexó esta ia-db) a `fd6a1ed` durante la documentación. Correcciones incrementales aplicadas: versión de `MotorDsl.*` en `Ejemplo_MotorDSL_Dialog` (1.0.12 → 1.0.13, índices [00](indexes/00_MASTER-INDEX.md)/[03](indexes/03_Impresion-Termica.md)) y Readmes invertidos del dominio Maps (índice [05](indexes/05_Mapas.md)). Detalle en el [GAP-REPORT §1](../Ejemplos_Maui_Devices-docs/GAP-REPORT.md).
>
> **Actualización incremental v1.1 (2026-07-17):** dos lotes de trabajo posteriores a `fd6a1ed`, todos acotados a la app híbrida (`Ejemplo_Maui_Hibrida`) y a `Ejemplo_MotorDSL_Dialog`; **la librería `MotorDsl.*` no se modificó** (sigue en 1.0.13) y los ejemplos aislados de GPS/Red/Telefonía **no se tocaron** (índices [04](indexes/04_GPS.md)/[06](indexes/06_Telefonia.md)/[07](indexes/07_Red-Conectividad.md) sin cambios). (1) *UX de impresión* (CHANGELOG 2026-07-16): catálogo de errores con código, `DocumentResult` tipado, salida al cambiar de impresora — ya reflejado en el [índice 03 §10](indexes/03_Impresion-Termica.md). (2) *Armonización de overlays y primer proyecto de tests* (CHANGELOG 2026-07-17): los cuatro overlays de la híbrida al mismo patrón (interfaces `I*Service`, `IUiDispatcher`, catálogos `GPS-*`/`TEL-*`) y `Ejemplo_Maui_Hibrida.Tests` (116 tests xUnit, 5 invariantes) — [índice 08 §5.1 y §9](indexes/08_App-Hibrida-Integrada.md).

> **Actualización incremental v1.2 (2026-07-17):** el workflow CI de la app híbrida se **recategorizó** de `gps` a una categoría propia `Integrada`: `cd-ios-gps.Ejemplo_Maui_Hibrida.yml` → `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` (contenido idéntico; solo cambia el archivo/categoría). Sigue habiendo **18 workflows** en total; GPS queda con 1. Corregido en el [índice 09](indexes/09_CI-CD-y-Build.md) (§1 árbol, §2 convención, §3.1/§3.2 tablas) y en el [índice 10 §4](indexes/10_Documentacion-Transversal.md) (referencia de estilo del CHANGELOG y puntero a la entrada más reciente, ahora `2026-07-17`). El resto de cambios del árbol de trabajo (armonización de overlays, tests, UX de impresión) ya estaban reflejados en v1.1; las utilidades de simulación en la raíz `Utilities/` (`flows/`, `end2end/`, `simular_ui.sh`) quedan fuera del alcance indexado.

> **Actualización incremental v1.3 (2026-07-17):** las **pruebas end2end sobre la UI real** (Maestro «dedo virtual» + grabación de video en el simulador iOS: workflow `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` → `Utilities/simular_ui.sh` → `Utilities/end2end/<PACKAGE_NAME>.yaml`) **siguen fuera del alcance indexado** (nota v1.2), pero ya **no quedan sin documentar**: el conjunto documental derivado las describe como puntero en su [README (§ Pruebas End2End)](../Ejemplos_Maui_Devices-docs/README.md). Verificado contra el origen `@a994257`; detalle en el [CHANGELOG documental](../CHANGELOG.md) (entrada «Documentación de pruebas End2End»). No cambió ningún índice temático (00–10): esta nota es solo una referencia cruzada.

> **Actualización incremental v1.4 (2026-07-21):** seis commits posteriores a `a994257` (hasta `39e55e5`), **todos acotados a la técnica de simulación end2end de la app híbrida**; no cambió código funcional de ningún ejemplo (el único archivo `.cs` tocado, `Integrada/Ejemplo_Maui_Hibrida/App.xaml.cs`, solo varió en espacios en blanco). Cambios indexados en el [índice 09](indexes/09_CI-CD-y-Build.md): (1) el `push` sobre `main` del workflow `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` pasó de **comentado a activo** (filtrado a `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/**`) — corregidas las afirmaciones de §2 que daban «`push` comentado» para los 18 workflows; (2) nueva **§4.2** que documenta la variante `Integrada` del pipeline (env `SCRIPT_SIMULATOR: ./Utilities/simular_ui.sh` y `MAESTRO_VERSION: 1.41.0`, step de instalación de Maestro, artefacto `recorrido.mp4` en vez de GIF, boot del simulador por GUI con timeout/reintento y pre-warm de la grabación). En el [índice 10 §4](indexes/10_Documentacion-Transversal.md) se actualizó la entrada más reciente del CHANGELOG (`2026-07-18`) y se registraron las subsecciones `### Corregido` / `### Activado` y la línea `Alcance:`. Se mantiene la decisión de v1.2: el **detalle interno** de `Utilities/simular_ui.sh` y `Utilities/end2end/*.yaml` queda fuera del alcance indexado (solo se documenta su acoplamiento con el workflow). Sin cambios en los índices 00–08.

> **Actualización incremental v1.5 (2026-07-23):** refactor del **puente de comandos por URL** de la app híbrida (cambios en el árbol de trabajo **sin commitear**, sobre `@39e55e5`; HEAD sigue en `39e55e5`). Todo acotado a `Ejemplos_Devices/Integrada/` → índice [08](indexes/08_App-Hibrida-Integrada.md); no se tocó ningún ejemplo aislado (00–07, 09, 10 sin cambios). Cambios: (1) la clasificación de la URL se separó de la ejecución — `UrlCommandDispatcher.IsCommand`/`DispatchAsync` → `Plan(url)` (síncrono, un `UrlPlan`) + `ExecuteAsync(plan,url)`; la cancelación pasó de «es comando ⇒ cancelo» a un **OR sobre `CancelsNavigation`** de los handlers que matchean. (2) `IUrlCommandHandler` ganó tres *default interface members* (`CancelsNavigation`, `DeliveryFor(url)`, `OnMatchedSync(url)`) — los 7 handlers actuales no se editan. (3) nuevo enum `CommandDelivery` (`None`/`Injection`/`Substitution`) y record `UrlPlan`. (4) `GpsCommandHandler` explicitó **dos modos por URL**: con `param` inyecta, sin `param` **re-navega siempre** (centinela `0.0/0.0` si falla el dispositivo — antes dejaba la página congelada). (5) el camino web de GPS de `Panel.razor` pasó de inyección en `#contenidoCoordenada` a re-navegar a la **nueva página `/geolocalizacion`** (`GeoLocalizacion.razor`). (6) `MainViewModel.Navigating` reescrito en fase síncrona (decide `e.Cancel` desde el plan) + fase asíncrona. (7) 9 tests nuevos del puente (`UrlCommandDispatcherTests` + ampliación de `GpsCommandHandlerTests`; ~116→~125). Índices tocados: [08](indexes/08_App-Hibrida-Integrada.md) (§2, §3, §4.1/§4.2/§4.3 nueva, §6.2, §7.3, §8, §9) y [00](indexes/00_MASTER-INDEX.md) (conteo de tests). Fuera de alcance como antes: `Utilities/` (simulación end2end).

> **Actualización incremental v1.6 (2026-07-23):** el refactor del puente que v1.5 indexó desde el árbol de trabajo **ya está commiteado** — HEAD pasó de `39e55e5` a **`0ef370b`** («feat(hibrida): separa clasificacion de ejecucion en el puente de comandos por URL (Plan 1)»), cuyo contenido coincide con lo indexado en v1.5 (entrada `2026-07-23` del CHANGELOG); no hizo falta reindexarlo. Sobre ese commit hay **nuevos cambios sin commitear** (7 archivos), todos en `Ejemplos_Devices/Integrada/` → índice [08](indexes/08_App-Hibrida-Integrada.md); ningún ejemplo aislado se tocó (00–07, 09, 10 sin cambios) y el CHANGELOG aún no los registra. Cambios: (1) **`Panel.razor` recuperó el camino de inyección**: ahora hay **dos tarjetas GPS** que ejercitan los **dos modos de `CommandDelivery`** — «Solicitar coordenadas» → `/geolocalizacion?coordenadas=coordenadas` (`Substitution`) y la nueva «Solicitar GeoPosicion» → `/panel?…&param=contenidoCoordenada` (`Injection`, con el `div` destino ya no comentado). Esto **corrige la afirmación de v1.5** de que el camino web «pasó de inyección a re-navegación»: no la reemplazó, la sumó. Se registra además la trampa del comentario XML de `OnSolicitarCoordenadas`, que describe el modo contrario al que implementa. (2) **Namespaces normalizados a la raíz `LibApp.*`**: los dos rezagados bajo `Ejemplo_Maui_Hibrida.LibApp.*` (`Devices/GPS/ApiRelayService.cs` y `UrlCommands/Handlers/PrintCommandHandler.cs`) migraron; ya no queda ninguna referencia al prefijo viejo en `Integrada/`. (3) Reordenamiento de `using` en `MauiProgram.cs` y los handlers de GPS/QR/SendApi (cosmético) — **desplazó todas las líneas de `MauiProgram.cs` ~+7**, así que se recalibraron sus anclas en el índice 08 (§1.1, §4.1, §4.2, §5.1, §8) y las de `GpsCommandHandler.cs` y `Panel.razor`. Sin cambios de comportamiento en el dispatcher ni en los tests (~125, §9 intacto). Fuera de alcance como antes: `Utilities/` (simulación end2end).

> **Actualización incremental v1.7 (2026-08-05):** los cambios que v1.6 indexó desde el árbol de trabajo **ya están commiteados** — HEAD pasó de `0ef370b` a **`285f6fb`** («feat(hibrida): panel con ambos modos de GPS y namespaces LibApp normalizados»), cuyo contenido coincide con lo indexado en v1.6 y quedó registrado en el CHANGELOG (entrada `2026-07-23`, la más reciente): no hizo falta reindexarlo. Sobre ese commit hay **dos cambios locales**, ambos sin impacto de comportamiento: (1) `Panel.razor` sin commitear — el `<p>` de la tarjeta «Solicitar GeoPosicion» ahora describe bien su modo («…y las inyecta al DOM», antes copiaba el texto de la otra tarjeta) y el bloque `#region coordenadas` se subdividió en `page redict` / `inject en el DOM`, lo que **desplazó los métodos ~+5 líneas**; recalibradas las anclas del [índice 08 §7.1](indexes/08_App-Hibrida-Integrada.md) (`:168-182` → `:168-209`, comentario-trampa `:172-173` → `:175-176`). La trampa del comentario XML de `OnSolicitarCoordenadas` **sigue vigente**. (2) archivo *untracked* `Ejemplos_Devices/Ejemplos_Devices.slnLaunch` (perfil VS «Hibrida-Con-WS»: híbrida en Moto G42 + `Ejemplo_ws_Blazor` en `https`) → registrado en el [índice 09 §6](indexes/09_CI-CD-y-Build.md) marcado como configuración local sin versionar. Actualizada además la entrada más reciente del CHANGELOG en el [índice 10 §4](indexes/10_Documentacion-Transversal.md). Sin cambios en los índices 00–07. Fuera de alcance como antes: `Utilities/` (simulación end2end).

## Manifiesto de generación

- Generado por : /IA.Prompting.Templates/Tool-Prompts/Iniciar-Indexado.md
- Alcance      : Ejemplos_Maui_Devices (dominios: Cámara, QR, Impresión térmica, GPS, Mapas, Telefonía, Red, App híbrida integrada, CI/CD, Documentación transversal)
- Fuentes      : Ejemplos_Devices/ (Camera, QR, Printer, GPS, Maps, Phone, Red, Integrada, Docs, scripts, Ejemplos_Devices.slnLaunch), .github/workflows/, README.md, CHANGELOG.md, vs.bat, .gitignore
- Exclusiones  : .git, bin, obj, .vs, Platforms/*/Resources, wwwroot/lib, binarios/imágenes, artefactos de build, y lo ignorado por .gitignore
- Generado     : 2026-07-14 · Versión: 1.0
- Actualizado  : 2026-08-05 · Versión: 1.7 (incremental; lo indexado en v1.6 quedó commiteado en `@285f6fb`; anclas de `Panel.razor` recalibradas por el resubdivido de regiones y alta del perfil de arranque local `Ejemplos_Devices.slnLaunch` — commit `@285f6fb` + 2 cambios locales, ver nota de sincronización)
- Actualizar   : /IA/IA.Prompts/Tool-Prompts/Indexado-Documentado/Actualizar-Indexado.md
