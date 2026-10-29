# 06 — Telefonía

> **Propósito**: destilar el dominio `Phone`: las dos formas de iniciar una llamada desde .NET MAUI — abrir el marcador del sistema (`Ejemplo_Maui_Dialer`) o llamar directo (`Ejemplo_Maui_DirectCall`) —, el permiso `CALL_PHONE`, la implementación por plataforma y la arquitectura servicio → `CallResult` → overlay de DirectCall.
> **Fuente primaria**: `Ejemplos_Devices/Phone/**` (repo de código, commit `285f6fb`). Rutas relativas a la raíz de ese repo.
> **Índices relacionados**: [00_MASTER-INDEX](00_MASTER-INDEX.md) (patrón transversal servicio → resultado tipado → overlay) · [04_GPS](04_GPS.md) (`GpsOverlayViewModel`, gemelo del overlay de llamada) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (llamada de 2.ª generación en `LibApp/Devices/Phone` y `CallCommandHandler`) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) (workflows `cd-ios-phone.*`, firma con entitlements) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md) (`Docs/otros/AsyncRelayCommandOptions.md`, pieza `phone` del repo de documentación).

---

## 1. Lección del dominio en una tabla

| | `Ejemplo_Maui_Dialer` | `Ejemplo_Maui_DirectCall` |
|---|---|---|
| Qué hace | Abre el **marcador** del SO con el número precargado; el usuario toca «llamar» | **Android**: inicia la llamada sin confirmación (`Intent.ActionCall`). **iOS**: cae al marcador (no hay API de llamada directa) |
| API | `PhoneDialer.Default.Open(...)` (MAUI Essentials) | Android: `Intent.ActionCall` + `tel:` · iOS/MacCatalyst: `PhoneDialer.Default.Open(...)` |
| Permiso runtime | Ninguno | Android: `Permissions.Phone` (`CALL_PHONE` declarado) · iOS: ninguno |
| Arquitectura | Code-behind, `BindingContext = this`, `DisplayAlertAsync` | MVVM sin librerías: DI + `CallCoordinator` (singleton) + `CallResult` (record tipado) + `CallPermissionOverlayView` |
| Mensajes al usuario | Alertas modales | Overlay a pantalla completa con botones según el estado |
| Archivos de código propio | 2 (`Pages/MainPage.xaml[.cs]`) | 11 (Services, ViewModels, Controls, Pages) |
| `ApplicationId` | `com.ejemplos.phone.dialer` | `com.ejemplos.phone.directcall` |
| `ApplicationTitle` | `Maui Dialer` | `Llamada directa` |

Idea central: **delegar la confirmación al usuario elimina el permiso; automatizarla lo exige** (y por eso DirectCall necesita toda la maquinaria de permisos/overlay).

---

## 2. Árbol de fuentes (sin `Resources/` de plantilla)

```
Ejemplos_Devices/Phone/
├── Ejemplo_Maui_Dialer/
│   ├── Ejemplo_Maui_Dialer.csproj        net10.0-android (+ios en macOS), MAUI 10.0.60 fijo
│   ├── MauiProgram.cs                    sin DI propia
│   ├── App.xaml[.cs] · AppShell.xaml     Shell con una sola ruta MainPage
│   ├── Pages/MainPage.xaml[.cs]          ← todo el ejemplo
│   └── Platforms/
│       ├── Android/AndroidManifest.xml   <queries> ACTION_DIAL + tel
│       └── iOS/Info.plist · Entitlements*.plist
└── Ejemplo_Maui_DirectCall/
    ├── Ejemplo_Maui_DirectCall.csproj    MAUI $(MauiVersion)
    ├── MauiProgram.cs                    AddServices(): DI
    ├── Services/
    │   ├── PhoneDialerDevice.cs          #if ANDROID / IOS||MACCATALYST / else
    │   ├── CallCoordinator.cs            punto único de entrada, dueño del overlay
    │   └── CallResult.cs                 record abstracto con 7 casos
    ├── ViewModels/
    │   ├── ViewModelBase.cs              INotifyPropertyChanged + SetProperty<T>
    │   ├── AsyncRelayCommand.cs          AsyncRelayCommand + RelayCommand
    │   ├── CallOverlayViewModel.cs       estado y comandos del overlay
    │   └── MainPageViewModel.cs          Telefono + RealizarLLamadaCommand
    ├── Controls/CallPermissionOverlayView.xaml[.cs]
    ├── Pages/MainPage.xaml[.cs]
    ├── Resources/Fonts/MaterialIconsOutlined-Regular.otf   ícono phone_disabled
    └── Platforms/
        ├── Android/AndroidManifest.xml   CALL_PHONE + READ_PHONE_STATE, <queries> VIEW + tel
        └── iOS/Info.plist · Entitlements*.plist
```

