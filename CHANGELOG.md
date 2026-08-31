# Changelog

Cambios notables de la documentación de `Ejemplos_Maui_Devices` (`Ejemplos_Maui_Devices.Documentacion`).
Formato basado en [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/).

## [2026-07-23] — ia-db v1.6: los dos modos de `CommandDelivery` del GPS + namespaces `LibApp.*`

Actualización incremental de la `ia-db`. El refactor del puente que v1.5 indexó desde el árbol
de trabajo **ya está commiteado** en el origen: HEAD pasó de `@39e55e5` a **`@0ef370b`**
(«feat(hibrida): separa clasificacion de ejecucion en el puente de comandos por URL (Plan 1)»),
con contenido idéntico a lo indexado, por lo que no hizo falta reindexarlo. Sobre ese commit hay
**nuevos cambios sin commitear** (7 archivos), todos en `Ejemplos_Devices/Integrada/` → índice
[08](ia-db/indexes/08_App-Hibrida-Integrada.md); ningún ejemplo aislado se tocó (00–07, 09, 10
sin cambios). Fuera de alcance como antes: `Utilities/` (simulación end2end).

### Agregado

- **Índice 08 §2**: nota de **namespaces normalizados a la raíz `LibApp.*`** — los dos rezagados
  bajo `Ejemplo_Maui_Hibrida.LibApp.*` (`Devices/GPS/ApiRelayService.cs` y
  `UrlCommands/Handlers/PrintCommandHandler.cs`) ya migraron; sólo `MauiProgram.cs`, `App/AppShell`
  y `ViewModels/` viven en el namespace del ensamblado. Importa para el linkeo de fuentes de los
  tests (§9).
- **Índice 08 §7.3**: tabla comparativa de las **dos tarjetas GPS** de `Panel.razor`
  («Solicitar coordenadas» → `Substitution`, «Solicitar GeoPosicion» → `Injection`), más la
  **trampa de lectura** del comentario XML de `OnSolicitarCoordenadas`, que describe el modo
  contrario al que implementa.
- **`PROMPTs/Guias/Crear-Extender-Comando.md`**: tool-prompt para generar la guía rápida de
  extensión de comandos *queryparam* de `Ejemplo_Maui_Hibrida`.
- **`Guias/Guias-Desarrollador/`**: carpeta de guías de desarrollador con los marcadores
  `README.md` y `Extender-Comando.md` (aún vacíos, pendientes de generación).

### Corregido

- **Índice 08 §7.3**: v1.5 afirmaba que el camino web de GPS «pasó de inyección a re-navegación».
  No la reemplazó: **la sumó**. Los dos modos de `CommandDelivery` conviven en `Panel.razor` y el
  `div#contenidoCoordenada` volvió a estar activo para el modo `Injection`.

### Modificado

- **Índice 08 §1.1/§4.1/§4.2/§5.1/§8**: anclas de línea recalibradas — el reordenamiento de
  `using` desplazó todas las líneas de `MauiProgram.cs` ~+7 (`108-115` → `116-123`, etc.); también
  se recalibraron las de `GpsCommandHandler.cs` (`38-95` → `35-105`) y `Panel.razor`.
- **`PROMPTs/`**: renombre de carpetas a plural — `Documentacion/` → `Documentaciones/` e
  `Implementar/` → `Implementaciones/` (contenido sin cambios), más la nueva `Guias/`.
- **`ia-db/README.md` a v1.6**: nota de sincronización incremental v1.6 (2026-07-23) y manifiesto
  de generación actualizado (origen `@0ef370b` + cambios sin commitear).

Sin cambios de comportamiento en el dispatcher ni en los tests (~125, §9 intacto).

## [2026-07-23] — ia-db v1.5: refactor del puente de comandos por URL de la app híbrida

