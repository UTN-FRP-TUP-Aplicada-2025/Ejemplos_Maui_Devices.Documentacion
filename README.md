# Ejemplos_Maui_Devices.Documentacion

Documentación de [`Ejemplos_Maui_Devices`](../Ejemplos_Maui_Devices/), la colección de ejemplos didácticos de acceso a dispositivos desde .NET MAUI (cámara, QR, impresión térmica, GPS, mapas, telefonía, red) más la app híbrida que los integra tras un WebView con backend Blazor.

Esta carpeta **no contiene código**: documenta el repositorio hermano y es de solo lectura respecto de él.

## Por dónde entrar

Hay **dos puertas** y sirven para cosas distintas. Elegí según lo que necesites:

| Necesito… | Voy a… | Por qué |
|---|---|---|
| **Entender** el sistema, sus decisiones y sus flujos | [`Ejemplos_Maui_Devices-docs/`](Ejemplos_Maui_Devices-docs/README.md) | Conjunto documental narrativo: visión, arquitectura C4, ADRs, piezas por dominio, runbook, onboarding |
| **Ubicar algo** en el código sin leerlo entero | [`ia-db/`](ia-db/README.md) | Índices navegables del código fuente, trazables línea por línea |
| Consultar un análisis puntual | [`Analisis/`](Analisis/) | Estudios sobre temas concretos, fuera del corpus principal |
| Ver qué cambió y cuándo | [`CHANGELOG.md`](CHANGELOG.md) | |

**Regla práctica para agentes**: recuperá contexto desde la `ia-db` **antes** de releer fuentes; usá el conjunto documental para el porqué. La `ia-db` dice *dónde está*; los docs dicen *por qué es así*.

## Atajos a lo más consultado

| Tema | Documento |
|---|---|
| Visión y alcance de la solución | [vision](Ejemplos_Maui_Devices-docs/docs/00-overview/vision.md) |
| Mapa de todas las piezas | [system-map](Ejemplos_Maui_Devices-docs/docs/00-overview/system-map.md) |
| Las 8 decisiones de arquitectura | [ADRs](Ejemplos_Maui_Devices-docs/docs/04-decisions/) |
| **Overlays de dispositivo — cómo se fundamenta el patrón** | [07-overlays-dispositivos](Ejemplos_Maui_Devices-docs/docs/01-architecture/07-overlays-dispositivos.md) |
| **Overlays — qué ve el usuario en cada estado** | [08-pantallas-por-dispositivo](Ejemplos_Maui_Devices-docs/docs/01-architecture/08-pantallas-por-dispositivo.md) |
| Integrar la web con lo nativo | [contrato del puente](Ejemplos_Maui_Devices-docs/docs/pieces/integrada/bridge-contract.md) |
| Compilar y ejecutar | [build-and-run](Ejemplos_Maui_Devices-docs/docs/07-operations/build-and-run.md) |
| Arrancar como dev nuevo | [developer-setup](Ejemplos_Maui_Devices-docs/docs/08-onboarding/developer-setup.md) |
| Qué falta o está desactualizado | [GAP-REPORT](Ejemplos_Maui_Devices-docs/GAP-REPORT.md) |

## Estado

Todo el corpus está en `status: draft`: lo generó un agente por ingeniería inversa (`origin: reverse-engineered`) y su promoción a `approved` es humana. Lo inferido lleva `confidence`; las contradicciones entre fuentes **se reportan**, no se resuelven en silencio.

⚠️ **Requiere acción humana**: hay una API key de Google Maps hardcodeada en el origen — ver [GAP-REPORT §3.1](Ejemplos_Maui_Devices-docs/GAP-REPORT.md).