Ambos proyectos están en `Ejemplos_Devices/Ejemplos_Devices.slnx:81-85` (carpeta `/Ejemplos/Phone/`), y sus workflows en `slnx:211-212`.

---

## 3. Configuración de proyecto (.csproj) comparada

| Propiedad | Dialer | DirectCall |
|---|---|---|
| `TargetFrameworks` | `net10.0-android`; `net10.0-ios` sólo si el host es macOS (`csproj:4-5`) | Ídem (`csproj:4-5`) |
| `SupportedOSPlatformVersion` Android | **25.0** (`Ejemplo_Maui_Dialer.csproj:28`) | **21.0** (`Ejemplo_Maui_DirectCall.csproj:26`) |
| `SupportedOSPlatformVersion` iOS | 15.0 (`:27`) | 15.0 (`:25`) |
| `MauiXamlInflator` | `SourceGen` (`:16`) | `SourceGen` (`:14`) |
| `Microsoft.Maui.Controls` | `10.0.60` fijo (`:78`) | `$(MauiVersion)` (`:80`) — la variable no se define en el repo (no hay `Directory.Build.props`); la aporta el workload MAUI |
| `Microsoft.Extensions.Logging.Debug` | `10.0.8` (`:79`) | `10.0.0` (`:81`) |
| Android | APK, sin split por ABI, 4 RIDs, `max-page-size=16384` (`:33-50`) | Ídem (`:31-48`) |
| iOS Release | Intérprete + trimming `SdkOnly`, AOT sólo `ios-arm64` (`:54-64`) | Ídem (`:52-62`) |

El 21.0 de DirectCall es el único entre los `<Project>` de `Ejemplos_Devices.slnx`: el resto declara 25.0. El otro `.csproj` con 21.0 es `Printer/Ejemplo_Docs_Printer/Prompts_Generacion_Apk_ejemplo/ficheros_claude.ia/Thermalprintermaui.csproj`, que el slnx lista sólo como `<File>` (`slnx:132`) → [03_Impresion-Termica](03_Impresion-Termica.md).

La plantilla de build es la misma del resto del repo (ver [09_CI-CD-y-Build](09_CI-CD-y-Build.md)); las diferencias de versión son de deriva entre ejemplos, no de diseño.

---

## 4. `Ejemplo_Maui_Dialer` — marcador del sistema

### 4.1 Flujo

```
Entry {Binding Telefono}  ──(BindingContext = this)──►  MainPage.Telefono
Button.Clicked ─► btnAbrirMarcador_Clicked
    ├─ vacío/blanco ............► DisplayAlertAsync("Atención", "Ingresá un número de teléfono")
    ├─ PhoneDialer.Default.IsSupported ─► PhoneDialer.Default.Open(Telefono)   (no se espera nada)
    ├─ !IsSupported ............► DisplayAlertAsync("Error", "Este dispositivo no puede hacer llamadas")
    ├─ FeatureNotSupportedException ► mismo mensaje (con punto final)
    └─ Exception ex ............► "No se pudo abrir el marcador: {ex.Message}"
```

### 4.2 Puntos del código

| Qué | Dónde |
|---|---|
| Propiedad `Telefono` con guarda anti-bucle y `OnPropertyChanged()` (comentarios didácticos) | `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/Pages/MainPage.xaml.cs:6-18` |
| La página es su propio `BindingContext` | `.../Pages/MainPage.xaml.cs:20-24` |
| Handler completo (validación, `IsSupported`, `Open`, catches) | `.../Pages/MainPage.xaml.cs:26-53` |
| UI: `Label` con ejemplo `1155551234`, `Entry`, `Button` | `.../Pages/MainPage.xaml:9-12` |
| Shell con una ruta `MainPage` (`DataTemplate`, sin DI) | `.../AppShell.xaml:10` |
| `MauiProgram` mínimo: fuentes + `AddDebug` | `.../MauiProgram.cs:8-24` |