Actualización incremental de la `ia-db` contra el árbol de trabajo de `Ejemplos_Maui_Devices`
sobre `@39e55e5` (cambios **sin commitear**; HEAD sigue en `39e55e5`). Todo acotado a
`Ejemplos_Devices/Integrada/` → índice [08](ia-db/indexes/08_App-Hibrida-Integrada.md); no se
tocó ningún ejemplo aislado (00–07, 09, 10 sin cambios). El refactor separa la **clasificación**
de la URL (`Plan`, síncrono) de su **ejecución** (`ExecuteAsync`, async), introduce el enum
`CommandDelivery` (`None`/`Injection`/`Substitution`) y explicita los dos modos por URL de GPS.
Fuera de alcance como antes: `Utilities/` (simulación end2end).

### Agregado

- **Índice 08 §4.3 «Modos de entrega (`CommandDelivery`)»**: tabla `None`/`Injection`/
  `Substitution` — cómo cada comando devuelve su resultado a la web (propiedad del comando
  concreto vía `DeliveryFor(url)`, no del handler), más la **invariante `Substitution`**
  (debe re-navegar siempre; centinela `0.0/0.0` si falla el dispositivo).
- **Índice 08 §7.3**: nueva página Blazor `GeoLocalizacion.razor` (`/geolocalizacion`) y el
  cambio del camino web de GPS de inyección en `#contenidoCoordenada` a modo `Substitution`.
- **Índice 08 §9**: los **9 tests nuevos del puente** (`UrlCommandDispatcherTests` +
  ampliación de `GpsCommandHandlerTests`), el linkeo por comodín de `LibApp/UrlCommands/*.cs`
  en el `.csproj` y el caso *skip en DEBUG* del invariante de continuación.

### Modificado

- **Índice 08 §2/§3/§4.1/§4.2/§6.2**: contrato `IUrlCommandHandler` con tres *default
  interface members* (`CancelsNavigation`, `DeliveryFor(url)`, `OnMatchedSync(url)`);
  `UrlCommandDispatcher` con `Plan(url)`/`ExecuteAsync(plan,url)` (se conservan
  `DispatchAsync`/`IsCommand` por compatibilidad); `MainViewModel.Navigating` reescrito en
  fase síncrona (decide `e.Cancel` desde el plan) + fase asíncrona; nuevos `CommandDelivery.cs`
  y `UrlPlan.cs`.
- **Índice 00**: conteo de tests `116` → `~125` (árbol y tabla de proyectos).
- **`ia-db/README.md` a v1.5**: nota de sincronización incremental v1.5 (2026-07-23) y
  manifiesto de generación actualizado (origen `@39e55e5`, árbol de trabajo sin commitear).

## [2026-07-21] — ia-db v1.4: CI de la app híbrida (`push` activo + variante end2end/Maestro)

Actualización incremental de la `ia-db` contra el origen `Ejemplos_Maui_Devices @39e55e5`
(seis commits posteriores a `a994257`), **todos acotados a la técnica de simulación end2end
de la app híbrida**: no cambió código funcional de ningún ejemplo (el único `.cs` tocado,
`Integrada/Ejemplo_Maui_Hibrida/App.xaml.cs`, solo varió en espacios en blanco). El conjunto
documental (`Ejemplos_Maui_Devices-docs/`) no requirió cambios. Se mantiene la decisión de
v1.2: el detalle interno de `Utilities/simular_ui.sh` y `Utilities/end2end/*.yaml` queda
fuera del alcance indexado; solo se documenta su acoplamiento con el workflow.

### Agregado

- **Índice 09 §4.2 «Variante `Integrada` — simulación end2end con dedo virtual»**: tabla de
  las diferencias del pipeline (`SCRIPT_SIMULATOR: ./Utilities/simular_ui.sh`,
  `MAESTRO_VERSION: 1.41.0`, step de instalación de Maestro, artefacto `recorrido.mp4` +
  `debug_logs/` en vez de GIF, ausencia del step de `ffmpeg`), más el **arranque endurecido
  del simulador** (pre-warm por GUI en el build, `bootstatus -b` 240 s → `shutdown`+`erase`
  → reintento de 300 s, peor caso ~9 min) y el **pre-warm de la grabación** (lanzar app +
  espera antes de `simctl io … recordVideo`, con `extendedWaitUntil` en el flujo Maestro).
