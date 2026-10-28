# 00 — Índice maestro de `Ejemplos_Maui_Devices`

> **Propósito:** visión general del repositorio en una lectura: qué es, con qué stack, qué proyectos tiene y dónde vive cada uno, qué patrones se repiten entre dominios, qué decisiones lo gobiernan y cuáles son las trampas que conviene conocer antes de abrir un índice de dominio.
>
> **Fuente primaria:** `README.md`, `Ejemplos_Devices/Ejemplos_Devices.slnx`, los 25 `.csproj` versionados y `.github/workflows/`; el resto se resume desde los índices 01–10, que tienen las citas `ruta:línea`.
>
> Rutas relativas a la raíz del repo de código. Estado indexado: `cc80ce1` (equivale a `285f6fb` más dos commits que sólo corrigen mayúsculas en rutas; ver [09 §15](09_CI-CD-y-Build.md)).

---

## §1 Qué es

| | |
|---|---|
| **Naturaleza** | Colección **didáctica** de ejemplos .NET MAUI de la cátedra (UTN), más una app híbrida que los integra. No es un producto ni una librería |
| **Idea rectora** | Un ejemplo mínimo por técnica; cada variante copia a la anterior y cambia lo justo (ADR-0001 del repo hermano) |
| **Dominios de dispositivo** | Cámara · QR · Impresión térmica Bluetooth · GPS · Mapas · Telefonía · Red |
| **Integración** | `Ejemplo_Maui_Hibrida`: `WebView` sobre un Blazor Server remoto (`https://aplicada.somee.com`) con un puente de comandos por URL que ejecuta las capacidades nativas |
| **Repo hermano** | `Ejemplos_Maui_Devices.Documentacion`: ADR, planes de análisis, CHANGELOG documental y esta ia-db. Los comentarios del código citan sus documentos |

La duplicación entre ejemplos, el código comentado y los proyectos «en construcción» son deliberados: forman parte del recorrido didáctico, no son defectos a corregir.

---

## §2 Stack

| Capa | Tecnología | Fuente |
|------|------------|--------|
| Lenguaje / runtime | C# · .NET 10 | `*.csproj` |
| UI | .NET MAUI (XAML; `MauiXamlInflator=SourceGen` en la híbrida y en la selfie) | `Ejemplo_Maui_Hibrida.csproj` |
| TFM | `net10.0-android`; `net10.0-ios` sólo cuando se compila en macOS (`IsOSPlatform('osx')`); Mapas suma `net10.0-windows10.0.19041.0` en Windows | línea 4-6 de cada `.csproj` MAUI |
| Mínimos | iOS 15.0 en todas; Android 25 salvo `Ejemplo_Maui_DirectCall` (21); `targetSdk` 36 en los manifiestos que lo declaran | `SupportedOSPlatformVersion`, `AndroidManifest.xml` |
| Web | ASP.NET Core Blazor Interactive Server (`Microsoft.NET.Sdk.Web`, net10.0) + OpenAPI + Scalar | `Integrada/Ejemplo_ws_Blazor/Ejemplo_ws_Blazor.csproj` |
| Tests | xUnit 2.9.2 sobre net10.0 con `UseMaui=true`, por linkeo de fuentes | `Integrada/Ejemplo_Maui_Hibrida.Tests/*.csproj` |
| MVVM | CommunityToolkit.Mvvm 8.4.2 (híbrida, MotorDSL_Dialog); `ViewModelBase` / `AsyncRelayCommand` propios en GPS y DirectCall | [04](04_GPS.md), [06](06_Telefonia.md) |
| CI | GitHub Actions, sólo iOS: `macos-15`, Xcode 26.0 bajado de Google Drive, .NET SDK 10.0.300, simulador, firma ad hoc, Maestro 1.41.0 en la híbrida | [09](09_CI-CD-y-Build.md) |
| Versiones centralizadas | No hay `global.json`, `Directory.Build.props`, `Directory.Packages.props` ni `nuget.config`: cada `.csproj` fija las suyas, y `Microsoft.Maui.Controls` varía entre `$(MauiVersion)` y 10.0.30/31/60/80 | [09 §13](09_CI-CD-y-Build.md) |

**NuGets por dominio**

