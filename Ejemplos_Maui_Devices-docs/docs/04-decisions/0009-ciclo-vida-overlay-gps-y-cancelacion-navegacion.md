---
doc_id: ADR-0009
doc_type: adr
title: Cierre determinista del overlay GPS (Opción A) y cancelación robusta del puente de navegación
version: 2.1.0
status: accepted
origin: designed
confidence: high
owner: fernandofilipuzzi
last_review: 2026-07-17
review_cycle_days: 0
audience: [dev, arquitectos]
traces: [ADR-0002, ADR-0003, PIECE-INT-001, PIECE-GPS-001]
classification: uso-interno
---

# ADR-0009: Cierre determinista del overlay GPS (Opción A) y cancelación robusta del puente de navegación

> Decisión **de diseño** (no reverse-engineered) tomada a partir del análisis de comportamiento manual `Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02.md` y **validada en dispositivo real** (moto_g42, Android). Corrige tres defectos observados en la página Blazor `/Panel` desde la app híbrida.
>
> **Nota de revisión (v2.1.0):** la v1 adoptó la *Opción B* (mantener el busy hasta `Navigated`); la validación en dispositivo la **rechazó** (colgaba el overlay), y la v2 pasó a la **Opción A** (§D3). La validación siguió mostrando que **entregar la coordenada re-navegando** recargaba la página y no la mostraba ante coordenada idéntica; la v2.1 agrega **D4** (entrega por inyección JS, igual que cámara/QR/API). Se conserva el registro del descarte de B por trazabilidad.

## Contexto

El puente de comandos por URL (Canal B, [ADR-0003](0003-puente-webview-comandos-url.md)) intercepta la navegación del `WebView` en `MainViewModel.Navigating`. **Todos** los botones de `Panel.razor` navegan con `NavigateTo(url, forceLoad: true)`, es decir, provocan una **navegación de documento completa**; si la app no la cancela, el `WebView` recarga la página. Del análisis manual surgieron tres comportamientos no deseados con causa raíz distinta:

1. **Recarga al hacer *varios clicks*** en Foto / Selfie / QR (puntos 3.b, 4.b, 6.b del análisis). Con un solo click funciona; con clicks repetidos, la captura se hace y **la página se refresca**, perdiendo el resultado inyectado en el DOM. Los logs muestran el síntoma: `TypeError: Failed to fetch` → `Connection disconnected` → reconexión del circuito SignalR, propio de una recarga de documento.
2. **Coordenadas sin separador decimal** (punto 1.a): la latitud/longitud se muestran como enteros largos (`-31.7496689` → `-317496689`).
3. **El overlay "Buscando posición GPS…" se cierra en seguida** (punto 1.b), sin acompañar la consulta asíncrona al dispositivo.

### Causas raíz

| # | Causa raíz | Evidencia (código) |
|---|---|---|
| 1 | `[RelayCommand] async Task Navigating` genera un `AsyncRelayCommand` que **por defecto no permite ejecución concurrente**. Mientras el 1er comando espera a la cámara/QR/GPS, `CanExecute==false`; el `EventToCommandBehavior` **no invoca** el 2º `Navigating` → `e.Cancel` **no se fija** → el `WebView` ejecuta la navegación real (recarga). | `MainViewModel.cs` (`Navigating`) |
| 2 | La URL se arma con `$"Latitud={s.Location.Latitude}"` usando la **cultura del dispositivo** (es-* ⇒ coma), pero Blazor enlaza `[SupplyParameterFromQuery] double` con **`InvariantCulture`** (espera punto). La coma se descarta al parsear. | `GpsCommandHandler.cs`, `Panel.razor` |
| 3 | `GpsService.ObtenerUbicacionAsync` usaba `GetLastKnownLocationAsync` (caché); si había un fix < 1 min, retornaba **instantáneo** y el overlay hacía `ShowBusy()`→`Hide()` en el mismo instante (parpadeo). | `GpsService.cs`, `GpsOverlayViewModel.cs` |

## Decisión

Se aplican tres correcciones coordinadas:

### D1 — Cancelación robusta + guard de reentrada del puente

`Navigating` se declara con `AllowConcurrentExecutions = true` y **siempre** fija `e.Cancel = true` ante una URL-comando, aun cuando ya haya un comando en curso. Un flag de reentrada (`comandoEnCurso`) descarta el despacho duplicado (el click repetido se cancela y se ignora, en vez de recargar).