- **Índice 10 §4**: variantes de subsección observadas en el CHANGELOG del origen
  (`### Corregido`, `### Activado`) y la línea `Alcance:` que abre varias entradas.

### Corregido

- **Índice 09 §2**: la tabla afirmaba `push` **comentado** para los 18 workflows. Ahora
  distingue los 17 de camera/gps/phone/printer/qr (solo `workflow_call`) del de la híbrida,
  `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml`, con el **`push` activado** sobre `main`
  filtrado a `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/**`.
- **Índice 00**, principio «CI centrado en iOS»: la simulación por GIF ya no es universal
  (la híbrida produce video end2end con Maestro).
- **Índice 10 §4**, puntero a la entrada más reciente del CHANGELOG del origen:
  `2026-07-17` → `2026-07-18`.

### Modificado

- **`ia-db/README.md` a v1.4**: nota de sincronización incremental v1.4 (2026-07-21) y
  manifiesto de generación actualizado (origen `@39e55e5`). Sin cambios en los índices 01–08.

## [2026-07-17] — Documentación de pruebas End2End (dedo virtual)

Se documenta la técnica de prueba **end2end sobre la UI real** (Maestro «dedo virtual» +
grabación de video en el simulador iOS) que ya existía en el repositorio de código pero no
estaba reflejada en el conjunto documental. Se agrega como **puntero** en el `README.md` del
conjunto documental; las utilidades de simulación (`Utilities/`) siguen **fuera del alcance
indexado de la ia-db** (nota v1.2). No se modificó el código fuente (solo lectura). Toda
afirmación se verificó contra el origen `Ejemplos_Maui_Devices @a994257`: el workflow
`cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` (build de simulador iOS + Maestro 1.41.0 +
`./Utilities/simular_ui.sh`), `simular_ui.sh` (`simctl io recordVideo` → `recorrido.mp4`,
`maestro test -e APP_ID=… end2end/${PACKAGE_NAME}.yaml`, `debug_logs/`), `simular.sh` (legado
screenshots→GIF por `ffmpeg`) y los textos reales de `MainPage.xaml` / el botón sólo-ícono
`arrow_back` de `QRLectorPage.xaml`.

### Agregado

- **Sección «Pruebas End2End (dedo virtual)»** en `Ejemplos_Maui_Devices-docs/README.md`:
  describe la cadena CI → `Utilities/simular_ui.sh` → flujo Maestro
  `Utilities/end2end/<PACKAGE_NAME>.yaml`, la evidencia que deja (`recorrido.mp4` + `debug_logs/`),
  las tres estrategias para *generar* el flujo (barrido caja-negra por `adb`/`maestro hierarchy`,
  grabación con `maestro record`/`studio`, derivación estática desde los `Text=` de las páginas
  MAUI) y los textos reales de los cuatro botones nativos de `MainPage` de la híbrida
  (`Volver` · `Geo Pos` · `Llamar` · `Leer QR`).
- **Nota de deuda técnica** en esa sección: los flujos `…gps.yaml` y `…qr.dialog.yaml` conservan
  el contenido de la híbrida (con el texto obsoleto «Geo Posicionar»); `…integrada.hibrida.yaml`
  fue rehecho desde la UI real por Estrategia B (validación en dispositivo pendiente).

## [2026-07-17] — Armonización de overlays, primer proyecto de tests y recategorización CI

Dos lotes de trabajo posteriores a la UX de impresión, acotados a la app híbrida
(`Ejemplo_Maui_Hibrida`): los cuatro overlays de dispositivo se llevan al mismo patrón,
nace el primer proyecto de tests de la solución y el workflow CI de la híbrida se
recategoriza. La librería `MotorDsl.*` no se modificó (sigue en 1.0.13) y los ejemplos
aislados de GPS/Red/Telefonía no se tocaron.

### Agregado