- No hay permisos ni overlay: el SO muestra el marcador y la confirmación queda del lado del usuario.
- `Open` es sincrónico; el `async` del handler sólo sirve para las alertas.

---

## 5. `Ejemplo_Maui_DirectCall` — llamada directa

### 5.1 Capas y responsabilidades

```
MainPage (XAML)                       Grid raíz: contenido + overlay superpuesto (z-order)
  └─ BindingContext = MainPageViewModel (inyectado)
       ├─ Telefono
       ├─ RealizarLLamadaCommand ─► CallCoordinator.CallAsync(Telefono)
       └─ CallOverlay ──────────────► CallCoordinator.Overlay  (CallOverlayViewModel)
                                             ▲
CallCoordinator (singleton) ─────────────────┘ dueño del overlay, del CTS y de _ultimoNumero
  └─ PhoneDialerDevice (transient) ─► CallResult (record tipado)
```

| Registro DI | Vida | Motivo (comentario del código) | Línea |
|---|---|---|---|
| `PhoneDialerDevice` | Transient | servicio de dispositivo sin estado | `Ejemplos_Devices/Phone/Ejemplo_Maui_DirectCall/MauiProgram.cs:34` |
| `CallCoordinator` | Singleton | estado del overlay compartido entre callers | `.../MauiProgram.cs:36-38` |
| `MainPageViewModel` | Transient | no arrastrar estado entre navegaciones | `.../MauiProgram.cs:40-42` |
| `MainPage` | Transient | constructor recibe el VM | `.../MauiProgram.cs:43` |

`MainPage` sólo tiene el constructor `MainPage(MainPageViewModel vm)` (`.../Pages/MainPage.xaml.cs:14-19`) y el Shell la instancia por `ContentTemplate="{DataTemplate pages:MainPage}"` (`.../AppShell.xaml:9`); que la resolución pase por el contenedor es comportamiento de MAUI, no código del repo (no verificado en el ejemplo).

`MauiProgram` registra además la fuente `MaterialIconsOutlined` (`.../MauiProgram.cs:21`), usada por el ícono `phone_disabled` del overlay.

### 5.2 `PhoneDialerDevice.CallPhoneAsync` por plataforma

| Rama | Comportamiento | Líneas |
|---|---|---|
| Común | Si el token ya está cancelado → `Cancelled` | `Services/PhoneDialerDevice.cs:22` |
| `#if ANDROID` | 1) `PedirPermisoAndroidAsync` → si devuelve resultado, corta. 2) Re-chequeo de cancelación. 3) `RealizarLlamadaAndroid`; `FeatureNotSupportedException` → `NotSupported`, otra → `Failure(ex.Message)` | `:24-37` |
| `#elif IOS \|\| MACCATALYST` | `!PhoneDialer.Default.IsSupported` → `NotSupported`; si no, `Open(numero)` → `Success` («Success» = el dialer se abrió). `ArgumentNullException` → `Failure("Número inválido.")` | `:39-55` |
| `#else` (Windows/otros) | `NotSupported` | `:57-61` |

Permiso en Android (`PedirPermisoAndroidAsync`, `:67-87`):

```
CheckStatusAsync<Permissions.Phone>
   Granted ───────────► null (seguir)
   Restricted ────────► PermissionRestricted
   (cancelado) ───────► Cancelled
RequestAsync<Permissions.Phone>
   Granted ───────────► null
   Restricted ────────► PermissionRestricted
   otro ─► ShouldShowRationale<Permissions.Phone>()
             true  ───► PermissionDenied          (el SO todavía puede mostrar el diálogo)
             false ───► PermissionDeniedPermanent (“no volver a preguntar” → ajustes)
```

Llamada (`RealizarLlamadaAndroid`, `:89-100`): toma `Platform.CurrentActivity` (si es null → `Failure("No hay actividad activa…")`), arma `new Intent(Intent.ActionCall)` con `tel:{numero}`, `ActivityFlags.NewTask`, `StartActivity` → `Success(numero)`. El número **no se normaliza ni valida** más allá del chequeo de vacío del coordinador.

### 5.3 `CallResult` (record tipado)

`Services/CallResult.cs:6-28` — `abstract record CallResult` con casos anidados `sealed`:

