# 07 — Red y Conectividad

> **Propósito**: destilar el dominio de red de la colección: la API `Connectivity` de .NET MAUI (`NetworkAccess`, `ConnectionProfiles`, evento `ConnectivityChanged`), el permiso Android `ACCESS_NETWORK_STATE`, el proyecto `Ejemplo_Maui_Connectivity` (esqueleto «en construcción») y la nota `Ejemplo_Docs_Red`.
> **Fuente primaria**: `Ejemplos_Devices/Red/**` (commit `285f6fb`): `Ejemplo_Maui_Connectivity/` (proyecto MAUI) y `Ejemplo_Docs_Red/Readme.md` (nota de cátedra).
> **Índices relacionados**: [00_MASTER-INDEX](00_MASTER-INDEX.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (uso *real* de conectividad: `NetworkService`, sonda activa, overlay de red) · [09_CI-CD-y-Build](09_CI-CD-y-Build.md) (solución `.slnx`, pipelines iOS) · [10_Documentacion-Transversal](10_Documentacion-Transversal.md) (mapa del `README.md` raíz y de las carpetas `Ejemplo_Docs_*`).

---

## 1. Resumen en una pantalla

| Dato | Valor | Fuente |
|---|---|---|
| Proyectos del dominio | 1 app MAUI (`Ejemplo_Maui_Connectivity`) + 1 carpeta de docs (`Ejemplo_Docs_Red`) | `git ls-files Ejemplos_Devices/Red` |
| Estado | Esqueleto didáctico «en construcción»: la página está vacía y la clase de ejemplo no se usa | `README.md:17-18`, `Pages/MainPage.xaml:6-8` |
| API cubierta | `Microsoft.Maui.Networking.Connectivity` (estática), `NetworkAccess`, `ConnectionProfile`, `ConnectivityChangedEventArgs` | `Utilities/ConnectivityTest.cs:9-48` |
| Identidad | `ApplicationId` `com.ejemplos.red.connectivity`, `ApplicationTitle` «Maui Conexion», versión `1.0` (1) | `.csproj:14-19` |
| Target | `net10.0-android` (+ `net10.0-ios` sólo si se compila en macOS) | `Ejemplo_Maui_Connectivity.csproj:4-5` |
| Permisos Android | `ACCESS_NETWORK_STATE`, `INTERNET` | `Platforms/Android/AndroidManifest.xml:10-11` |
| Permisos iOS | ninguno declarado (no hace falta clave en `Info.plist` para `Connectivity`) | `Platforms/iOS/Info.plist:1-32` |
| Paquetes | `Microsoft.Maui.Controls` (`$(MauiVersion)`), `Microsoft.Extensions.Logging.Debug` 10.0.2 | `.csproj:80-81` |
| Pipeline CI | ninguno (no hay `cd-ios-red.*.yml` en `.github/workflows/`) | `ls .github/workflows` |
| Documentación oficial citada | learn.microsoft.com → *platform-integration/communication/networking* (net-maui-10.0) | `Ejemplo_Docs_Red/Readme.md:3` |

Rutas de este índice relativas a la raíz del repo de código; se abrevia `Ejemplos_Devices/Red/Ejemplo_Maui_Connectivity/` como **`EMC/`**.

---

## 2. Inventario de fuentes

```
Ejemplos_Devices/Red/
├── Ejemplo_Docs_Red/
│   └── Readme.md                  ← nota: URL oficial, enum NetworkAccess, copia de ConnectivityTest (Windows-1252)
└── Ejemplo_Maui_Connectivity/     ← EMC/
    ├── Ejemplo_Maui_Connectivity.csproj   ← RootNamespace = Ejemplo_Maui_Conexion (⚠️ §7)
    ├── MauiProgram.cs             ← plantilla: fuentes + logging Debug; sin DI de IConnectivity
    ├── App.xaml / App.xaml.cs     ← CreateWindow → new Window(new AppShell())
    ├── AppShell.xaml(.cs)         ← un ShellContent → pages:MainPage, Title="Conexion"
    ├── Pages/MainPage.xaml(.cs)   ← ContentPage con <ScrollView> vacío
    ├── Utilities/ConnectivityTest.cs  ← ÚNICO código del dominio (suscripción al evento)
    ├── Platforms/Android/         ← AndroidManifest (permisos), MainActivity, MainApplication
    ├── Platforms/iOS/             ← Info.plist, AppDelegate, Program, PrivacyInfo.xcprivacy
    ├── Platforms/Windows/         ← presente pero NO compilado (⚠️ §7)
    ├── Properties/launchSettings.json  ← sólo perfil "Windows Machine"
    └── Resources/                 ← plantilla MAUI (íconos, fuentes, Styles, Raw/AboutAssets.txt)
```

Registro en la solución: carpeta `/Ejemplos/Red/` con el `.csproj` (`<Deploy />`) y `/Ejemplos/Red/Docs/` con el `Readme.md` (`Ejemplos_Devices/Ejemplos_Devices.slnx:187-194`). Detalle de la solución en [09_CI-CD-y-Build](09_CI-CD-y-Build.md).

Historial del dominio (`git log -- Ejemplos_Devices/Red`): `3a85bda` (alta; mensaje «.»), `f7c1099` («probando comipilar todo»), `7a8bf15` («prueba io integrado gps+status red»).

---

## 3. La API `Connectivity` tal como la usa el repo

### 3.1 Dos formas de consultar

| Forma | Dónde aparece | Uso |
|---|---|---|
| Consulta puntual: `Connectivity.Current.NetworkAccess` | `Ejemplo_Docs_Red/Readme.md:7-12` (sólo en la nota; no hay código que la ejecute en `EMC/`) | Leer el estado en un momento dado |
| Evento: `Connectivity.ConnectivityChanged += …` | `EMC/Utilities/ConnectivityTest.cs:9-10` | Reaccionar a cambios (Wi-Fi ↔ datos, pérdida de enlace) |
| Inyección `IConnectivity` | **no** en este dominio; sí en la app híbrida (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/MauiProgram.cs:94`) | Ver [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) |

### 3.2 Enum `NetworkAccess` (según la nota)

`Ejemplo_Docs_Red/Readme.md:15-19`:

| Valor real del enum | Rótulo en la nota | Significado |
|---|---|---|
| `Internet` | «Internet» | acceso local e Internet |
| `ConstrainedInternet` | «ConstrainedInternet» | Internet limitada: portal cautivo; tras autenticarse en el portal se concede acceso |
| `Local` | «Local» | sólo red local |
| `None` | ⚠️ «Ninguno» | sin conectividad |
| `Unknown` | ⚠️ «Desconocido» | no se puede determinar |

⚠️ La nota traduce `None`/`Unknown` al castellano; en el código hay que escribir `NetworkAccess.None` / `NetworkAccess.Unknown`.

### 3.3 Enum `ConnectionProfile` (lo que el ejemplo distingue)

`EMC/Utilities/ConnectivityTest.cs:26-45` recorre `e.ConnectionProfiles` y reconoce `Bluetooth` → «Bluetooth», `Cellular` → «Cell», `Ethernet` → «Ethernet», `WiFi` → «WiFi»; cualquier otro valor cae en `default: break` (no se imprime).

---

## 4. `ConnectivityTest` — anatomía

`EMC/Utilities/ConnectivityTest.cs` (namespace `Ejemplo_Maui_Conexion.Utilities`, línea 5):

| Líneas | Pieza | Qué hace |
|---|---|---|
| 9-10 | Constructor (expression-bodied) | Suscribe `Connectivity_ConnectivityChanged` al evento **estático** `Connectivity.ConnectivityChanged` |
| 12-13 | Finalizador `~ConnectivityTest()` | Desuscribe el handler |
| 15 | Firma del handler | `(object sender, ConnectivityChangedEventArgs e)` |
| 17-18 | Rama 1 | `ConstrainedInternet` → `Console.WriteLine("Internet access is available but is limited.")` |
| 20-21 | Rama 2 | cualquier valor ≠ `Internet` (incluye `Local`, `None`, `Unknown`) → «Internet access has been lost.» |
| 24 | Encabezado | `Console.Write("Connections active: ")` |
| 26-45 | `foreach` + `switch` | escribe el nombre de cada perfil activo |
| 47 | Cierre | `Console.WriteLine()` |

Flujo del evento:

```
SO (Android ConnectivityManager / iOS) ──cambio de red──▶ Connectivity.ConnectivityChanged (static)
        │
        ▼
ConnectivityTest.Connectivity_ConnectivityChanged(sender, e)
        ├─ e.NetworkAccess == ConstrainedInternet ─▶ "available but is limited"
        ├─ e.NetworkAccess != Internet ───────────▶ "has been lost"
        └─ siempre ─▶ "Connections active: " + [Bluetooth|Cell|Ethernet|WiFi]* + "\n"
```

Observaciones de lectura (descriptivas; el ejemplo es la plantilla de Microsoft Learn transcrita):

- ⚠️ Los nombres de perfil se concatenan sin separador (`Console.Write` sin espacio, líneas 31-40): con Wi-Fi y datos activos sale `Connections active: CellWiFi` (el orden depende del SO; no verificado).
- ⚠️ Desuscribir en el finalizador no libera nada en la práctica: el evento estático mantiene viva la referencia al delegado (y por él a la instancia), así que el GC no la recolecta y el finalizador no corre mientras siga suscripta. Es razonamiento sobre la semántica de .NET, no algo ejecutado en el repo.
- La salida va a `Console` (en Android termina en el log del dispositivo; destino exacto no verificado), no a la UI.
- Cuando el acceso vuelve a `Internet` no se imprime ningún mensaje de «recuperado»: sólo la línea de perfiles.

---

## 5. Esqueleto de la app `Ejemplo_Maui_Connectivity`

| Pieza | Contenido | Fuente |
|---|---|---|
| `MauiProgram.CreateMauiApp` | `UseMauiApp<App>()`, fuentes OpenSans, `AddDebug()` en DEBUG; **no** registra servicios ni páginas | `EMC/MauiProgram.cs:7-23` |
| `App` | `CreateWindow` → `new Window(new AppShell())` | `EMC/App.xaml.cs:12-15` |
| `AppShell` | `Title="Conexion"`, un `ShellContent` `Route="MainPage"` | `EMC/AppShell.xaml:8-13` |
| `MainPage` | `<ScrollView>` vacío; code-behind sólo `InitializeComponent()` | `EMC/Pages/MainPage.xaml:6-8`, `EMC/Pages/MainPage.xaml.cs:6-9` |
| `ConnectivityTest` | definido pero **sin ninguna instancia** en el repo (`git grep ConnectivityTest` sólo encuentra la clase y la nota) | `EMC/Utilities/ConnectivityTest.cs:7` |

```
Arranque:  MainApplication / AppDelegate ─▶ MauiProgram ─▶ App ─▶ AppShell ─▶ MainPage (vacía)
                                                                     ✗ nadie hace new ConnectivityTest()
```

⚠️ Al ejecutar la app se ve una página en blanco y no se escribe nada en consola: el ejemplo de conectividad está escrito, pero no conectado. Es coherente con el «En construcción …» del `README.md:17-18`.

---

## 6. Plataformas, permisos y build

### 6.1 Android

| Elemento | Valor | Fuente |
|---|---|---|
| `ACCESS_NETWORK_STATE` | declarado (requerido por `Connectivity` en Android; permiso «normal», no se pide en tiempo de ejecución) | `EMC/Platforms/Android/AndroidManifest.xml:10` |
| `INTERNET` | declarado | `AndroidManifest.xml:11` |
| SDK | `minSdkVersion=25`, `targetSdkVersion=36` (coincide con `SupportedOSPlatformVersion` 25.0) | `AndroidManifest.xml:13`, `.csproj:24` |
| Páginas de 16 KB | `AndroidLdFlags -Wl,-z,max-page-size=16384` | `.csproj:32` |
| RIDs | `android-arm;android-arm64;android-x86;android-x64` | `.csproj:33` |
| Empaquetado | `apk` en Debug y Release, sin APK por ABI; Debug con `EmbedAssembliesIntoApk` | `.csproj:36-48` |
| `MainActivity` | plantilla (`SingleTop`, `Maui.SplashTheme`) | `EMC/Platforms/Android/MainActivity.cs:7-10` |

El mismo par `ACCESS_NETWORK_STATE` + `INTERNET` aparece en 21 de los 22 `AndroidManifest.xml` versionados de la colección (falta sólo en `Ejemplos_Devices/GPS/Ejemplo_Maui_GPS`) porque viene con la plantilla MAUI (p. ej. `Ejemplos_Devices/Phone/Ejemplo_Maui_Dialer/Platforms/Android/AndroidManifest.xml:10`). Sólo la app híbrida lo comenta explícitamente como requisito de `IConnectivity` (`Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/Platforms/Android/AndroidManifest.xml:36-40`) → [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md).

### 6.2 iOS

| Elemento | Valor | Fuente |
|---|---|---|
| TFM | sólo si el host es macOS | `.csproj:5` |
| Versión mínima | 15.0 | `.csproj:23` |
| `Info.plist` | sin claves de privacidad (plantilla) | `EMC/Platforms/iOS/Info.plist:4-31` |
| Privacy manifest | entradas de plantilla (FileTimestamp, SystemBootTime, DiskSpace) | `EMC/Platforms/iOS/Resources/PrivacyInfo.xcprivacy:13-49` |
| Release | intérprete, `PublishTrimmed`, `MtouchLink SdkOnly`, `--linkskip=System.Net.Mail --linkskip=System.Net.Ping`, AOT sólo en `ios-arm64` | `.csproj:52-62` |

Los ajustes de Release iOS son los comunes de la colección; su porqué está en [09_CI-CD-y-Build](09_CI-CD-y-Build.md). Este proyecto no tiene workflow propio.

### 6.3 Windows

⚠️ Existen `EMC/Platforms/Windows/*` (App, `Package.appxmanifest` con placeholders, `app.manifest`) y el perfil `"Windows Machine"` (`EMC/Properties/launchSettings.json:3-6`), pero `net10.0-windows…` **no** figura en `TargetFrameworks` (`.csproj:4-5`): esos archivos no se compilan. Las propiedades `windows` de `.csproj:21,25-26` quedan inertes.

---

## 7. Trampas de lectura y discrepancias (⚠️)

| # | Trampa | Evidencia |
|---|---|---|
| 1 | El proyecto se llama `Connectivity` pero el namespace raíz es `Ejemplo_Maui_Conexion` (y `ApplicationTitle` «Maui Conexion»); buscá por ambos nombres | `.csproj:8,14`; `MauiProgram.cs:3` |
| 2 | `ConnectivityTest` nunca se instancia y `MainPage` está vacía: la app no demuestra nada al correrla | §5 |
| 3 | La nota usa «Ninguno»/«Desconocido»; el enum es `None`/`Unknown` | `Readme.md:18-19` |
| 4 | El `Readme.md` está en Windows-1252 (no UTF-8): los guiones largos (bytes `0x97`, líneas 16 y 18) y la «ó» de «autenticación» (línea 16) se ven como `�` en visores UTF-8 | `file Readme.md` → «Non-ISO extended-ASCII text» |
| 5 | El bloque de código de la nota es copia literal de `ConnectivityTest.cs` (sin `using` ni namespace): si cambia uno, el otro queda desfasado | `Readme.md:22-64` vs `ConnectivityTest.cs:7-49` |
| 6 | La nota muestra `Connectivity.Current.NetworkAccess` (instancia); `ConnectivityTest` usa el evento estático `Connectivity.ConnectivityChanged`. Ambas formas existen en MAUI | `Readme.md:7`; `ConnectivityTest.cs:10` |
| 7 | `NetworkAccess.Internet` dice que el SO **cree** que hay Internet; no valida que un sitio responda. La app híbrida agrega una sonda HTTP con cuerpo conocido para detectar portales cautivos | `Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/Networks/Services/NetworkService.cs:12-16,48-70` → [08](08_App-Hibrida-Integrada.md) |
| 8 | Carpeta `Platforms/Windows` y perfil «Windows Machine» sin TFM Windows | §6.3 |
| 9 | Mensajes de consola en inglés y perfiles concatenados sin separador | `ConnectivityTest.cs:18,21,24-40` |
| 10 | La línea «status» suelta de la nota es un rótulo sin desarrollar | `Readme.md:5` |

---

## 8. Relación con el resto de la colección

| Tema | Dónde se resuelve | Índice |
|---|---|---|
| Conectividad «de verdad» (servicio con `IConnectivity` inyectado, `IsOnline`, evento `Action<bool>`, resultado tipado `NetworkResult`, overlay de red sobre el `WebView`) | `Ejemplos_Devices/Integrada/Ejemplo_Maui_Hibrida/LibApp/Devices/Networks/**`, `Ejemplos_Devices/Docs/web-hibrida/maui-hibrido.md:92` | [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) |
| Patrón transversal servicio → resultado tipado → overlay | — | [00_MASTER-INDEX](00_MASTER-INDEX.md) |
| Entrada «Red y Conectividad» del `README.md` raíz | `README.md:17-18` | [10_Documentacion-Transversal](10_Documentacion-Transversal.md) |
| Registro en `.slnx` y ausencia de pipeline | `Ejemplos_Devices/Ejemplos_Devices.slnx:187-194` | [09_CI-CD-y-Build](09_CI-CD-y-Build.md) |

Comparación didáctica (qué agrega la híbrida sobre este esqueleto):

| Aspecto | `Ejemplo_Maui_Connectivity` | `Ejemplo_Maui_Hibrida` (`NetworkService.cs`) |
|---|---|---|
| Acceso a la API | estático `Connectivity.ConnectivityChanged` | `IConnectivity` por constructor (`:26-30`); se registra `Connectivity.Current` como singleton (`Integrada/Ejemplo_Maui_Hibrida/MauiProgram.cs:94`) |
| Criterio «online» | implícito (`!= Internet` ⇒ «lost») | `NetworkAccess == Internet` (`:32`) |
| Verificación real | no | sonda a `msftconnecttest.com` con marcador (`:15-16,58-68`) |
| Salida | `Console` | evento `Action<bool>` + `NetworkResult` tipado |
| Ciclo de vida | finalizador | singleton de DI: `AddSingleton<INetworkService, NetworkService>()` (`Integrada/Ejemplo_Maui_Hibrida/MauiProgram.cs:96`); detalle en [08](08_App-Hibrida-Integrada.md) |

---

## 9. Preguntas frecuentes

| Pregunta | Respuesta corta | § |
|---|---|---|
| ¿Qué permiso necesita Android para `Connectivity`? | `ACCESS_NETWORK_STATE` (declarado en el manifest; no se pide en runtime) | §6.1 |
| ¿Y iOS? | Nada en `Info.plist` | §6.2 |
| ¿Por qué no pasa nada al correr la app? | `MainPage` vacía y `ConnectivityTest` no se instancia | §5 |
| ¿Cómo lo pongo a funcionar? | Instanciar `ConnectivityTest` (p. ej. en `MainPage` o en `App`) y mirar la consola/log; o leer `Connectivity.Current.NetworkAccess` y mostrarlo en la UI. Sugerencia, no está en el repo | §3.1, §5 |
| ¿`Internet` garantiza que un sitio cargue? | No; ver la sonda activa de la híbrida | §7 #7 |
| ¿Dónde está el namespace `Ejemplo_Maui_Connectivity`? | No existe: es `Ejemplo_Maui_Conexion` | §7 #1 |
| ¿Corre en Windows? | No con el `.csproj` actual (sin TFM Windows) | §6.3 |
| ¿Tiene pipeline de CI? | No | §1 |
| ¿Por qué se ve `�` en el Readme? | Codificación Windows-1252 | §7 #4 |
