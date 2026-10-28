# 07 — Red y conectividad

> **Propósito:** mapa del dominio «Red y conectividad»: el ejemplo `Ejemplo_Maui_Connectivity` (API `Microsoft.Maui.Networking.Connectivity`) y su nota `Ejemplo_Docs_Red/Readme.md`. Cubre cómo consultar el estado de red, cómo detectar cambios, qué permisos pide cada plataforma y qué está todavía «en construcción».
>
> **Fuente primaria:** `Ejemplos_Devices/Red/**` (commit `285f6fb`).
>
> **Rutas:** relativas a la raíz del repo de código; `Red/` abrevia `Ejemplos_Devices/Red/`.
>
> **Fuera de alcance:** el uso de `Connectivity` dentro de la app híbrida (`Integrada/Ejemplo_Maui_Hibrida/MauiProgram.cs:94` registra `Connectivity.Current` como singleton) → [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## §1 Mapa de proyectos del dominio

| Proyecto | Ruta | Técnica | Archivos clave | En .slnx | Workflow CI |
|---|---|---|---|---|---|
| `Ejemplo_Maui_Connectivity` | `Red/Ejemplo_Maui_Connectivity/` | `Connectivity.ConnectivityChanged` + `ConnectionProfiles` (MAUI Essentials, sin NuGet extra) | `Utilities/ConnectivityTest.cs`, `Pages/MainPage.xaml(.cs)`, `Platforms/Android/AndroidManifest.xml`, `.csproj` | sí (`Ejemplos_Devices/Ejemplos_Devices.slnx:187-191`, carpeta `/Ejemplos/Red/`) | **no** (no hay `cd-*` de Red en `.github/workflows/`) |
| `Ejemplo_Docs_Red` | `Red/Ejemplo_Docs_Red/Readme.md` | Nota con el enlace a la doc oficial, valores de `NetworkAccess` y la clase de ejemplo | `Readme.md` | sí, como archivo de solución (`.slnx:192-194`) | — |

Estado del ejemplo: **en construcción**. La clase de detección existe, pero la página no la usa y no muestra nada (§6.1, §6.2).

### §1.1 Árbol del proyecto (solo lo versionado y con valor)

```
Red/Ejemplo_Maui_Connectivity/
├── Ejemplo_Maui_Connectivity.csproj   TFM android (+ios en macOS); RootNamespace Ejemplo_Maui_Conexion
├── MauiProgram.cs                     builder estándar (fuentes + logging Debug); sin registros de red
├── App.xaml(.cs) · AppShell.xaml(.cs) Shell con una ShellContent «Home» → MainPage
├── Pages/MainPage.xaml(.cs)           ContentPage con ScrollView vacío
├── Utilities/ConnectivityTest.cs      suscripción a ConnectivityChanged y log por consola
├── Platforms/Android/                 AndroidManifest (ACCESS_NETWORK_STATE, INTERNET), MainActivity, MainApplication
├── Platforms/iOS/                     Info.plist de plantilla, AppDelegate, Program (Resources/ fuera de indexado)
├── Platforms/Windows/                 App.xaml(.cs), Package.appxmanifest, app.manifest (⚠️ §6.4)
└── Properties/launchSettings.json     perfil «Windows Machine»
```

---

## §2 Configuración del proyecto (`Ejemplo_Maui_Connectivity.csproj`)

| Propiedad | Valor | Línea |
|---|---|---|
| `TargetFrameworks` | `net10.0-android`; `net10.0-ios` solo en macOS | `:4-5` |
| `RootNamespace` | `Ejemplo_Maui_Conexion` (≠ nombre del proyecto) | `:8` |
| `ApplicationTitle` / `ApplicationId` | `Maui Conexion` / `com.ejemplos.red.connectivity` | `:14`, `:16` |
| `SupportedOSPlatformVersion` | iOS 15.0 · Android 25.0 · Windows 10.0.17763.0 | `:23-25` |
| `TargetPlatformMinVersion` (Windows) | 10.0.17763.0 | `:26` |
| Android | `max-page-size=16384`; RIDs `android-arm;android-arm64;android-x86;android-x64`; APK sin split por ABI | `:31-48` |
| iOS Release | `UseInterpreter`, `PublishTrimmed`, `MtouchLink=SdkOnly`, AOT solo `ios-arm64` | `:52-62` |
| Paquetes | `Microsoft.Maui.Controls` `$(MauiVersion)` · `Microsoft.Extensions.Logging.Debug` 10.0.2 | `:80-81` |

> `Connectivity` viene en el propio MAUI (Essentials): no hace falta un NuGet aparte (no hay otro `PackageReference` en el `.csproj`).

---

## §3 API `Connectivity` usada

### §3.1 Consulta puntual del estado

Está documentada en el Readme (`Red/Ejemplo_Docs_Red/Readme.md:7-12`). El código del proyecto no la usa:

```csharp
NetworkAccess accessType = Connectivity.Current.NetworkAccess;
if (accessType == NetworkAccess.Internet) { /* hay internet */ }
```

### §3.2 Estados `NetworkAccess` (según el Readme, `:15-19`)

| Valor del enum | Significado (Readme) | Rama que lo trata en `ConnectivityTest` |
|---|---|---|
| `Internet` | Acceso local e Internet | — (no imprime aviso) |
| `ConstrainedInternet` | Acceso limitado; portal cautivo que da acceso a internet tras autenticarse | «Internet access is available but is limited.» (`ConnectivityTest.cs:17-18`) |
| `Local` | Solo red local | «Internet access has been lost.» (`:20-21`, rama `!= Internet`) |
| `None` (el Readme dice «Ninguno») | Sin conectividad | ídem |
| `Unknown` (el Readme dice «Desconocido») | No se puede determinar | ídem |

> El Readme traduce los nombres de `None` y `Unknown`; en código hay que escribirlos en inglés: `NetworkAccess.None` y `NetworkAccess.Unknown`.

### §3.3 Perfiles de conexión (`ConnectionProfile`)

`ConnectivityTest.cs:26-45` recorre `e.ConnectionProfiles` y escribe:

| `ConnectionProfile` | Texto impreso |
|---|---|
| `Bluetooth` | `Bluetooth` |
| `Cellular` | `Cell` |
| `Ethernet` | `Ethernet` |
| `WiFi` | `WiFi` |
| otro (`default`) | nada |

---

## §4 Detección de cambios (`Utilities/ConnectivityTest.cs`)

```
new ConnectivityTest()                              ctor  :9-10
 └─ Connectivity.ConnectivityChanged += Connectivity_ConnectivityChanged   (evento estático)
Connectivity_ConnectivityChanged(sender, ConnectivityChangedEventArgs e)   :15-48
 ├─ e.NetworkAccess == ConstrainedInternet → Console.WriteLine("…limited.")
 ├─ else if e.NetworkAccess != Internet    → Console.WriteLine("…lost.")   (Local, None, Unknown)
 ├─ Console.Write("Connections active: ")                                 :24 (siempre)
 └─ foreach e.ConnectionProfiles           → Console.Write(perfil, sin separador)  +  Console.WriteLine()  :26-47
~ConnectivityTest()                                 finalizador  :12-13
 └─ Connectivity.ConnectivityChanged -= …
```

- El cuerpo de la clase es **idéntico** al bloque de código del Readme (`Red/Ejemplo_Docs_Red/Readme.md:22-64`); el `.cs` solo agrega los `using` y el `namespace Ejemplo_Maui_Conexion.Utilities` (`ConnectivityTest.cs:1-5`). Que reproduzca el ejemplo de la doc de Microsoft enlazada en `:3` es lo esperable, pero (no verificado) contra esa página.
- La salida va a `Console`: en el dispositivo se ve en la salida de depuración o en el log de la plataforma, no en la UI.
- Uso desde una página o servicio con inyección de dependencias: `Connectivity.Current` es la instancia `IConnectivity`. En la híbrida se registra como singleton (`Integrada/Ejemplo_Maui_Hibrida/MauiProgram.cs:94`) → [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

---

## §5 Permisos y configuración por plataforma

| Plataforma | Archivo | Qué declara | ¿Lo necesita `Connectivity`? |
|---|---|---|---|
| Android | `Red/Ejemplo_Maui_Connectivity/Platforms/Android/AndroidManifest.xml:10` | `android.permission.ACCESS_NETWORK_STATE` | **Sí**: es el permiso para leer el estado de red (según la doc oficial enlazada en el Readme `:3`; el repo no lo explica) |
| Android | ídem `:11` | `android.permission.INTERNET` | No para leer el estado; sí para usar la red |
| Android | ídem `:13` | `uses-sdk` min 25 · target 36 | — |
| Android | ídem `:2` | `<manifest>` sin atributo `package` (lo aporta `ApplicationId`) | — |
| iOS | `Platforms/iOS/Info.plist:4-31` | Plantilla: `LSRequiresIPhoneOS`, `UIDeviceFamily`, `arm64`, orientaciones, `XSAppIconAssets`; **sin** claves `NS*UsageDescription` | No requiere claves de privacidad (ninguna declarada) |
| Windows | `Platforms/Windows/Package.appxmanifest:42-44` | `runFullTrust` | Inerte: Windows no está en los TFM (⚠️ §6.4) |

> Solo `ACCESS_NETWORK_STATE` habilita la lectura del estado en Android. Que iOS no necesite ninguna clave sale de que el `Info.plist` no declara ninguna; el repo no lo documenta explícitamente.

---

## §6 Trampas de lectura y discrepancias ⚠️

### §6.1 ⚠️ La detección nunca se activa

- `ConnectivityTest` no se instancia en ningún lado: `git grep ConnectivityTest` solo encuentra su propia definición (`Utilities/ConnectivityTest.cs:7-13`).
- `MauiProgram.cs` no la registra (`:9-16`) y `MainPage.xaml.cs:6-9` solo llama a `InitializeComponent()`. Al correr la app no se suscribe nada y no se imprime nada.

### §6.2 ⚠️ La página está vacía

`Pages/MainPage.xaml:6-8` contiene un `<ScrollView>` sin hijos. La app arranca con una pantalla en blanco (el título de Shell es «Conexion», `AppShell.xaml:8`). No hay controles que muestren `NetworkAccess` ni perfiles.

### §6.3 ⚠️ El finalizador no desuscribe en la práctica

- La desuscripción está en el finalizador `~ConnectivityTest()` (`ConnectivityTest.cs:12-13`), copiado del Readme (`:27-28`).
- `Connectivity.ConnectivityChanged` es un evento **estático**: la suscripción mantiene viva la instancia, el GC no la recolecta y el finalizador no corre mientras siga suscripta.
- Para un ejemplo real conviene suscribir en `OnAppearing` y desuscribir en `OnDisappearing`, o implementar `IDisposable`. Es una lectura técnica del patrón; el comportamiento en ejecución no se verificó en este repo.

### §6.4 ⚠️ Carpeta `Platforms/Windows` sin TFM Windows

- El proyecto versiona `Platforms/Windows/` (`App.xaml(.cs)`, `Package.appxmanifest`, `app.manifest`) y propiedades de Windows en el `.csproj` (`WindowsPackageType` `:21`; condiciones `windows` `:25-26`).
- Sin embargo, `TargetFrameworks` solo incluye `net10.0-android` y `net10.0-ios` (`:4-5`): esos archivos no se compilan.
- El perfil «Windows Machine» de `Properties/launchSettings.json` tampoco tiene un TFM que lo respalde.
- Es herencia de la plantilla MAUI; los proyectos QR, por ejemplo, no versionan esa carpeta.

### §6.5 Nombres que no coinciden

| Dónde | Valor |
|---|---|
| Carpeta / `.csproj` | `Ejemplo_Maui_Connectivity` |
| `RootNamespace` y todos los `namespace` | `Ejemplo_Maui_Conexion` (`.csproj:8`, `App.xaml.cs:3`, `Platforms/Windows/App.xaml.cs:6`) |
| `ApplicationId` | `com.ejemplos.red.connectivity` (no sigue el patrón `com.ejemplos.devices.<dominio>…` de la mayoría, p. ej. QR o GPS; Phone tampoco: `com.ejemplos.phone.*`) |

Al buscar tipos, hay que usar `Ejemplo_Maui_Conexion.*`, no `Ejemplo_Maui_Connectivity.*`.

### §6.6 Codificación del Readme

`Red/Ejemplo_Docs_Red/Readme.md` no está en UTF-8: `file` lo detecta como «Non-ISO extended-ASCII text». Los guiones largos se ven como `�` (`:16`, `:18`). Al citarlo o procesarlo hay que convertirlo (p. ej. desde Windows-1252) o asumir que esos caracteres se pierden.

### §6.7 Sin CI ni Readme de proyecto

- No hay `Readme.md` dentro de `Ejemplo_Maui_Connectivity/`: la única documentación es `Ejemplo_Docs_Red/Readme.md`.
- Ningún workflow de `.github/workflows/` compila este proyecto. Se compila localmente con `dotnet build` para `net10.0-android` (no verificado: no se corrió el build).

---

## §7 Preguntas frecuentes

| Pregunta | Respuesta corta | § |
|---|---|---|
| ¿Cómo sé si hay internet ahora? | `Connectivity.Current.NetworkAccess == NetworkAccess.Internet` | §3.1 |
| ¿Qué significa `ConstrainedInternet`? | Hay un portal cautivo: acceso limitado hasta autenticarse | §3.2 |
| ¿Cómo me entero de que se cayó la red? | Suscribirse a `Connectivity.ConnectivityChanged` y evaluar `e.NetworkAccess` | §4 |
| ¿Cómo sé si estoy por WiFi o por datos móviles? | `e.ConnectionProfiles` (`WiFi`, `Cellular`, `Ethernet`, `Bluetooth`) | §3.3 |
| ¿Qué permiso pide Android? | `ACCESS_NETWORK_STATE` | §5 |
| ¿Necesito algo en `Info.plist`? | El ejemplo no declara nada y no lo requiere | §5 |
| ¿Hace falta un NuGet? | No; viene con MAUI | §2 |
| ¿Por qué la app no muestra nada? | La página está vacía y `ConnectivityTest` no se instancia | §6.1, §6.2 |
| ¿Por qué no encuentro el namespace `Ejemplo_Maui_Connectivity`? | Es `Ejemplo_Maui_Conexion` | §6.5 |
| ¿Hay pipeline que lo compile? | No | §6.7 |
| ¿Cómo se usa la conectividad en la app híbrida? | → [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) | — |

---

## §8 Tabla de fuentes

| Fuente | Qué aporta |
|---|---|
| `Ejemplos_Devices/Red/Ejemplo_Maui_Connectivity/Ejemplo_Maui_Connectivity.csproj` | TFM, `RootNamespace`, `ApplicationId`, paquetes |
| `Red/Ejemplo_Maui_Connectivity/Utilities/ConnectivityTest.cs` | Detección de cambios y perfiles |
| `Red/Ejemplo_Maui_Connectivity/Pages/MainPage.xaml(.cs)` | Página vacía |
| `Red/Ejemplo_Maui_Connectivity/MauiProgram.cs`, `App.xaml(.cs)`, `AppShell.xaml(.cs)` | Arranque estándar sin registros de red |
| `Red/Ejemplo_Maui_Connectivity/Platforms/Android/AndroidManifest.xml` | `ACCESS_NETWORK_STATE`, `INTERNET` |
| `Red/Ejemplo_Maui_Connectivity/Platforms/iOS/Info.plist` | iOS sin claves de privacidad de uso |
| `Red/Ejemplo_Maui_Connectivity/Platforms/Windows/*`, `Properties/launchSettings.json` | Restos de plantilla Windows |
| `Red/Ejemplo_Docs_Red/Readme.md` | Enlace a la doc oficial, estados `NetworkAccess`, clase de ejemplo |
| `Ejemplos_Devices/Ejemplos_Devices.slnx:187-194` | Pertenencia a la solución |
| `.github/workflows/` | Ausencia de workflow para Red |
| `Integrada/Ejemplo_Maui_Hibrida/MauiProgram.cs:94` | Referencia cruzada: `Connectivity.Current` en DI |