| Caso | Significado (comentario) | Reacción del overlay (`CallCoordinator.Aplicar`, `Services/CallCoordinator.cs:94-120`) |
|---|---|---|
| `Success(string Numero)` | llamada iniciada / dialer abierto | `Hide()` |
| `PermissionDenied` | denegado, se puede volver a pedir | `ShowPermissionDenied(canRetry: true)` |
| `PermissionDeniedPermanent` | denegado persistente | `ShowPermissionDenied(canRetry: false)` |
| `PermissionRestricted` | MDM / control parental | `ShowRestricted()` |
| `NotSupported` | sin telefonía (tablet, simulador) | `ShowNotSupported()` |
| `Cancelled` | token cancelado | `Hide()` |
| `Failure(string Message)` | error inesperado | `ShowFailure(Message)` |

### 5.4 `CallCoordinator.CallAsync`

`Services/CallCoordinator.cs:31-73`:

1. Número vacío → `Failure("Ingresá un número de teléfono.")`, aplicado al overlay y devuelto (`:33-38`). No pasa por el dispositivo.
2. Guarda `_ultimoNumero` (`:40`) — lo usa el reintento desde el overlay.
3. Cancela/descarta el CTS previo y crea uno nuevo (enlazado si vino un `ct`) (`:42-46`).
4. `Overlay.ShowBusy()` en hilo principal (`:48`).
5. `await _dialer.CallPhoneAsync(...)` → `Aplicar(result)` en hilo principal (`:50-54`); `OperationCanceledException` → `Cancelled`; otra → `Failure` (`:56-67`); `finally` libera el CTS (`:68-72`).

Auxiliares: `Cancelar()` (`:75`), `ReintentarAsync()` reusa `_ultimoNumero` o esconde el overlay (`:77-86`), `AbrirAjustes()` → `AppInfo.Current.ShowSettingsUI()` con `catch` vacío (`:88-92`).

### 5.5 Overlay: `CallOverlayViewModel` + `CallPermissionOverlayView`

Estado (`ViewModels/CallOverlayViewModel.cs`): `IsBusy`, `IsDenied` (ambos notifican `IsVisible`, `:23-43`), `Title`, `Message`, `CanRetry`, `CanOpenSettings`, `CanRetryAction`; `IsVisible => _isBusy || _isDenied` (`:88`). Comandos (`:12-21`): `PedirPermisoCommand` y `ReintentarCommand` comparten el callback `onRetry` (textos distintos según la situación); `AbrirAjustesCommand`; `CerrarOverlayCommand` → `Hide`.

| Método | Título | `CanRetry` («Pedir permiso») | `CanRetryAction` («Reintentar») | `CanOpenSettings` | Líneas |
|---|---|---|---|---|---|
| `ShowBusy()` | «Iniciando llamada...» | — | — | — | `:95-101` |
| `ShowPermissionDenied(true)` | «Permiso de llamadas necesario» | ✔ | ✘ | ✔ | `:103-118` |
| `ShowPermissionDenied(false)` | «Acceso a llamadas denegado» | ✘ | ✔ | ✔ | `:103-118` |
| `ShowRestricted()` | «Acceso restringido» | ✘ | ✔ | ✔ | `:120-129` |
| `ShowNotSupported()` | «Función no disponible» | ✘ | ✘ | ✘ | `:131-140` |
| `ShowFailure(msg)` | «Error al llamar» | ✘ | ✔ | ✘ | `:142-151` |
| `Hide()` | — | — | — | — | `:153-157` |

Regla visible en la tabla: «Pedir permiso» y «Reintentar» nunca coexisten; «Abrir configuración» sólo aparece cuando el problema es de permisos.

Vista (`Controls/CallPermissionOverlayView.xaml`):

| Elemento | Detalle | Líneas |
|---|---|---|
| Raíz `ContentView` | `IsVisible="{Binding IsVisible}"`, bloquea input (`InputTransparent=False`, `CascadeInputTransparent=False`) | `:5-10` |
| Comentario de diseño | backdrop en el `Grid` raíz; alpha `0xE6` (~90 %) | `:12-26` |
| `Grid` backdrop `#E6000000` | cubre toda el área | `:27` |
| Panel de espera (`IsBusy`) | `ActivityIndicator` + `Title` + «Aguardá unos segundos» | `:30-34` |
| Panel denegado (`IsDenied`) | ícono `phone_disabled`, `Title`, `Message`, 4 botones (Pedir permiso / Reintentar / Abrir configuración / Cerrar) | `:37-75` |