| Dominio | Paquetes clave |
|---------|----------------|
| Cámara | CommunityToolkit.Maui.Camera 6.0.0 (6.1.0 en la híbrida), MetadataExtractor 2.9.0, SkiaSharp 3.119.x |
| QR | BarcodeScanner.Mobile.Maui 9.0.1 (BSM), BarcodeScanning.Native.Maui 3.0.4 (BSN, también en la híbrida), CameraScanner.Maui 1.8.31 (CS), CameraMaui 1.4.5 (ZN), AdamE.Google.iOS.GoogleUtilities 8.1.0.3 sólo para `iossimulator-x64` |
| Impresión | ESCPOS_NET 2.2.1 (ESC/POS crudo); MotorDsl.* 1.0.12 (`Ejemplo_MotorDSL`) y 1.0.13 (`Ejemplo_MotorDSL_Dialog` y la híbrida), siete paquetes |
| Mapas | Microsoft.Maui.Controls.Maps 10.0.40 |
| GPS · Telefonía · Red | Sin NuGets de dominio: Essentials de MAUI (`Geolocation`, `PhoneDialer`, `Connectivity`) |

---

## §3 Catálogo de proyectos

25 `.csproj` versionados: 22 apps MAUI, 1 web Blazor, 1 de tests y 1 artefacto documental. 23 figuran en la `.slnx`; quedan fuera los tests y el artefacto.

| Dominio | Proyecto | Técnica en una línea | `.slnx` | Workflow iOS | Índice |
|---------|----------|----------------------|:------:|:------------:|--------|
| Cámara | `Camera/Ejemplo_Photo_MediaPicker` | Diálogo nativo `MediaPicker.CapturePhotoAsync` detrás de un servicio | sí | no (`.bat` local) | [01](01_Camara.md) |
| Cámara | `Camera/Ejemplo_Photo_MiMediaPicker_Callback` | Página propia con `CameraView`; devuelve el path por `Action<string?>` vía Shell | sí | sí | [01](01_Camara.md) |
| Cámara | `Camera/Ejemplo_Photo_MiMediaPicker_Task` | Igual, pero devuelve por `TaskCompletionSource<Image>` | sí | sí | [01](01_Camara.md) |
| Cámara | `Camera/Ejemplo_Photo_MiMediaPicker_Callback_Normalizacion` | Callback + normalización EXIF (MetadataExtractor + SkiaSharp) | sí | sí | [01](01_Camara.md) |
| Cámara | `Camera/Ejemplo_Photo_MiMediaSelfie_Callback_Normalizacion` | Selfie: cámara frontal, máscara oval, normalización | sí | sí | [01](01_Camara.md) |
| QR | `QR/BSM.LectorQR` · `QR/BSM.LectorQR_Dialog` | BarcodeScanner.Mobile.Maui (ML Kit); simulador en x64 + Rosetta | sí | sí | [02](02_QR.md) |
| QR | `QR/BSN.LectorQR` · `QR/BSN.LectorQR_Dialog` | BarcodeScanning.Native.Maui (ML Kit / Apple Vision) — la recomendada | sí | sí | [02](02_QR.md) |
| QR | `QR/CS.LectorQR` · `QR/CS.LectorQR_Dialog` | CameraScanner.Maui | sí | sí | [02](02_QR.md) |
| QR | `QR/ZN.LectorQR` · `QR/ZN.LectorQR_Dialog` | CameraMaui (no ZXing.Net.Maui, pese al nombre) | sí | sí | [02](02_QR.md) |
| Impresión | `Printer/Ejemplo_ThermalPrinter` | ESC/POS crudo con ESCPOS_NET sobre socket RFCOMM propio | sí | no | [03](03_Impresion-Termica.md) |
| Impresión | `Printer/Ejemplo_MotorDSL` | Motor DSL: documento JSON → raster ESC/POS con controles de la librería | sí | sí | [03](03_Impresion-Termica.md) |
| Impresión | `Printer/Ejemplo_MotorDSL_Dialog` | Motor DSL + overlay MVVM + resultados tipados + catálogo `PRN-*` | sí | no | [03](03_Impresion-Termica.md) |
| GPS | `GPS/Ejemplo_Maui_GPS` | `Geolocation` + permisos normalizados + geocodificación inversa con Google | sí | sí | [04](04_GPS.md) |
| Mapas | `Maps/Ejemplo_Maui_Mapas` | Control `Map`: `MapSpan`, pines, `MapType` | sí | no | [05](05_Mapas.md) |
| Telefonía | `Phone/Ejemplo_Maui_Dialer` | `PhoneDialer.Open`, sin permisos | sí | sí | [06](06_Telefonia.md) |
| Telefonía | `Phone/Ejemplo_Maui_DirectCall` | `CALL_PHONE` + `Intent.ActionCall` en Android; marcador en iOS | sí | sí | [06](06_Telefonia.md) |
| Red | `Red/Ejemplo_Maui_Connectivity` | `Connectivity` (en construcción) | sí | no | [07](07_Red-Conectividad.md) |
| Integrada | `Integrada/Ejemplo_Maui_Hibrida` | `WebView` + puente de comandos por URL + overlays de dispositivo | sí | sí (único con `push`) | [08](08_App-Hibrida-Integrada.md) |
| Integrada | `Integrada/Ejemplo_ws_Blazor` | Backend Blazor Server: `Panel.razor` dispara los comandos | sí (sin `<Deploy/>`) | no | [08](08_App-Hibrida-Integrada.md) |
| Integrada | `Integrada/Ejemplo_Maui_Hibrida.Tests` | xUnit por linkeo de fuentes; 78 atributos `Fact`/`Theory` | **no** | **no** | [08 §10](08_App-Hibrida-Integrada.md) |
| Documental | `Printer/Ejemplo_Docs_Printer/…/ficheros_claude.ia/Thermalprintermaui.csproj` | Código generado por IA, net8.0; no es parte de la solución | no | no | [03](03_Impresion-Termica.md) |

