# 07 — Red y conectividad

> **Propósito:** qué hay (y qué no) en el ejemplo de conectividad: API `Connectivity`, estados de `NetworkAccess`, perfiles de conexión y detección de cambios.
> **Fuentes primarias:** `Ejemplos_Devices/Red/Ejemplo_Maui_Connectivity/`, `Ejemplos_Devices/Red/Ejemplo_Docs_Red/Readme.md`, `Ejemplos_Devices/Ejemplos_Devices.slnx:187-194`. HEAD indexado `cc80ce1`.
> La implementación completa (servicio + overlay «Sin conexión») vive en la app híbrida (`Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/Networks/`): ver [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## 1. Inventario

| Proyecto | ApplicationId | TFM / mínimos | NuGet | En `.slnx` |
|---|---|---|---|---|
| `Ejemplo_Maui_Connectivity` | `com.ejemplos.red.connectivity` (`Ejemplo_Maui_Connectivity.csproj:16`) | `net10.0-android` (+`net10.0-ios` en macOS) `:4-5`; iOS 15.0 / Android 25.0 `:23-24` | `Microsoft.Maui.Controls` `$(MauiVersion)`, `Microsoft.Extensions.Logging.Debug` 10.0.2 (`:80-81`) | sí (`Ejemplos_Devices.slnx:188`) |
| `Ejemplo_Docs_Red` (sólo prosa) | — | — | — | sí, como archivo (`.slnx:192-194`) |

## 2. API usada

| Elemento | Uso | Cita |
|---|---|---|
| `Connectivity.ConnectivityChanged` | suscripción en el constructor | `Utilities/ConnectivityTest.cs:9-10` |
| desuscripción | en el **finalizador** `~ConnectivityTest()` | `Utilities/ConnectivityTest.cs:12-13` |
| `e.NetworkAccess` | `ConstrainedInternet` → «limitado»; distinto de `Internet` → «perdido» | `Utilities/ConnectivityTest.cs:17-21` |
| `e.ConnectionProfiles` | recorre `Bluetooth`, `Cellular`, `Ethernet`, `WiFi` | `Utilities/ConnectivityTest.cs:26-44` |
| Consulta puntual `Connectivity.Current.NetworkAccess` | sólo en la prosa | `Ejemplo_Docs_Red/Readme.md:7-12` |

Salida por `Console.WriteLine`, sin UI.

## 3. Estados de `NetworkAccess` (según la prosa)

| Valor | Significado | Cita |
|---|---|---|
| `Internet` | acceso local e Internet | `Ejemplo_Docs_Red/Readme.md:15` |
| `ConstrainedInternet` | portal cautivo / acceso limitado | `:16` |
| `Local` | sólo red local | `:17` |
| `None` | sin conectividad | `:18` |
| `Unknown` | no determinable | `:19` |

## 4. Permisos

| Plataforma | Declaración | Cita |
|---|---|---|
| Android | `ACCESS_NETWORK_STATE`, `INTERNET` | `Platforms/Android/AndroidManifest.xml:10-11` |
| iOS | ninguna clave específica (no hace falta permiso para `Connectivity`) | `Platforms/iOS/Info.plist` |

## 5. Avisos (⚠️)

- ⚠️ `ConnectivityTest` no se instancia en ningún lado (sólo aparece en su archivo y en el Readme); la `MainPage` tiene un `ScrollView` vacío (`Pages/MainPage.xaml:6-8`). El ejemplo no hace nada visible al ejecutarse (B-05 del GAP-REPORT).
- ⚠️ Desuscribirse en un finalizador es poco fiable: mientras el evento estático retiene al objeto, el finalizador no corre.
- ⚠️ Namespace `Ejemplo_Maui_Conexion` (`RootNamespace` en `Ejemplo_Maui_Connectivity.csproj:8`; `MauiProgram.cs:3`) no coincide con la carpeta/proyecto.
- ⚠️ Existe `Platforms/Windows/` y propiedades Windows en el csproj (`:21,25-26`), pero los TFM no incluyen Windows (`:4-5`).
- ⚠️ `Utilities/ConnectivityTest.cs` es copia literal del bloque de `Ejemplo_Docs_Red/Readme.md:22-64` (ejemplo de la documentación oficial). `Readme.md` está en codificación no UTF-8.

## 6. Fuentes

| Ruta | Contiene |
|---|---|
| `Ejemplos_Devices/Red/Ejemplo_Maui_Connectivity/Utilities/ConnectivityTest.cs` | Suscripción a `ConnectivityChanged` |
| `Ejemplos_Devices/Red/Ejemplo_Maui_Connectivity/Platforms/Android/AndroidManifest.xml` | Permisos de red |
| `Ejemplos_Devices/Red/Ejemplo_Docs_Red/Readme.md` | Enlace a la doc MAUI, estados de `NetworkAccess`, ejemplo |

## 7. Prosa relacionada

- [Pieza red](../../Ejemplos_Maui_Devices-docs/docs/pieces/red/README.md) — estructura, ejecución, permisos, snippets, extensiones y preguntas guía.
- [Plan-Armonizacion-Overlays](../../Analisis/Plan-Armonizacion-Overlays.md) — overlay de Red de la híbrida (cumple los invariantes; pendientes P-2/P-8).
- [GAP-REPORT](../../Ejemplos_Maui_Devices-docs/GAP-REPORT.md) — B-05.