- **Suite de tests `Ejemplo_Maui_Hibrida.Tests`** — *primer proyecto de tests de toda la
  solución* (índice 08 §9): 116 tests xUnit sobre `net10.0` plano que corren en CI/escritorio
  **sin emulador ni dispositivo**, viable porque los ViewModels ya no tocan la plataforma.
  Codifica los **cinco invariantes** del patrón de overlay (una pantalla por variante,
  variante siempre alcanzable, nada de mensajes crudos, un único botón primario, el VM no
  colapsa la variante). Arrancó en 34 rojos que reproducían defectos documentados y cerró
  en 116/116.
- **Dos documentos de arquitectura de overlays** (`docs/01-architecture/`):
  `07-overlays-dispositivos.md` (ARCH-OVL-001, el *porqué* del patrón y cómo construir uno)
  y `08-pantallas-por-dispositivo.md` (ARCH-OVL-002, catálogo de pantallas con los mensajes
  literales del código).
- **ADR-0009** (`docs/04-decisions/`): cierre determinista del overlay GPS (Opción A) y
  cancelación robusta del puente de navegación; decisión *de diseño*, validada en dispositivo
  real (moto_g42, Android).
- **`README.md` de la carpeta `.Documentacion`**: portada con las dos puertas de entrada
  (conjunto documental para *entender*, `ia-db` para *ubicar*), atajos a lo más consultado y
  el aviso de la API key de Google Maps hardcodeada.
- **`Analisis/Plan-Armonizacion-Overlays.md`**: plan verificable de la armonización, ya
  ejecutado (cinco invariantes cumplidos, 116 tests en verde).
- **Reorganización de `PROMPTs/`** en subcarpetas `Comportamientos/`, `Documentacion/` e
  `Implementar/`, con nuevos tool-prompts de análisis de comportamiento y de flujos end-to-end.

### Modificado

- **Armonización de los cuatro overlays** (GPS, Red, Telefonía, Impresión) al mismo patrón
  que estrenó el de impresión (índice 08 §5.1): servicios detrás de interfaces
  `I*Service` registradas por DI, `IUiDispatcher` que abstrae `MainThread`, catálogos de
  error con código `GPS-*` y `TEL-*` (espejo de `PRN-*`) y un único botón primario por
  pantalla de error. Eliminado código muerto (`case Success` inalcanzable, props sin
  consumidor) y corregido el reporte de host en el fallo de DNS.
- **Recategorización del workflow CI de la híbrida** (índice 09): `cd-ios-gps.Ejemplo_Maui_Hibrida.yml`
  → `cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml` (contenido idéntico; nueva categoría
  `Integrada`). Siguen siendo 18 workflows; GPS queda con 1.
- **ia-db actualizada a v1.2**: notas de sincronización incremental (v1.1 y v1.2), índice 00
  (árbol y tabla con el proyecto de tests), índice 08 (§5.1 armonización, §9 suite de tests)
  e índice 10 (referencia de estilo del CHANGELOG y puntero a la entrada más reciente).

## [2026-07-16] — Catálogo de errores de impresión y documento tipado

Actualización de los índices ia-db de impresión tras el rediseño del manejo de fallos
en `Ejemplo_MotorDSL_Dialog` y `Ejemplo_Maui_Hibrida`.

### Agregado

- **Catálogo de errores con código** (índice 03 §10.3): tabla de los 14 códigos
  `PRN-*` con su causa y acción primaria, más el modelo `PrintFailure`
  (`Code` · `Title` · `UserMessage` · `TechnicalMessage` · `Exception?`) y la nota de
  por qué el atasco de papel y la impresión desvanecida quedan deliberadamente sin código.
- **`DocumentResult` tipado** (índice 08 §6.1): variantes `Ok` / `NetworkError` /
  `InvalidContract` del GET del comprobante, con su tratamiento y reintentabilidad,
  reemplazando el string vacío que dejaba al usuario sin overlay hasta 30 s.