En la página: `Grid` raíz con el `ScrollView` y luego `<controls:CallPermissionOverlayView BindingContext="{Binding CallOverlay}" />` (`Pages/MainPage.xaml:8-26`, overlay en `:24`). El botón «LLamar» usa `Command="{Binding RealizarLLamadaCommand}"` (`:16-18`).

### 5.6 Infraestructura MVVM sin dependencias

| Clase | Rol | Líneas |
|---|---|---|
| `ViewModelBase` | `INotifyPropertyChanged` + `SetProperty<T>` con `EqualityComparer` | `ViewModels/ViewModelBase.cs:8-24` |
| `AsyncRelayCommand` | `async void Execute`, evita reentrada con `IsRunning`, `CanExecute = !_isRunning && canExecute` | `ViewModels/AsyncRelayCommand.cs:10-58` |
| `RelayCommand` | variante sincrónica (abrir ajustes, cerrar) | `ViewModels/AsyncRelayCommand.cs:61-77` |
| `MainPageViewModel` | `Telefono` + `RealizarLLamadaCommand` que sólo delega en el coordinador | `ViewModels/MainPageViewModel.cs:6-43` |

`AsyncRelayCommand.cs` es idéntico al de `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/AsyncRelayCommand.cs` salvo el namespace y una línea en blanco (duplicación deliberada: un proyecto por técnica). Su explicación larga vive en `Ejemplos_Devices/Docs/otros/AsyncRelayCommandOptions.md` → ver [10_Documentacion-Transversal](10_Documentacion-Transversal.md).

---

## 6. Manifiestos y declaraciones por plataforma

### 6.1 Android

| Declaración | Dialer (`Platforms/Android/AndroidManifest.xml`) | DirectCall (`Platforms/Android/AndroidManifest.xml`) |
|---|---|---|
| `INTERNET`, `ACCESS_NETWORK_STATE` | `:10-11` (plantilla; el ejemplo no usa red) | `:10-11` (ídem) |
| `CALL_PHONE` | — | `:14` (comentario `:13`: «Para llamada directa (Intent.ActionCall) → SÍ necesita permiso») |
| `READ_PHONE_STATE` | — | `:15` |
| `<queries>` | `action.DIAL` + `scheme=tel` (`:14-19`), comentario: «Necesario para que PhoneDialer.IsSupported detecte la app de teléfono» (`:13`) | `action.VIEW` + `scheme=tel` (`:17-22`) |

- Qué permisos concretos agrupa `Permissions.Phone` de MAUI en Android (si incluye `READ_PHONE_STATE` además de `CALL_PHONE`) depende de la librería, no del repo: **no verificado**.
- La rama Android de DirectCall no consulta `PhoneDialer` ni resuelve el intent antes de `StartActivity` (`Services/PhoneDialerDevice.cs:89-100`); el bloque `<queries>` VIEW no es invocado explícitamente por el código propio.

### 6.2 iOS

| Clave | Dialer (`Platforms/iOS/Info.plist`) | DirectCall (`Platforms/iOS/Info.plist`) |
|---|---|---|
| Identidad | `CFBundleIdentifier` `com.ejemplos.phone.dialer` (`:43-44`), display «Maui Dialer» (`:40-41`) | `com.ejemplos.phone.directcall` (`:43-44`), «Llamada directa» (`:40-41`) |
| `MinimumOSVersion` | 15.0 (`:34-35`) | 15.0 (`:34-35`) |
| `NSCameraUsageDescription` | presente (`:59-60`), resto de plantilla de cámara | presente (`:59-60`), ídem |
| `LSApplicationQueriesSchemes` | `whatsapp`, `tg` — **sin `tel`** (`:62-67`) | **ausente** |
| Permiso de telefonía | no existe en iOS | no existe en iOS |

`Entitlements.plist` y `Entitlements.Development.plist` de ambos proyectos declaran `aps-environment=development` y `get-task-allow=true` (`Platforms/iOS/Entitlements*.plist:5-9`): son de plantilla (push), no de telefonía. El workflow firma con `Entitlements.Development.plist` → ver [09_CI-CD-y-Build](09_CI-CD-y-Build.md).