```csharp
// MainViewModel.cs
private bool comandoEnCurso;

[RelayCommand(AllowConcurrentExecutions = true)]
private async Task Navigating(WebNavigatingEventArgs e)
{
    if (_dispatcher.IsCommand(e.Url))
    {
        e.Cancel = true;             // SIEMPRE cancelar la URL-comando: nunca debe recargar

        if (comandoEnCurso)          // click duplicado mientras hay un comando en curso:
        {                            // ya se canceló la navegación; se descarta el duplicado.
            IsRefreshing = false;
            return;
        }

        comandoEnCurso = true;
        try
        {
            var outcome = await _dispatcher.DispatchAsync(e.Url);
            if (outcome.NavigateTo is not null)
                Url = outcome.NavigateTo;
        }
        finally { comandoEnCurso = false; }
    }

    IsRefreshing = false;
}
```

### D2 — Coordenadas con `InvariantCulture`

`GpsCommandHandler` formatea las coordenadas con cultura invariante para que el punto decimal sobreviva a la reescritura de la URL y al parseo de Blazor.

```csharp
// GpsCommandHandler.cs
var lat = s.Location.Latitude.ToString(CultureInfo.InvariantCulture);
var lng = s.Location.Longitude.ToString(CultureInfo.InvariantCulture);

var next = url.Replace("coordenadas=coordenadas",
    $"Latitud={lat}&Longitud={lng}&",
    StringComparison.OrdinalIgnoreCase);
```

### D3 — Ciclo de vida del overlay GPS: **Opción A** (lectura fresca + cierre determinista)

Se **elimina** el atajo de última ubicación conocida y se lee siempre del dispositivo con `GetLocationAsync`. Así el `Busy` representa una espera real (soluciona 1.b) y el overlay se cierra de forma **determinista y autocontenida** en `Procesar`, sin depender de ningún evento externo.

```csharp
// GpsService.ObtenerUbicacionAsync — lectura fresca (sin GetLastKnownLocationAsync)
var request = new GeolocationRequest(GeolocationAccuracy.Medium, DefaultTimeout);
var location = await Geolocation.Default.GetLocationAsync(request, ct);
return location is null ? new GpsResult.NoSignal() : new GpsResult.Success(location);
```

```csharp
// GpsOverlayViewModel.Procesar — en éxito se oculta acá, no en Navigated
if (result is GpsResult.Success)
{
    Hide();
    return;
}
Mostrar(GpsErrorCatalog.Describir(result));
```

### D4 — Entrega del resultado GPS por **inyección JS** (no por re-navegación)

Verificado en dispositivo: entregar la coordenada **re-navegando** (`Url = next`) recargaba la página del `WebView` — parpadeo, transición "sin dato" y, cuando la coordenada era idéntica a la anterior, la página **no volvía a mostrarla** (misma URL ⇒ `WebView.Source` no dispara `PropertyChanged` ⇒ no navega). Se cambió la entrega al **mismo patrón que cámara/QR/API**: la app intercepta (`e.Cancel`, que el converter respeta al pasar la misma instancia de `WebNavigatingEventArgs`), toma el GPS con el overlay de espera visible sobre la página **estática**, y **inyecta** el texto en `element#{id}` sin recargar.

- **Backend** (`Panel.razor`): el `div` de coordenadas lleva `id="contenidoCoordenada"` y el botón navega con `param`:
  `/panel?coordenadas=coordenadas&param=contenidoCoordenada`.
- **App** (`GpsCommandHandler`): con `param` → `_bridge.RunScript("… getElementById(id).textContent = 'Latitud: …, Longitud: …' …")` y `BridgeOutcome(true, null)` (se queda). Sin `param` (botón nativo "Geo Pos") → **fallback**: re-navega con las coordenadas en la query **+ nonce** (`_ts={seq}`) que fuerza el cambio de URL aunque la coordenada sea idéntica.

> **Requiere desplegar el backend.** El `WebView` carga el sitio **remoto** (`https://aplicada.somee.com`); la inyección sólo funciona cuando `Ejemplo_ws_Blazor` con el `Panel.razor` actualizado está publicado. Hasta entonces la app usa el fallback de re-navegación+nonce (la coordenada se muestra, con recarga).

## Alternativas

