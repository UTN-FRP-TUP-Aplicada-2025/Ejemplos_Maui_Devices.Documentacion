# 06 — Telefonía

> **Propósito:** los dos ejemplos de teléfono: abrir el marcador del sistema (sin permisos) vs llamada directa con `CALL_PHONE`, y el patrón servicio → resultado tipado → overlay MVVM que estrena `Ejemplo_Maui_DirectCall`.
> **Fuentes primarias:** `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/`, `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/`, `Ejemplos_Devices/Ejemplos_Devices.slnx:81-88`. HEAD indexado `cc80ce1`.
> CI (`cd-ios-phone.*`): [09_CI-CD-y-Build](09_CI-CD-y-Build.md). Telefonía dentro de la app híbrida (comando `phone=phone`): [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## 1. Inventario

| Proyecto | ApplicationId | TFM / mínimos | NuGet | En `.slnx` |
|---|---|---|---|---|
| `Ejemplo_Maui_Dialer` | `com.ejemplos.phone.dialer` (`Ejemplo_Maui_Dialer.csproj:20`) | `net10.0-android` (+`net10.0-ios` en macOS) `:4-5`; iOS 15.0 / Android 25.0 `:27-28` | `Microsoft.Maui.Controls` 10.0.60, `Microsoft.Extensions.Logging.Debug` 10.0.8 (`:78-79`) | sí (`Ejemplos_Devices.slnx:82`) |
| `Ejemplo_Maui_DirectCall` | `com.ejemplos.phone.directcall` (`Ejemplo_Maui_DirectCall.csproj:18`) | ídem `:4-5`; iOS 15.0 / **Android 21.0** `:25-26` | `Microsoft.Maui.Controls` `$(MauiVersion)`, `Microsoft.Extensions.Logging.Debug` 10.0.0 (`:80-81`) | sí (`.slnx:85`) |

No hay `Ejemplo_Docs_*` de teléfono en el código.

## 2. Comparación

| Aspecto | Dialer | DirectCall |
|---|---|---|
| API | `PhoneDialer.Default.IsSupported` / `.Open(numero)` (`Ejemplo_Maui_Dialer/Pages/MainPage.xaml.cs:36-38`) | Android: `Intent.ActionCall` con `tel:` (`Services/PhoneDialerDevice.cs:91-99`); iOS/MacCatalyst: `PhoneDialer.Default.Open` (`:39-55`); resto: `NotSupported` (`:57-60`) |
| Resultado para el usuario | Se abre el marcador; el usuario toca «llamar» | Android llama sin confirmación; en iOS «Success» = «se abrió el dialer» (`PhoneDialerDevice.cs:45-47`) |
| Permiso runtime | ninguno | `Permissions.Phone` (Check → Request → `ShouldShowRationale`) (`PhoneDialerDevice.cs:67-87`) |
| Manifest Android | `<queries>` `DIAL` + `tel` para que `IsSupported` detecte la app de teléfono (`Platforms/Android/AndroidManifest.xml:13-19`) | `CALL_PHONE`, `READ_PHONE_STATE` (`:13-15`) + `<queries>` `VIEW` + `tel` (`:17-22`) |
| Arquitectura | Code-behind; la página es su propio `BindingContext` (`Pages/MainPage.xaml.cs:6-24`) | MVVM + DI: `PhoneDialerDevice` → `CallCoordinator` → `CallResult` → `CallOverlayViewModel` |
| Errores | `DisplayAlertAsync` por caso (`Pages/MainPage.xaml.cs:28-52`) | Overlay con estados y botones (§3) |

## 3. DirectCall: servicio → resultado tipado → overlay

```
MainPage (DI) ─ BindingContext ─► MainPageViewModel ─ RealizarLLamadaCommand ─► CallCoordinator.CallAsync(numero)
                                         │                                      ├─ Overlay.ShowBusy()
                                         │                                      ├─ PhoneDialerDevice.CallPhoneAsync → CallResult
                                         │                                      └─ Aplicar(result) → Overlay.Show*/Hide
<controls:CallPermissionOverlayView BindingContext="{Binding CallOverlay}"/>  ◄── CallOverlayViewModel (dueño: coordinador singleton)
```

| Pieza | Rol | Cita |
|---|---|---|
| Registro DI | `PhoneDialerDevice` transient, `CallCoordinator` **singleton** (overlay compartido), VM y página transient | `MauiProgram.cs:31-46` |
| `PhoneDialerDevice` | Ejecuta la llamada; no conoce UI | `Services/PhoneDialerDevice.cs:18-62` |
| `CallResult` | `abstract record` con `Success(Numero)`, `PermissionDenied`, `PermissionDeniedPermanent`, `PermissionRestricted`, `NotSupported`, `Cancelled`, `Failure(Message)` | `Services/CallResult.cs:6-28` |
| `CallCoordinator` | Punto único de llamada; valida, cancela la anterior, guarda último número, traduce resultado a overlay en `MainThread` | `Services/CallCoordinator.cs:14-121` |
| Mapeo resultado → overlay | `switch` en `Aplicar` | `Services/CallCoordinator.cs:94-120` |
| Reintento desde overlay | reusa `_ultimoNumero` | `Services/CallCoordinator.cs:77-86` |
| Ajustes | `AppInfo.Current.ShowSettingsUI()` | `Services/CallCoordinator.cs:88-92` |
| `CallOverlayViewModel` | Estado (`IsBusy`, `IsDenied`, `CanRetry`, `CanRetryAction`, `CanOpenSettings`) + 4 comandos; `IsVisible = IsBusy || IsDenied` | `ViewModels/CallOverlayViewModel.cs:10-157` |
| `MainPageViewModel` | Sólo dispara `CallAsync`; expone `CallOverlay` | `ViewModels/MainPageViewModel.cs:7-42` |
| Overlay XAML | `ContentView` con backdrop `#E6000000` que bloquea input; paneles por `IsBusy`/`IsDenied` | `Controls/CallPermissionOverlayView.xaml:5,27-71` |
| Infra MVVM sin librerías | `ViewModelBase.SetProperty`, `AsyncRelayCommand` (evita ejecuciones concurrentes), `RelayCommand` | `ViewModels/ViewModelBase.cs:8-21`, `ViewModels/AsyncRelayCommand.cs:10-61` |

### 3.1 Tabla de estados del overlay

| `CallResult` | Método | Título | Botones visibles |
|---|---|---|---|
| (en curso) | `ShowBusy` | «Iniciando llamada...» | — |
| `PermissionDenied` | `ShowPermissionDenied(true)` | «Permiso de llamadas necesario» | Pedir permiso, Abrir configuración |
| `PermissionDeniedPermanent` | `ShowPermissionDenied(false)` | «Acceso a llamadas denegado» | Reintentar, Abrir configuración |
| `PermissionRestricted` | `ShowRestricted` | «Acceso restringido» | Reintentar, Abrir configuración |
| `NotSupported` | `ShowNotSupported` | «Función no disponible» | (sólo cerrar) |
| `Failure` | `ShowFailure` | «Error al llamar» | Reintentar |
| `Success` / `Cancelled` | `Hide` | — | — |

Citas: `ViewModels/CallOverlayViewModel.cs:95-157`. El cerrar (`CerrarOverlayCommand`) está en `:20` y el XAML `Controls/CallPermissionOverlayView.xaml:71`.

### 3.2 Permiso `CALL_PHONE` (Android)

`Permissions.Phone`: `CheckStatusAsync` → `Granted` sigue; `Restricted` corta; si no, `RequestAsync`; tras denegar, `ShouldShowRationale` distingue «puede volver a preguntar» (`PermissionDenied`) de «no volver a preguntar» (`PermissionDeniedPermanent`) (`Services/PhoneDialerDevice.cs:67-87`). Qué permisos exactos del manifest agrupa `Permissions.Phone` en MAUI: no verificado en este repo.

## 4. Permisos por plataforma

| Plataforma | Dialer | DirectCall |
|---|---|---|
| Android | sin permisos; `<queries>` DIAL/tel (`Ejemplo_Maui_Dialer/Platforms/Android/AndroidManifest.xml:14-19`) | `CALL_PHONE` `:14`, `READ_PHONE_STATE` `:15`, `<queries>` VIEW/tel `:17-22` |
| iOS | sin permiso runtime; `LSApplicationQueriesSchemes` con `whatsapp`, `tg` (no `tel`) (`Platforms/iOS/Info.plist:62-67`) | sin `LSApplicationQueriesSchemes` |

- ⚠️ `PhoneDialerDevice.cs:10-12` y `:36-38` afirman que `tel` está en `LSApplicationQueriesSchemes` «ya declarado en este proyecto»; el `Info.plist` de DirectCall no tiene esa clave (DR-03 del GAP-REPORT).
- ⚠️ Ambos `Info.plist` traen `NSCameraUsageDescription` («…tomar fotos») heredado de plantilla (`Ejemplo_Maui_Dialer/Platforms/iOS/Info.plist:59-60`; `Ejemplo_Maui_DirectCall/Platforms/iOS/Info.plist:59-60`).
- ⚠️ El `<queries>` de DirectCall declara `VIEW` (`AndroidManifest.xml:19`) aunque la llamada usa `ActionCall`; en iOS el camino `PhoneDialer` es el que necesitaría visibilidad.
- `CallOverlayViewModel` se declara «espejo del `GpsOverlayViewModel`» (`ViewModels/CallOverlayViewModel.cs:6`): ver [04_GPS](04_GPS.md). Es overlay de 1ª generación, no hereda de la base común de la híbrida.

## 5. Fuentes

| Ruta | Contiene |
|---|---|
| `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/Pages/MainPage.xaml(.cs)` | Entry enlazado + `PhoneDialer.Open` |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/Platforms/Android/AndroidManifest.xml` | `<queries>` DIAL/tel |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Services/` | `PhoneDialerDevice`, `CallCoordinator`, `CallResult` |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/ViewModels/` | VMs del overlay y la página, comandos, base MVVM |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Controls/CallPermissionOverlayView.xaml` | Overlay reutilizable |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/MauiProgram.cs` | Registro DI con lifetimes comentados |

## 6. Prosa relacionada

- [Pieza phone](../../Ejemplos_Maui_Devices-docs/docs/pieces/phone/README.md) — estructura, cómo ejecutar, semántica de `CALL_PHONE`, snippets y preguntas guía.
- [ADR-0002 servicio tipado + overlay MVVM](../../Ejemplos_Maui_Devices-docs/docs/04-decisions/0002-servicio-tipado-overlay-mvvm.md) — el patrón que DirectCall ejemplifica.
- [07-overlays-dispositivos](../../Ejemplos_Maui_Devices-docs/docs/01-architecture/07-overlays-dispositivos.md) — overlays de 1ª generación vs los de la híbrida.
- [Plan-Armonizacion-Overlays](../../Analisis/Plan-Armonizacion-Overlays.md) — armonización del overlay de Telefonía de la híbrida (códigos `TEL-*`); DirectCall queda fuera por ser 1ª generación.
- [GAP-REPORT](../../Ejemplos_Maui_Devices-docs/GAP-REPORT.md) — DR-03.