Cada dominio tiene además una carpeta `Ejemplo_Docs_*` con su prosa; el mapa completo está en [10](10_Documentacion-Transversal.md).

**Workflows:** 18 `cd-ios-<categoría>.<Ejemplo>.yml`, que cubren 17 proyectos; el 18.º (`cd-ios-qr.Ejemplo_LectorQR_Dialog.yml`) apunta a un proyecto que ya no existe ([09 §4](09_CI-CD-y-Build.md)).

---

## §4 Arquitectura en una figura

```
                 ┌───────────────────────── Ejemplos por dominio (aislados) ─────────────────────────┐
                 │ Camera · QR · Printer · GPS · Maps · Phone · Red                                    │
                 │ cada uno: Servicio (no conoce la UI) → Resultado tipado → Overlay / Página → VM     │
                 └─────────────────────────────────────┬──────────────────────────────────────────────┘
                                                       │ se portan a LibApp.Devices.*
                                                       ▼
  Ejemplo_ws_Blazor (remoto)            Ejemplo_Maui_Hibrida
  ┌─────────────────────┐   navega   ┌───────────────────────────────────────────────────────────┐
  │ Panel.razor         │ ─────────▶ │ WebView ── Navigating ──▶ UrlCommandDispatcher             │
  │ NavigateTo(cmd URL) │            │   Plan (síncrono: e.Cancel)  →  ExecuteAsync (handler)     │
  │ DOM: resultados     │ ◀───────── │   Injection: EvaluateJavaScriptAsync en el DOM             │
  └─────────────────────┘  JS / URL  │   Substitution: re-navega con el resultado (_ts)           │
                                     │ Overlays: GPS · Llamada · Red · Impresora (StatusOverlayVM) │
                                     └───────────────────────────────────────────────────────────┘
```

Detalle del puente, del catálogo de comandos y de los dos modos de `CommandDelivery`: [08 §2–§5](08_App-Hibrida-Integrada.md).

---

## §5 Patrones que se repiten