`Properties/launchSettings.json` de ambos sólo trae el perfil «Windows Machine», aunque ningún `.csproj` apunta a Windows (resto de plantilla).

---

## 7. Comportamiento esperado por plataforma (según el código)

| Escenario | Dialer | DirectCall |
|---|---|---|
| Android con telefonía | Se abre el marcador con el número | Diálogo de permiso la 1.ª vez; si se concede, **llama sin confirmación** |
| Android, permiso denegado | n/a | Overlay «Permiso de llamadas necesario» con «Pedir permiso» + «Abrir configuración» |
| Android, «no volver a preguntar» | n/a | Overlay «Acceso a llamadas denegado» con «Reintentar» + «Abrir configuración» |
| iPhone | Marcador del SO | Marcador del SO (mismo comportamiento que Dialer; el overlay se oculta con `Success`) |
| Tablet sin SIM / simulador | Alerta «no puede hacer llamadas» si `IsSupported` es false | Overlay «Función no disponible» (iOS vía `IsSupported`; en Android la ruta de error depende de lo que lance `StartActivity`: **no verificado**) |
| Windows / otro | (no hay TFM) | `#else` → `NotSupported` (rama inalcanzable con los TFM actuales) |

---

## 8. Relación con otros dominios

| Pieza | Relación | Índice |
|---|---|---|
| `GpsOverlayViewModel` (`Ejemplos_Devices/GPS/Ejemplo_Maui_GPS/ViewModels/`) | `CallOverlayViewModel` se declara «Espejo del GpsOverlayViewModel» (`ViewModels/CallOverlayViewModel.cs:5-6`); ambos son overlays de 1.ª generación, autocontenidos | [04_GPS](04_GPS.md) |
| `LibApp/Devices/Phone/**` y `LibApp/UrlCommands/Handlers/CallCommandHandler.cs` en `Ejemplo_Maui_Hibrida` | Evolución (2.ª gen.): `ICallService`/`CallService`, modelos `CallResult`, `CallFailure`, `CallMode`; `CallOverlayViewModel` hereda de `StatusOverlayViewModel` y usa `CommunityToolkit.Mvvm` | [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) |
| `Ejemplos_Devices/Docs/web-hibrida/llamada.md` | Diseño de la llamada dentro de la app híbrida | [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) |
| `.github/workflows/cd-ios-phone.Ejemplo_Dialer.yml` / `cd-ios-phone.Ejemplo_DirectCall.yml` | Publicación iOS; sólo `workflow_call` (el disparo por `push` está comentado, `:3-15`); no se encontró otro workflow que los invoque. Difieren en nombre, paquete, proyecto y `XCODE_FILE_INSTALLER` | [09_CI-CD-y-Build](09_CI-CD-y-Build.md) |
| Repo de documentación: `Ejemplos_Maui_Devices-docs/docs/pieces/phone/README.md`, ADR `0001-un-proyecto-por-tecnica.md`, `0002-servicio-tipado-overlay-mvvm.md`, `Analisis/Plan-Armonizacion-Overlays.md` | Prosa sobre este dominio (existen; verificado) | [10_Documentacion-Transversal](10_Documentacion-Transversal.md) |

No hay workflow de Android para `Phone` ni pruebas automatizadas de estos dos proyectos (las pruebas de overlay de llamada, `CallOverlayTests.cs`, son de la app híbrida).

---

## 9. ⚠️ Trampas de lectura y discrepancias