- **(D3) Opción B — mantener el caché y cerrar el overlay en `Navigated`** *(implementada primero y **descartada** tras validar en dispositivo)*. En éxito el overlay permanecía en `Busy` y se cerraba recién cuando el `WebView` terminaba la re-navegación con las coordenadas, para evitar el parpadeo/hueco visual. **Falla real observada:** el cierre dependía de que `Navigated` se disparara, y el `WebView` **no** navega —ni dispara `Navigated`— cuando la URL destino es **idéntica** a la actual (p. ej., pedir otra vez la misma coordenada, o taps repetidos que producen el mismo `Latitud=X&Longitud=Y`). Resultado: el overlay quedaba **colgado para siempre en "Buscando posición GPS"** aunque la coordenada ya estuviera en el DOM. Además acoplaba el ciclo de vida del overlay a un evento del `WebView`, contra el patrón autocontenido de [ADR-0002](0002-servicio-tipado-overlay-mvvm.md). El descarte es la razón de ser de este ADR v2.
- **(D1) `AllowConcurrentExecutions = true` sin guard**: permitiría dos despachos de dispositivo simultáneos (dos cámaras) → descartada. El guard de reentrada conserva la cancelación pero serializa el efecto nativo.
- **(D1) Deshabilitar el botón en Blazor mientras procesa**: refuerzo defensivo del lado web, pero **no** resuelve la causa raíz (la fuga de recarga está en la intercepción nativa). Queda como mejora opcional.

## Consecuencias

- (+) Foto/Selfie/QR ya no recargan ante clicks repetidos: el resultado inyectado en el DOM persiste.
- (+) Las coordenadas se muestran con separador decimal correcto, independiente de la cultura del dispositivo (validado en dispositivo: `Latitud: -31.7496028, Longitud: -60.521316`).
- (+) El overlay GPS **no puede colgarse**: su cierre es determinista y autocontenido (no depende de `Navigated`). El `Busy` representa una espera real.
- (–) La lectura es **más lenta y consume más batería** que el atajo de caché (obliga a encender el receptor aunque haya un fix reciente) y falla más en interiores → cae en `NoSignal`/timeout con más frecuencia. Es aceptable para este panel de ejemplo y lo maneja el catálogo de errores (`GPS-*`).
- (–) Tras cerrar el overlay en `Procesar`, la re-navegación con las coordenadas puede producir un **breve hueco visual** mientras el `WebView` recarga. Es cosmético; se toleró a cambio de eliminar el cuelgue de la Opción B.

## Validación

- **Tests (`Ejemplo_Maui_Hibrida.Tests`, sin dispositivo, 122/122 verdes)**: (a) `GpsCommandHandlerTests` — con `param` inyecta por JS sin re-navegar y con punto decimal bajo cultura `es-AR`; sin `param` re-navega con coordenadas invariantes + nonce; el nonce hace la URL distinta en cada llamada; (b) `NavigatingReentrancyTests` — con un handler en vuelo, el 2º evento fija `e.Cancel==true` y **no** re-despacha (el assert falla con el código previo); (c) `GpsOverlayTests.Exito_oculta_el_overlay` — el éxito deja el overlay en `None`.
- **En dispositivo (moto_g42)**: build+install+launch limpios; coordenada mostrada **con separador decimal** (`-31.7496028`); la Opción B quedaba colgada en `Buscando posición GPS` (evidencia que motivó A); con el fallback de re-navegación se confirmó que una coordenada idéntica no re-mostraba el dato (motivó el nonce y D4).
- **Pendiente (requiere despliegue / interacción física)**: publicar `Ejemplo_ws_Blazor` (Panel.razor con `id`+`param`) en somee y verificar que "Tomar Coordenadas" muestra el overlay de espera e inyecta la coordenada **sin recargar**; matriz de Foto/Selfie/QR con varios clicks.

## Evidencia

- Análisis origen: `Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02.md`
- Código: `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/ViewModels/MainViewModel.cs`, `LibApp/UrlCommands/Handlers/GpsCommandHandler.cs`, `LibApp/Devices/GPS/ViewModels/GpsOverlayViewModel.cs`, `LibApp/Devices/GPS/Services/GpsService.cs`
- ia-db: [índice 08 — App híbrida integrada](../../../ia-db/indexes/08_App-Hibrida-Integrada.md)
- Relacionados: [ADR-0003 — Puente WebView por comandos URL](0003-puente-webview-comandos-url.md) · [ADR-0002 — Servicio tipado + overlay MVVM](0002-servicio-tipado-overlay-mvvm.md) · [overlays por dispositivo](../01-architecture/07-overlays-dispositivos.md)