| Patrón | Dónde | Índice |
|--------|-------|--------|
| **Escalera didáctica**: cada variante copia a la anterior y agrega una sola idea (MediaPicker → `CameraView` → callback/Task → normalización → selfie; ESC/POS crudo → DSL → DSL con overlay; QR simple → `_Dialog`) | Camera, Printer, QR | [01 §4](01_Camara.md), [03 §1](03_Impresion-Termica.md), [02 §2](02_QR.md) |
| **Servicio que nunca lanza + resultado `abstract record`** con casos anidados (`GpsResult`, `CallResult`, `NetworkResult`, `DiscoverResult`, `PrintResult`); el VM hace `switch` en vez de `try/catch` | GPS, Phone, Printer, Híbrida | [04 §4](04_GPS.md), [06 §4](06_Telefonia.md), [03 §4](03_Impresion-Termica.md) |
| **Overlay sobre `StatusOverlayViewModel`** (None/Busy/Error + botonera `OverlayAction`), con un coordinador singleton dueño del VM y del `CancellationTokenSource` | GPS, Phone, MotorDSL_Dialog, Híbrida | [08 §6](08_App-Hibrida-Integrada.md) |
| **Catálogos de error con códigos estables por acción del usuario** (`GPS-*`, `TEL-*`, `PRN-*`); `DisplayMessage` = mensaje accionable + código para dictar a soporte | GPS, Phone, Printer | [03 §6](03_Impresion-Termica.md), [08 §6.2](08_App-Hibrida-Integrada.md) |
| **Permisos Android con `ShouldShowRationale`**: reintentable o «abrir configuración» (`AppInfo.ShowSettingsUI`); guarda de simulador iOS en cámara | Camera, GPS, Phone, Printer | [01 §8](01_Camara.md), [04 §4.1](04_GPS.md) |
| **Modal asíncrono**: `TaskCompletionSource` (QR `_Dialog`, cámara Task) o callback por `[QueryProperty]` (cámara callback) | Camera, QR, Híbrida | [01 §6](01_Camara.md), [02 §2.1](02_QR.md) |
| **Secreto por plantilla**: `ApiKeys.cs.template` versionado, `**/Services/ApiKeys.cs` en `.gitignore`, generado por `sed` desde secrets en CI | GPS | [04 §6](04_GPS.md) |
| **`.csproj` de plantilla**: 4 RIDs Android, `max-page-size=16384`, APK; iOS Release con `UseInterpreter` + `MtouchLink SdkOnly` + AOT sólo `ios-arm64` | todos los MAUI | [09 §2](09_CI-CD-y-Build.md) |
| **Workflow de plantilla** copiado por ejemplo; 17 de 18 con `push` comentado y sólo `workflow_call` sin caller | CI | [09 §4–§5](09_CI-CD-y-Build.md) |
| **Prosa en tres capas**: `Ejemplos_Devices/Docs/` (transversal), `Ejemplo_Docs_*` (dominio), README dentro de los proyectos; el CHANGELOG es la única prosa al día con el código de la híbrida | todos | [10 §2](10_Documentacion-Transversal.md) |

---

## §6 Decisiones que gobiernan el repo

Los ADR viven en el repo hermano (`Ejemplos_Maui_Devices-docs/docs/04-decisions/`); el código cita algunos en comentarios. Qué documento justifica cada porqué, en [10 §8 y §10](10_Documentacion-Transversal.md).

| ADR | Decisión | Se ve en |
|-----|----------|----------|
| 0001 | Un proyecto por técnica, autocontenido (de ahí la duplicación y el linkeo de fuentes en los tests) | todo el árbol; [08 §10](08_App-Hibrida-Integrada.md) |
| 0002 | Servicio tipado + overlay MVVM | [04](04_GPS.md), [06](06_Telefonia.md), [08 §6](08_App-Hibrida-Integrada.md) |
| 0003 | Puente WebView ↔ nativo por comandos en la URL | [08 §3](08_App-Hibrida-Integrada.md) |
| 0004 | QR: BSN recomendada para iOS | [02 §6.3](02_QR.md) |
| 0005 | Impresión con el motor DSL | [03](03_Impresion-Termica.md) |
| 0006 | CI sólo iOS ad hoc; Android local | [09](09_CI-CD-y-Build.md) |
| 0007 | Secretos fuera del repo | [04 §6](04_GPS.md) |
| 0008 | Trust anchors embebidos para el WebView de Android | [08 §9](08_App-Hibrida-Integrada.md) |
| 0009 | Ciclo de vida del overlay GPS y cancelación de la navegación (`e.Cancel` antes del primer `await`) | [08 §3.2, §5](08_App-Hibrida-Integrada.md) |

---

## §7 Trampas que conviene saber antes de empezar

Todas están verificadas contra el código; el detalle y la cita están en el índice indicado.