| # | Trampa | Evidencia |
|---|---|---|
| 1 | ⚠️ El comentario de `PhoneDialerDevice` dice que iOS «requiere "tel" en LSApplicationQueriesSchemes… (ya declarado en este proyecto)», pero el `Info.plist` de DirectCall **no tiene** esa clave. Si `PhoneDialer.IsSupported` en iOS depende de ella es comportamiento de MAUI: no verificado | `Services/PhoneDialerDevice.cs:14-16`, `:40-42` vs `Ejemplo_Maui_DirectCall/Platforms/iOS/Info.plist:1-62` |
| 2 | ⚠️ El comentario de `CallResult.PermissionDenied` menciona «iOS: estado .NotDetermined», pero la rama iOS nunca pide permisos ni devuelve `PermissionDenied*`/`Restricted` | `Services/CallResult.cs:10-12` vs `Services/PhoneDialerDevice.cs:39-55` |
| 3 | ⚠️ En iOS, DirectCall **no** llama directo: abre el marcador igual que Dialer y devuelve `Success` | `Services/PhoneDialerDevice.cs:48-51` |
| 4 | El `<queries>` difiere: Dialer declara `DIAL`, DirectCall declara `VIEW`. No son intercambiables a ciegas al copiar manifiestos | `Ejemplo_Maui_Dialer/.../AndroidManifest.xml:14-19` vs `Ejemplo_Maui_DirectCall/.../AndroidManifest.xml:17-22` |
| 5 | `Cancelar()` es público pero nadie lo invoca; `AsyncRelayCommand.Execute` siempre pasa `CancellationToken.None`. La cancelación queda como costura didáctica | `Services/CallCoordinator.cs:75`, `ViewModels/AsyncRelayCommand.cs:48` |
| 6 | ⚠️ `Docs/otros/AsyncRelayCommandOptions.md:111` cita `MainPageViewModel.cs:47-58` con un `try/catch`, pero el archivo tiene 43 líneas y el VM no tiene `try/catch` (lo maneja el coordinador). El mismo doc muestra `PhoneDialer.Default.OpenAsync` (`:187`), API que el código del dominio no usa (usa `Open`). Además el link es relativo a `Docs/otros/` (`../Phone/...`) y resuelve a `Ejemplos_Devices/Docs/Phone/`, que no existe | `ViewModels/MainPageViewModel.cs:36-42` |
| 7 | Restos de plantilla: `NSCameraUsageDescription` en ambos `Info.plist`, `whatsapp`/`tg` en Dialer, `aps-environment` en entitlements, `INTERNET` en manifests, perfil «Windows Machine». No intervienen en telefonía | §6 |
| 8 | Nombres: el botón de Dialer es `btnAbrirMarcado` pero su handler `btnAbrirMarcador_Clicked`; en DirectCall `btnRealizarLLamada`/`RealizarLLamadaCommand` con doble «L» mayúscula. Respetalos tal cual al buscar | `Ejemplo_Maui_Dialer/Pages/MainPage.xaml:12`, `Ejemplo_Maui_DirectCall/Pages/MainPage.xaml:16-18` |
| 9 | Dialer valida vacío en la página; DirectCall lo valida en el coordinador (y lo muestra como `Failure` en el overlay, con botón «Reintentar» que reusará el último número no vacío, si existe) | `Ejemplo_Maui_Dialer/Pages/MainPage.xaml.cs:28-32`, `Services/CallCoordinator.cs:33-38`, `:77-86` |
| 10 | El `Entry` de ambos no fija `Keyboard="Telephone"`: se abre el teclado general | `Pages/MainPage.xaml:10` (Dialer) y `:14` (DirectCall) |

---

## 10. Preguntas frecuentes

| Pregunta | Respuesta corta | Ver |
|---|---|---|
| ¿Cuál de los dos necesita `CALL_PHONE`? | Sólo DirectCall, y sólo en Android | §6.1 |
| ¿Por qué Dialer no pide permisos? | El usuario confirma en el marcador del SO | §1, §4 |
| ¿DirectCall llama directo en iPhone? | No; abre el marcador | §5.2, §9 #3 |
| ¿Qué pasa si el usuario marca «no volver a preguntar»? | `PermissionDeniedPermanent` → overlay con «Reintentar» + «Abrir configuración» | §5.2, §5.5 |
| ¿Dónde se decide qué botones muestra el overlay? | En los `Show*` de `CallOverlayViewModel` | §5.5 |
| ¿Cómo llamo desde otro lugar de la app (no la página)? | Inyectá `CallCoordinator` y `await CallAsync(numero)`; el overlay se actualiza solo | §5.4 |
| ¿Por qué el coordinador es singleton? | El overlay y el reintento se comparten entre callers | §5.1 |
| ¿Hay pruebas unitarias del dominio? | No en `Phone/`; las de overlay de llamada son de la híbrida | §8 |
| ¿Dónde está la versión «de producción» de la llamada? | En `Ejemplo_Maui_Hibrida/LibApp/Devices/Phone` | [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) |
