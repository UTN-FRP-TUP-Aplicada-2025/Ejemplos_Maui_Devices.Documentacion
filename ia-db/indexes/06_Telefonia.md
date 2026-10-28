# 06 — Telefonía

> **Propósito:** mapa navegable del dominio «Telefonía» del repositorio `Ejemplos_Maui_Devices`: dos variantes que muestran el contraste entre **abrir el marcador** del sistema (`Ejemplo_Maui_Dialer`, `PhoneDialer.Default.Open`) y **llamar directo** (`Ejemplo_Maui_DirectCall`, permiso `CALL_PHONE` + `Intent.ActionCall` en Android, con overlay MVVM de permisos).
> **Fuente primaria:** `Ejemplos_Devices/Phone/**` (commit `285f6fb`). Contexto secundario: `.github/workflows/cd-ios-phone.Ejemplo_Dialer.yml`, `.github/workflows/cd-ios-phone.Ejemplo_DirectCall.yml`, `Ejemplos_Devices/Ejemplos_Devices.slnx`, `README.md`.
> Rutas relativas a la raíz del repo de código. El patrón coordinador + overlay que DirectCall replica se explica en detalle en [04_GPS](04_GPS.md) §3; la consolidación de telefonía en la app híbrida (`LibApp/Devices/Phone/`, con `CallMode`) está en [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## §1 Mapa del dominio

| Proyecto | Ruta | Técnica | Archivos clave | En `.slnx` | Workflow CI |
|---|---|---|---|---|---|
| `Ejemplo_Maui_Dialer` | `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/` | `PhoneDialer.Default.Open` desde code-behind; sin permisos runtime | `Pages/MainPage.xaml(.cs)`, `Platforms/Android/AndroidManifest.xml`, `Platforms/iOS/Info.plist` | Sí (`Ejemplos_Devices.slnx:82`) | Sí: `cd-ios-phone.Ejemplo_Dialer.yml` (solo `workflow_call`) |
| `Ejemplo_Maui_DirectCall` | `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/` | Android: `Permissions.Phone` + `Intent.ActionCall`; iOS: `PhoneDialer`; resultado tipado `CallResult` + `CallCoordinator` singleton + overlay MVVM | `Services/PhoneDialerDevice.cs`, `Services/CallCoordinator.cs`, `Services/CallResult.cs`, `ViewModels/CallOverlayViewModel.cs`, `Controls/CallPermissionOverlayView.xaml` | Sí (`Ejemplos_Devices.slnx:85`) | Sí: `cd-ios-phone.Ejemplo_DirectCall.yml` (solo `workflow_call`) |

No hay carpeta de documentación (`Ejemplo_Docs_Phone`) ni sección de Telefonía en el `README.md` raíz (`README.md:1-24`).

```
Ejemplos_Devices/Phone/
├── Ejemplo_Maui_Dialer/
│   ├── Ejemplo_Maui_Dialer.csproj
│   ├── MauiProgram.cs · App.xaml(.cs) · AppShell.xaml(.cs)
│   ├── Pages/MainPage.xaml(.cs)          ← Entry + botón; la página es su propio BindingContext
│   └── Platforms/ Android (manifest con <queries> DIAL) · iOS (Info.plist, Entitlements*)
└── Ejemplo_Maui_DirectCall/
    ├── Ejemplo_Maui_DirectCall.csproj
    ├── MauiProgram.cs                    ← DI (AddServices)
    ├── Controls/CallPermissionOverlayView.xaml(.cs)
    ├── Pages/MainPage.xaml(.cs)
    ├── Services/ CallCoordinator.cs · CallResult.cs · PhoneDialerDevice.cs
    ├── ViewModels/ AsyncRelayCommand.cs · CallOverlayViewModel.cs · MainPageViewModel.cs · ViewModelBase.cs
    └── Platforms/ Android (CALL_PHONE, READ_PHONE_STATE) · iOS (Info.plist, Entitlements*)
```

---

## §2 Stack y configuración

| Aspecto | `Ejemplo_Maui_Dialer` | `Ejemplo_Maui_DirectCall` |
|---|---|---|
| TFM | `net10.0-android` (+`net10.0-ios` en macOS) — `Ejemplo_Maui_Dialer.csproj:4-5` | ídem — `Ejemplo_Maui_DirectCall.csproj:4-5` |
| `ApplicationId` | `com.ejemplos.phone.dialer` — `.csproj:20` | `com.ejemplos.phone.directcall` — `.csproj:18` |
| `ApplicationTitle` | «Maui Dialer» — `.csproj:18` | «Llamada directa» — `.csproj:16` |
| Android mínimo | 25 — `.csproj:28` | **21** — `.csproj:26` (único del dominio) |
| iOS mínimo | 15.0 — `.csproj:27` | 15.0 — `.csproj:25` |
| `MauiXamlInflator` | `SourceGen` — `.csproj:16` | `SourceGen` — `.csproj:14` |
| `Microsoft.Maui.Controls` | **10.0.60 fijo** — `.csproj:78` | `$(MauiVersion)` — `.csproj:80` |
| `Microsoft.Extensions.Logging.Debug` | 10.0.8 — `.csproj:79` | 10.0.0 — `.csproj:81` |
| Fuentes | OpenSans — `MauiProgram.cs:15-16` | OpenSans + `MaterialIconsOutlined` (glifo `phone_disabled`) — `MauiProgram.cs:19-21` |
| DI | ninguno | `PhoneDialerDevice` transient · `CallCoordinator` singleton · `MainPageViewModel`/`MainPage` transient — `MauiProgram.cs:31-46`. Como el único consumidor de `PhoneDialerDevice` es el singleton, en la práctica vive una sola instancia (inocuo: no guarda estado) |
| MVVM | la página implementa la propiedad `Telefono` y es su propio `BindingContext` — `Pages/MainPage.xaml.cs:6-24` | `ViewModelBase` + `AsyncRelayCommand`/`RelayCommand` propios (idénticos a GPS salvo namespace) |

Ambos comparten el bloque Android (RIDs arm/arm64/x86/x64, páginas 16 KB, APK) y el bloque iOS Release (intérprete, `SdkOnly`, AOT solo `ios-arm64`) con el resto del repo (`Ejemplo_Maui_Dialer.csproj:33-64`, `Ejemplo_Maui_DirectCall.csproj:31-62`).

---

## §3 `Ejemplo_Maui_Dialer` — abrir el marcador

### §3.1 Flujo

| Paso | Qué ocurre | Fuente |
|---|---|---|
| 1 | `Entry` ligado a `Telefono` (TwoWay por defecto en `Entry.Text`); setter con guarda de igualdad + `OnPropertyChanged()` | `Pages/MainPage.xaml:10`, `Pages/MainPage.xaml.cs:6-18` |
| 2 | Botón «Abrir marcador telefónico» → `btnAbrirMarcador_Clicked` | `Pages/MainPage.xaml:12`, `Pages/MainPage.xaml.cs:26` |
| 3 | Número vacío → alerta «Ingresá un número de teléfono» | `Pages/MainPage.xaml.cs:28-32` |
| 4 | `PhoneDialer.Default.IsSupported` → `PhoneDialer.Default.Open(Telefono)`; si no, alerta | `Pages/MainPage.xaml.cs:36-43` |
| 5 | `FeatureNotSupportedException` y `Exception` → alertas | `Pages/MainPage.xaml.cs:45-52` |

El usuario confirma la llamada en el marcador del SO: la app **no** necesita permiso de llamadas.

### §3.2 Configuración por plataforma

| Plataforma | Elemento | Valor | Fuente |
|---|---|---|---|
| Android | `ACCESS_NETWORK_STATE`, `INTERNET` | declarados | `Platforms/Android/AndroidManifest.xml:10-11` |
| Android | `<queries><intent>` | `action.DIAL` + `scheme="tel"` («necesario para que `PhoneDialer.IsSupported` detecte la app de teléfono») | `Platforms/Android/AndroidManifest.xml:13-19` |
| Android | `CALL_PHONE` | **no** declarado (no hace falta para abrir el marcador) | `Platforms/Android/AndroidManifest.xml:1-21` |
| iOS | `LSApplicationQueriesSchemes` | `whatsapp`, `tg` («solo necesario si usás `CanOpenAsync` con apps de terceros») | `Platforms/iOS/Info.plist:62-67` |
| iOS | `NSCameraUsageDescription` | presente (residuo de plantilla de cámara) | `Platforms/iOS/Info.plist:59-60` |
| iOS | `CFBundleIdentifier` / versiones | `com.ejemplos.phone.dialer` · 1.0 / 1.0 | `Platforms/iOS/Info.plist:43-50` |
| iOS | Entitlements | `aps-environment=development`, `get-task-allow=true` | `Platforms/iOS/Entitlements.plist:5-9` |

---

## §4 `Ejemplo_Maui_DirectCall` — llamada directa

### §4.1 Capas

```
MainPage ──BindingContext──▶ MainPageViewModel ──▶ CallCoordinator (singleton)
   │                          RealizarLLamadaCommand      ├─ Overlay: CallOverlayViewModel
   └─ CallPermissionOverlayView ◀─Binding CallOverlay─────┤  ├─ _cts, _ultimoNumero
                                                          ▼
                                             PhoneDialerDevice (transient)
                                                #if ANDROID   Permissions.Phone + Intent.ActionCall
                                                #elif IOS||MACCATALYST  PhoneDialer.Default.Open
                                                #else         NotSupported
```

- `PhoneDialerDevice` «no conoce al overlay ni al ViewModel»; devuelve un `CallResult` (`Services/PhoneDialerDevice.cs:8-18`).
- `CallCoordinator` es el «punto único de entrada», dueño del overlay y de la cancelación, y recuerda el último número para reintentar desde el overlay (`Services/CallCoordinator.cs:5-13`).
- `MainPageViewModel` solo dispara `_calls.CallAsync(Telefono)` (`ViewModels/MainPageViewModel.cs:36-42`).
- La página enlaza el overlay con `BindingContext="{Binding CallOverlay}"` (`Pages/MainPage.xaml:24`).
- `MainPage` solo tiene el constructor `MainPage(MainPageViewModel vm)` (`Pages/MainPage.xaml.cs:14-19`) y `AppShell` la crea por `ContentTemplate="{DataTemplate pages:MainPage}"` (`AppShell.xaml:9`): depende de que Shell resuelva la página desde el contenedor DI donde está registrada (`MauiProgram.cs:43`). El Dialer, en cambio, usa constructor sin parámetros (`Ejemplo_Maui_Dialer/Pages/MainPage.xaml.cs:20-24`).

### §4.2 Secuencia «LLamar»

| Paso | Qué ocurre | Fuente |
|---|---|---|
| 1 | Botón «LLamar» → `RealizarLLamadaCommand` | `Pages/MainPage.xaml:16-18`, `ViewModels/MainPageViewModel.cs:19` |
| 2 | Número vacío → `CallResult.Failure("Ingresá un número de teléfono.")` aplicado al overlay (panel de error) | `Services/CallCoordinator.cs:33-38` |
| 3 | Guarda `_ultimoNumero`; cancela/descarta el `_cts` previo y crea uno nuevo | `Services/CallCoordinator.cs:40-46` |
| 4 | `Overlay.ShowBusy()` («Iniciando llamada...», `ActivityIndicator`) | `Services/CallCoordinator.cs:48`, `ViewModels/CallOverlayViewModel.cs:95-101` |
| 5 Android | `CheckStatusAsync<Permissions.Phone>()` → `Granted` sigue; `Restricted` corta; si no, `RequestAsync`; denegado → `ShouldShowRationale` decide `PermissionDenied` o `PermissionDeniedPermanent` | `Services/PhoneDialerDevice.cs:67-87` |
| 6 Android | `new Intent(Intent.ActionCall)` + `tel:{numero}` + `ActivityFlags.NewTask` → `Platform.CurrentActivity.StartActivity` → `Success` | `Services/PhoneDialerDevice.cs:89-100` |
| 5-6 iOS | `PhoneDialer.Default.IsSupported` → `Open(numero)` → `Success` («el dialer se abrió»; el usuario toca el botón verde) | `Services/PhoneDialerDevice.cs:39-55` |
| otras | `NotSupported` | `Services/PhoneDialerDevice.cs:57-61` |
| 7 | `Aplicar(result)` en el hilo de UI | `Services/CallCoordinator.cs:53`, `:94-120` |

### §4.3 Resultados tipados y overlay

`CallResult` (`Services/CallResult.cs:6-28`) y su efecto en el overlay (`Services/CallCoordinator.cs:94-120`, `ViewModels/CallOverlayViewModel.cs:95-157`):

| Caso | Overlay | «Pedir permiso» (`CanRetry`) | «Reintentar» (`CanRetryAction`) | «Abrir configuración» (`CanOpenSettings`) |
|---|---|---|---|---|
| `Success(Numero)` | se oculta | — | — | — |
| `PermissionDenied` | «Permiso de llamadas necesario» | sí | no | sí |
| `PermissionDeniedPermanent` | «Acceso a llamadas denegado» | no | sí | sí |
| `PermissionRestricted` | «Acceso restringido» | no | sí | sí |
| `NotSupported` | «Función no disponible» | no | no | no |
| `Cancelled` | se oculta | — | — | — |
| `Failure(Message)` | «Error al llamar» + mensaje | no | sí | no |

- «Pedir permiso» y «Reintentar» comparten el mismo callback (`ReintentarAsync` → `CallAsync(_ultimoNumero)`), con textos distintos según la situación (`ViewModels/CallOverlayViewModel.cs:14-18`, `Services/CallCoordinator.cs:77-86`).
- «Abrir configuración» → `AppInfo.Current.ShowSettingsUI()` dentro de `try/catch` vacío (`Services/CallCoordinator.cs:88-92`).
- «Cerrar» → `Hide()` (`ViewModels/CallOverlayViewModel.cs:20`, `:153-157`).
- El overlay tiene el fondo `#E6000000` en el `Grid` raíz con `InputTransparent=False` y `CascadeInputTransparent=False`, para bloquear el contenido de atrás (`Controls/CallPermissionOverlayView.xaml:5-10`, `:20-27`).

### §4.4 Configuración por plataforma

| Plataforma | Elemento | Valor | Fuente |
|---|---|---|---|
| Android | `CALL_PHONE` | declarado («Para llamada directa (`Intent.ActionCall`) → SÍ necesita permiso») | `Platforms/Android/AndroidManifest.xml:13-14` |
| Android | `READ_PHONE_STATE` | declarado | `Platforms/Android/AndroidManifest.xml:15` |
| Android | `ACCESS_NETWORK_STATE`, `INTERNET` | declarados | `Platforms/Android/AndroidManifest.xml:10-11` |
| Android | `<queries>` | `action.VIEW` + `scheme="tel"` | `Platforms/Android/AndroidManifest.xml:17-22` |
| Android | `uses-sdk` | no declarado en el manifiesto (sí en GPS y Mapas) | `Platforms/Android/AndroidManifest.xml:1-24` |
| iOS | `LSApplicationQueriesSchemes` | **no declarado** (ver §6 #1) | `Platforms/iOS/Info.plist:1-62` |
| iOS | `NSCameraUsageDescription` | presente (residuo de plantilla de cámara) | `Platforms/iOS/Info.plist:59-60` |
| iOS | `CFBundleIdentifier` / versiones | `com.ejemplos.phone.directcall` · 1.0 / 1.0 | `Platforms/iOS/Info.plist:43-50` |

Qué permisos exactos de Android pide `Permissions.Phone` de MAUI no está verificado en el repo; el manifiesto declara `CALL_PHONE` y `READ_PHONE_STATE`.

---

## §5 Build y CI

| Aspecto | Detalle | Fuente |
|---|---|---|
| Workflows | `cd-ios-phone.Ejemplo_Dialer.yml` y `cd-ios-phone.Ejemplo_DirectCall.yml` (438 líneas c/u), misma estructura que el de GPS: `macos-15`, Xcode 26.0 de Google Drive, .NET SDK 10.0.300, workloads 10.0.100, build Release `iossimulator-arm64`, firma ad-hoc, zip + GIF vía `Utilities/simular.sh` | `.github/workflows/cd-ios-phone.Ejemplo_DirectCall.yml:23-47` |
| Disparo | Solo `workflow_call`; el `push` está comentado y nada en el repo los invoca → no corren solos | `.github/workflows/cd-ios-phone.Ejemplo_Dialer.yml:3-15`, `.github/workflows/cd-ios-phone.Ejemplo_DirectCall.yml:3-15` |
| Paso `ApiKeys` | Comentado en ambos workflows (copiado del de GPS; Telefonía no usa claves). DirectCall conserva además el encabezado «Datos específicos del proyecto GPS» | `.github/workflows/cd-ios-phone.Ejemplo_DirectCall.yml:279-285`, `:33` |
| Diferencias Dialer↔DirectCall | Nombres/paquete, y `XCODE_FILE_INSTALLER` activo distinto (`Xcode_26.3_Universal.xip` en Dialer vs `Xcode_26_Universal.xip` en DirectCall) con el mismo `XCODE_VERSION` 26.0 y el mismo ID de Drive | `.github/workflows/cd-ios-phone.Ejemplo_Dialer.yml:25`, `.github/workflows/cd-ios-phone.Ejemplo_DirectCall.yml:25` |
| Android local | No hay `.bat` de lanzamiento para Telefonía (en `scripts/` solo GPS y MediaPicker) | `Ejemplos_Devices/Ejemplos_Devices.slnx:196-199` |
| Simulador iOS | El simulador no tiene telefonía; el resultado esperado del recorrido (`NotSupported` o dialer abierto) no está verificado | — |

---

## §6 Trampas y discrepancias ⚠️

| # | Trampa / discrepancia | Evidencia |
|---|---|---|
| 1 | ⚠️ **El comentario dice que `tel` ya está en `LSApplicationQueriesSchemes` del Info.plist, pero DirectCall no declara esa clave** (y el Info.plist del Dialer solo lista `whatsapp` y `tg`). Si iOS exige o no ese esquema para `PhoneDialer` no está verificado en el repo. | `Services/PhoneDialerDevice.cs:14-16`, `:40-42` vs `Ejemplo_Maui_DirectCall/Platforms/iOS/Info.plist:1-62`; `Ejemplo_Maui_Dialer/Platforms/iOS/Info.plist:63-67` |
| 2 | ⚠️ `CallResult.PermissionDenied` se documenta para «iOS: estado `.NotDetermined`», pero la rama iOS de `PhoneDialerDevice` no consulta permisos y nunca devuelve ese caso. | `Services/CallResult.cs:10-12` vs `Services/PhoneDialerDevice.cs:39-55` |
| 3 | ⚠️ En iOS «Success» significa «se abrió el dialer», no «se hizo la llamada»; en Android, «se lanzó el intent». | `Services/PhoneDialerDevice.cs:49-51`, `:95-99` |
| 4 | `CallCoordinator.Cancelar()` existe pero ningún comando ni botón lo invoca (a diferencia del GPS, que tiene «Cancelar»). | `Services/CallCoordinator.cs:75`; `ViewModels/MainPageViewModel.cs:1-43` |
| 5 | El token del comando nunca llega: `AsyncRelayCommand.Execute` pasa `CancellationToken.None` y `RealizarLLamadaAsync` no recibe `ct`. | `ViewModels/AsyncRelayCommand.cs:48`, `ViewModels/MainPageViewModel.cs:36-42` |
| 6 | El `<queries>` de DirectCall usa `action.VIEW` (`tel`), mientras que el código lanza `ACTION_CALL` y en iOS usa `PhoneDialer`; el del Dialer usa `action.DIAL`. Cada uno es una copia ajustada a mano. | `Ejemplo_Maui_DirectCall/Platforms/Android/AndroidManifest.xml:17-22`, `Ejemplo_Maui_Dialer/Platforms/Android/AndroidManifest.xml:13-19` |
| 7 | Comentario «Permisos necesarios para la captura de medios» + `NSCameraUsageDescription` en ambos Info.plist: residuo de los ejemplos de cámara; ninguna variante usa cámara. | `Ejemplo_Maui_Dialer/Platforms/iOS/Info.plist:55-60`, `Ejemplo_Maui_DirectCall/Platforms/iOS/Info.plist:55-60` |
| 8 | Versiones de NuGet desalineadas entre hermanos: Dialer fija `Microsoft.Maui.Controls` 10.0.60 y `Logging.Debug` 10.0.8; DirectCall usa `$(MauiVersion)` y `Logging.Debug` 10.0.0. | `Ejemplo_Maui_Dialer.csproj:78-79`, `Ejemplo_Maui_DirectCall.csproj:80-81` |
| 9 | DirectCall tiene Android mínimo 21 en el `.csproj` y no declara `uses-sdk`; el resto del dominio usa 25. | `Ejemplo_Maui_DirectCall.csproj:26` |
| 10 | El Dialer no valida el formato del número: lo pasa tal cual a `PhoneDialer.Default.Open`. DirectCall solo valida que no esté vacío. | `Ejemplo_Maui_Dialer/Pages/MainPage.xaml.cs:28-38`, `Services/CallCoordinator.cs:33-38` |
| 11 | ⚠️ En DirectCall el número vacío **no** abre una alerta: muestra el overlay «Error al llamar» con botón «Reintentar». Como la rama de número vacío retorna **antes** de asignar `_ultimoNumero`, «Reintentar» usa el número de la llamada **anterior**: si nunca hubo una, solo oculta el overlay; si la hubo, vuelve a llamar a ese número viejo aunque el `Entry` esté vacío. | `Services/CallCoordinator.cs:33-40`, `:77-86` |
| 12 | `CallCoordinator.AbrirAjustes` usa `AppInfo.Current.ShowSettingsUI()` y traga cualquier excepción; GPS usa `AppInfo.ShowSettingsUI()` sin `try`. | `Services/CallCoordinator.cs:88-92` vs `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/Services/LocationPermissionService.cs:43` |
| 13 | Telefonía no tiene documentación propia ni entrada en el README raíz. | `README.md:1-24` |

---

## §7 Diferencias entre variantes

| Aspecto | `Ejemplo_Maui_Dialer` | `Ejemplo_Maui_DirectCall` |
|---|---|---|
| Qué hace en Android | Abre el marcador con el número cargado | Llama directo (`ACTION_CALL`) |
| Qué hace en iOS | Abre el marcador | Abre el marcador (iOS no expone llamada directa) |
| Permiso runtime | ninguno | `Permissions.Phone` (Android) |
| Manifiesto Android | `<queries>` DIAL | `CALL_PHONE`, `READ_PHONE_STATE`, `<queries>` VIEW |
| Arquitectura | Code-behind, alertas `DisplayAlertAsync` | Servicio de dispositivo + coordinador singleton + overlay MVVM |
| Feedback | Alertas modales | Overlay con 4 botones según el caso |
| Plataformas no soportadas | Alerta | `CallResult.NotSupported` → panel «Función no disponible» |
| Equivalente en la híbrida | ver [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (`CallMode`) | ídem |

Relación con GPS: `CallOverlayViewModel` se declara «espejo del `GpsOverlayViewModel`» (`ViewModels/CallOverlayViewModel.cs:5-6`), pero agrega `CanRetryAction` y `CanOpenSettings` en lugar del `MustOpenSettings` derivado del GPS (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/GpsOverlayViewModel.cs:63`). Además, en DirectCall el overlay **se superpone** al contenido, mientras que en GPS la página oculta su contenido (`MostrandoContenido`).

---

## §8 Preguntas frecuentes

| Pregunta | Sección |
|---|---|
| ¿Cuál es la diferencia entre `PhoneDialer` y la llamada directa? | §7 |
| ¿Qué permiso necesita la llamada directa en Android y dónde se declara? | §4.4 |
| ¿Por qué el Dialer no pide permisos? | §3.1, §3.2 |
| ¿Para qué sirve el `<queries>` del manifiesto? | §3.2, §6 #6 |
| ¿Se puede llamar directo en iOS? | §4.2, §6 #3 |
| ¿Qué muestra el overlay en cada resultado y qué botones aparecen? | §4.3 |
| ¿Cómo reintenta el overlay sin el caller original? | §4.1, §4.3 |
| ¿Hace falta `LSApplicationQueriesSchemes` con `tel`? | §6 #1 |
| ¿Se puede cancelar una llamada en curso? | §6 #4, §6 #5 |
| ¿Corren los workflows de Telefonía en cada push? | §5 |
| ¿Qué versiones de NuGet usa cada variante? | §2, §6 #8 |

---

## §9 Fuentes

| Archivo | Líneas | Rol |
|---|---|---|
| `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/Ejemplo_Maui_Dialer.csproj` | 82 | TFM, ID, NuGets |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/Pages/MainPage.xaml(.cs)` | 17 / 54 | UI y `PhoneDialer` |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/Platforms/Android/AndroidManifest.xml` | 21 | `<queries>` DIAL |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/Platforms/iOS/Info.plist` | 70 | claves iOS |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Ejemplo_Maui_DirectCall.csproj` | 84 | TFM, ID, NuGets |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/MauiProgram.cs` | 47 | DI |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Services/PhoneDialerDevice.cs` | 102 | llamada por plataforma |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Services/CallCoordinator.cs` | 121 | punto único + overlay |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Services/CallResult.cs` | 28 | resultado tipado |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/ViewModels/CallOverlayViewModel.cs` | 158 | estado del overlay |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/ViewModels/MainPageViewModel.cs` | 43 | VM de página |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/ViewModels/AsyncRelayCommand.cs` · `ViewModelBase.cs` | 77 / 24 | base MVVM propia |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Controls/CallPermissionOverlayView.xaml` | 77 | overlay |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Pages/MainPage.xaml(.cs)` | 28 / 23 | UI |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Platforms/Android/AndroidManifest.xml` | 24 | `CALL_PHONE` |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/AppShell.xaml` | 11 | página raíz por `DataTemplate` |
| `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/Platforms/iOS/Info.plist` | 62 | claves iOS |
| `.github/workflows/cd-ios-phone.Ejemplo_Dialer.yml` · `cd-ios-phone.Ejemplo_DirectCall.yml` | 438 / 438 | CI iOS |
| `Ejemplos_Devices/Ejemplos_Devices.slnx` | 81-87, 211-212 | pertenencia a la solución (proyectos y workflows como `File`) |