| # | Trampa | Índice |
|---|--------|--------|
| 1 | `Ejemplo_Maui_Hibrida.Tests` no está en la `.slnx` y ningún workflow ejecuta `dotnet test`, aunque un comentario del `.csproj` diga que «corre en CI» | [08 §10](08_App-Hibrida-Integrada.md), [09 §15](09_CI-CD-y-Build.md) |
| 2 | `Ejemplo_Maui_GPS` no compila recién clonado: falta `Services/ApiKeys.cs`, que se genera desde la plantilla | [04 §6](04_GPS.md) |
| 3 | La clave de Google Maps de `Ejemplo_Maui_Mapas` está versionada en el `AndroidManifest.xml`, contra la regla de `secret.md` | [05 §5.1, §7](05_Mapas.md) |
| 4 | Los volteos EXIF 2/4/5/7 no se aplican: `canvas.Scale` corre después de `DrawBitmap` | [01 §15 T-7](01_Camara.md) |
| 5 | El label del domicilio en GPS no se actualiza: el setter de `Domicilio` escribe `_coordenadas` | [04 §10](04_GPS.md) |
| 6 | `ZN.*` usa CameraMaui, no ZXing.Net.Maui; y los Readme de BSN/CS/ZN son copias del de BSM | [02 §8](02_QR.md) |
| 7 | En `Ejemplo_ThermalPrinter`, QR y código de barras salen como texto, y «Grande (2x)» imprime más chico | [03 §10](03_Impresion-Termica.md) |
| 8 | `PhoneDialerDevice` afirma que `tel` está en `LSApplicationQueriesSchemes`, pero el `Info.plist` de DirectCall no lo declara | [06 §6](06_Telefonia.md) |
| 9 | La simulación del CI usa `continue-on-error: true`: el job queda verde aunque la app no arranque | [09 §8, §15](09_CI-CD-y-Build.md) |
| 10 | El `push` de la híbrida sólo se dispara por cambios en `Integrada/Ejemplo_Maui_Hibrida/**`: tocar Blazor, los tests o los flujos Maestro no lo lanza | [09 §4](09_CI-CD-y-Build.md) |
| 11 | La `.slnx` todavía tiene 2 rutas rotas (`GPS/Docs_GPS/Readme.md`, `Printer/Ejemplo_Docs_Maps/Borradores/Readme.md`) y `/Docs/web-hibrida/` declarada vacía | [09 §15 #6](09_CI-CD-y-Build.md), [10 §9](10_Documentacion-Transversal.md) |
| 12 | Los documentos de `Docs/web-hibrida/` y `Docs/qr-nuget/` describen estados anteriores del código; ante una contradicción, manda el código | [10 §13](10_Documentacion-Transversal.md) |
| 13 | En la híbrida, Camera/Selfie/SendApi interpolan valores entre comillas simples en el JS inyectado (GPS y QR usan `JsonSerializer`), y el relay `sendAPI` sólo permite `geolocate.somee.com` | [08 §4, §13](08_App-Hibrida-Integrada.md) |
| 14 | `CHANGELOG.md` y `Invariantes.cs` citan el repo hermano como `Ejemplos_Maui_Devices.Documentos`; el nombre real es `.Documentacion` | [10](10_Documentacion-Transversal.md) |

---

## §8 Cómo se construye y se prueba

| Plataforma | Cómo | Índice |
|------------|------|--------|
| Android | Local: `dotnet build -f net10.0-android -t:Run` desde los `.bat` de `Ejemplos_Devices/scripts/` o Visual Studio (`vs.bat`); sin CI | [09 §12](09_CI-CD-y-Build.md) |
| iOS | CI en `macos-15`: build Release `iossimulator-arm64` (x64 + Rosetta en BSM), firma ad hoc, `.app.zip` con `retention-days: 1`, evidencia GIF (`Utilities/simular.sh`) o MP4 + Maestro (`Utilities/simular_ui.sh`) | [09 §5–§9](09_CI-CD-y-Build.md) |
| End to end | Flujos Maestro en `Utilities/end2end/<ApplicationId>.yaml` (GPS, híbrida, QR dialog); sólo los corre el workflow de la híbrida | [09 §9](09_CI-CD-y-Build.md) |
| Unitarias | `Ejemplo_Maui_Hibrida.Tests` (xUnit, fakes a mano), sólo a mano | [08 §10](08_App-Hibrida-Integrada.md) |

---

## §9 Fuentes

| Fuente | Usada en |
|--------|----------|
| `README.md` | §1 |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | §3, §7 #11 |
| `Ejemplos_Devices/**/*.csproj` (25) | §2, §3 |
| `.github/workflows/*.yml` (18) | §3, §8 |
| `Ejemplos_Maui_Devices.Documentacion/Ejemplos_Maui_Devices-docs/docs/04-decisions/*.md` (repo hermano) | §6 |
| Índices [01](01_Camara.md)–[10](10_Documentacion-Transversal.md) | §4–§8 (cada fila remite a la sección con la cita) |
