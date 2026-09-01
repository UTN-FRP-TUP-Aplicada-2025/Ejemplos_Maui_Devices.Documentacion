# 06 — Telefonía

> **Propósito:** las dos formas de llamar desde MAUI —abrir el marcador del sistema vs. llamar directo— y por qué la segunda necesita permiso, `Intent` propio y overlay.
> **Fuente primaria:** `Ejemplos_Devices/Phone/` (2 proyectos).
> **Índices relacionados:** [00_MASTER-INDEX](00_MASTER-INDEX.md) · [04_GPS](04_GPS.md) (mismo patrón coordinador/overlay) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## 1. Los dos proyectos

| Proyecto | `ApplicationId` | Qué hace | Tamaño |
|----------|-----------------|----------|--------|
| `Ejemplo_Maui_Dialer` | `com.ejemplos.phone.dialer` | **Abre el marcador** del sistema con el número precargado; el usuario toca «llamar» | 54 líneas, todo en la página |
| `Ejemplo_Maui_DirectCall` | `com.ejemplos.phone.directcall` | **Inicia la llamada** sin intervención del usuario (Android) | 12 archivos, 4 capas |

El salto entre uno y otro es el tema del dominio: pasar de «pedirle al SO que muestre algo» a «ejercer una capacidad del dispositivo», con todo lo que eso arrastra — permiso runtime, código nativo por plataforma, resultado tipado y UI de error.

`Ejemplo_Maui_DirectCall` es el único ejemplo con `SupportedOSPlatformVersion` de Android **21.0** (los demás usan 25.0).

## 2. `Ejemplo_Maui_Dialer` — el marcador del sistema

Todo en `Pages/MainPage.xaml.cs` (54 l.), sin DI ni ViewModel: la página es su propio `BindingContext`, con una propiedad `Telefono` que implementa la guardia clásica del binding bidireccional (`if (telefono != value)` — el comentario del código la señala como «importante! evita que entre en un bucle»).

```csharp
if (PhoneDialer.Default.IsSupported)
    PhoneDialer.Default.Open(Telefono);
```

Tres caminos de error cubiertos: número vacío, `IsSupported == false` y `FeatureNotSupportedException`, más un `catch` general. **No declara ningún permiso**: `ACCESS_NETWORK_STATE` e `INTERNET` es todo lo que tiene el manifiesto.

## 3. `Ejemplo_Maui_DirectCall` — llamada directa

Misma arquitectura de cuatro capas que el GPS ([índice 04 §2](04_GPS.md)):

```
MainPage ──▶ MainPageViewModel ──▶ CallCoordinator (singleton) ──▶ PhoneDialerDevice
                                          │                              ▼
CallPermissionOverlayView ◀── CallOverlayViewModel                   CallResult
```

Registro (`MauiProgram.cs:34-43`): `PhoneDialerDevice` **transient**, `CallCoordinator` **singleton** («para que el estado del overlay sea compartido entre todos los callers»), `MainPageViewModel` y `MainPage` transient («para no arrastrar estado entre navegaciones»).

El VM es deliberadamente mínimo (43 líneas): expone `Telefono`, `CallOverlay => _calls.Overlay` y un único comando que hace `await _calls.CallAsync(Telefono)`. Validación, permisos, overlay y errores son del coordinador.

### 3.1 `PhoneDialerDevice` — tres implementaciones por plataforma

| Plataforma | Mecanismo | Qué significa «Success» |
|------------|-----------|------------------------|
| **Android** | Permiso runtime `CALL_PHONE` + `Intent.ActionCall` con `tel:{numero}` y `ActivityFlags.NewTask` | La llamada se inició |
| **iOS / MacCatalyst** | `PhoneDialer.Default.Open(numero)` — iOS **no expone API de llamada directa** | El **dialer se abrió**; la llamada la inicia el usuario |
| Windows / otras | — | `NotSupported` |

La asimetría está documentada en el propio archivo (`:39-51`): en iOS «Success» no quiere decir que se haya llamado. El manifiesto de Android declara `CALL_PHONE` y `READ_PHONE_STATE`.

`PedirPermisoAndroidAsync` (`:67-87`) devuelve `null` si el permiso quedó concedido y, si no, el `CallResult` que corresponda: `PermissionRestricted`, o `PermissionDenied` / `PermissionDeniedPermanent` según `ShouldShowRationale`. Chequea el `CancellationToken` entre pasos.

### 3.2 `CallResult` — 7 casos

`Success(Numero)` · `PermissionDenied` (se puede reintentar) · `PermissionDeniedPermanent` (solo ajustes) · `PermissionRestricted` (MDM/control parental) · `NotSupported` (tablet, simulador) · `Cancelled` · `Failure(Message)`.

### 3.3 `CallCoordinator`

Mismo contrato que `GpsCoordinator`: punto único de entrada, dueño del overlay y del `CancellationTokenSource`, todo cambio de UI por `MainThread.InvokeOnMainThreadAsync`.

Lo propio del teléfono: **recuerda `_ultimoNumero`**, porque el botón «Pedir permiso» del overlay tiene que poder reintentar la llamada cuando el caller original ya no está (`:12-13`, `:78-86`). Si no hay número recordado, el reintento solo oculta el overlay.

`Aplicar(result)` mapea los 7 casos a cinco estados de overlay: `Hide` (Success y Cancelled), `ShowPermissionDenied(canRetry: true|false)`, `ShowRestricted`, `ShowNotSupported`, `ShowFailure(msg)`. `AbrirAjustes` envuelve `AppInfo.Current.ShowSettingsUI()` en `try/catch` porque «no todos los dispositivos lo soportan».

La validación de número vacío vive en el coordinador (`:33-38`), no en el VM: devuelve `Failure("Ingresá un número de teléfono.")` y lo refleja en el overlay.

## 4. Observaciones

- ⚠️ **Comentario que no se corresponde con el proyecto:** `PhoneDialerDevice.cs:41-42` dice que iOS «requiere "tel" en `LSApplicationQueriesSchemes` del Info.plist (**ya declarado en este proyecto**)». En `Platforms/iOS/Info.plist` **no existe** la clave `LSApplicationQueriesSchemes`; las únicas claves de uso declaradas son las estándar más `NSCameraUsageDescription`. En iOS la apertura del dialer puede fallar por eso.
- Ambos proyectos tienen workflow de CI (`cd-ios-phone.Ejemplo_Dialer.yml` y `cd-ios-phone.Ejemplo_DirectCall.yml`), aunque `Ejemplo_Maui_DirectCall` en iOS solo abre el dialer.
- `Ejemplo_Maui_Dialer` usa `Microsoft.Maui.Controls` **10.0.60** fijo; `DirectCall`, `$(MauiVersion)`.

## 5. Fuentes

| Ruta | Contenido |
|------|-----------|
| `Phone/Ejemplo_Maui_Dialer/Pages/MainPage.xaml.cs` | `PhoneDialer.Default.Open` y sus caminos de error |
| `Phone/Ejemplo_Maui_DirectCall/Services/PhoneDialerDevice.cs` | Permiso `CALL_PHONE`, `Intent.ActionCall`, degradación por plataforma |
| `Phone/Ejemplo_Maui_DirectCall/Services/CallResult.cs` | Los 7 casos del resultado |
| `Phone/Ejemplo_Maui_DirectCall/Services/CallCoordinator.cs` | Overlay, cancelación y último número |
| `Phone/Ejemplo_Maui_DirectCall/{ViewModels,Controls}/` | VM mínimo y overlay de permisos |