- **Nuevas variantes de `DiscoverResult`** (`PermissionRevoked`) y nota de alcanzabilidad:
  `BluetoothOff` nunca se construía porque el catch de `ThermalPrinterService` la degradaba
  a `Empty`.
- **`PRN-DEV-ABSENT` y el matiz «presente = emparejada, no encendida»** (índice 03 §10.4),
  junto con «Olvidar y emparejar otra» (`ClearDefault` + `OpenBluetoothSettings`) y los
  alias por MAC.
- **`PROMPTs/`**: cuatro tool-prompts de análisis (comportamiento del software,
  capacidades de la librería `MotorDsl.*` y su aplicación, e implementación del PoC de
  impresión en producción).

### Modificado

- **Mapeo estado → capa de overlay** (índice 03 §10.5) reescrito sobre los códigos
  `PRN-*`, con la distinción entre predeterminada y elegida que falla, y la nota del
  flag `forzarSelector: true` que evita el bucle sin salida al «Elegir otra».
- **Diagrama de secuencia del puente** (índice 08 §6.2): incorpora la capa Busy durante
  el GET, la rama `NetworkError` / `InvalidContract` y el guard de `ImprimirAsync` que
  delega siempre en el overlay, también con el render en error.
- Referencias a fuentes sin número de línea en ambos índices, para que no queden
  desactualizadas al avanzar el origen.

## [2026-07-15] — Conjunto documental inicial conforme al Marco de Documentación de Software

Documentación generada por ingeniería inversa sobre el origen `Ejemplos_Maui_Devices`
(anclada al commit `24d611d`; el origen avanzó a `fd6a1ed` durante la generación).
Todo el corpus queda en `status: draft` a la espera de revisión y promoción humana.

### Agregado

- **Índice maestro y contrato máquina-legible** en `Ejemplos_Maui_Devices-docs/`:
  `README.md` (navegación humana), `docs-manifest.yaml` (piezas, `type`, gaps),
  `GLOSSARY.md` (vocabulario controlado) y `GAP-REPORT.md` (gap documental y derivas).
- **Nivel overview** (`docs/00-overview/`): `vision.md` y `system-map.md` con el
  inventario de las 23 piezas en 8 dominios, cada una con su `type` (Marco §7).
- **Arquitectura** (`docs/01-architecture/`): contexto y contenedores C4,
  `04-runtime-views.md` (tres flujos de punta a punta con datos sintéticos) y
  `06-crosscutting.md` (seguridad, permisos y convenciones MVVM).
- **8 ADRs reconstruidos** (`docs/04-decisions/`): un proyecto por técnica, servicio
  tipado + overlay MVVM, puente WebView↔nativo por URL, QR (BSN vs. BSM), motor DSL de
  impresión, CI iOS ad-hoc, secretos fuera del repo y trust anchors del WebView.
- **Catálogo de APIs** (`docs/05-apis/catalog.md`): puente de comandos por URL,
  endpoints del backend Blazor y servicios externos.
- **Runbook de build/CI** (`docs/07-operations/build-and-run.md`) y **onboarding**
  (`docs/08-onboarding/developer-setup.md`).
- **Nueve documentos de pieza** (`docs/pieces/`): camera, qr, printer, gps, maps, phone,
  red e integrada (con `bridge-contract.md`), con snippets de procedencia verificada.
- **ia-db**: nota de sincronización con el conjunto documental derivado y su ancla de commit.

### Corregido

- **Deriva por avance del origen** (`24d611d` → `fd6a1ed`): versión de `MotorDsl.*` en
  `Ejemplo_MotorDSL_Dialog` (1.0.12 → 1.0.13) actualizada en la pieza Printer, el
  ADR-0005 y los índices ia-db 00/03 («el código manda»).
- **Readmes invertidos del dominio Maps** en el índice ia-db 05: la nota útil vive en
  `Ejemplo_Maui_Mapas/Readme.md`, no en `Ejemplo_Docs_Maps/Readme.md`.
- Verificación de finalización: 29 `doc_id` únicos, cero enlaces relativos rotos y
  frontmatter completo en todo el corpus.
