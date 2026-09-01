# 07 — Red y conectividad

> **Propósito:** la API `Connectivity` de MAUI Essentials — estados de acceso, perfiles activos y el evento de cambio.
> **Fuente primaria:** `Ejemplos_Devices/Red/` (1 proyecto + 1 documento).
> **Índices relacionados:** [00_MASTER-INDEX](00_MASTER-INDEX.md) · [08_App-Hibrida-Integrada](08_App-Hibrida-Integrada.md) (ahí sí hay un `NetworkService` completo con overlay).

---

## 1. Estado del dominio

`Ejemplo_Maui_Connectivity` · `com.ejemplos.red.connectivity` · sin paquetes fuera de MAUI.

**Es un esqueleto**: la pantalla está vacía (`Pages/MainPage.xaml` tiene un `<ScrollView>` sin contenido y el code-behind solo llama a `InitializeComponent`). El valor del dominio está en el documento y en la clase de ejemplo, no en una app funcionando. El `README.md` raíz lo marca como «En construcción …».

La versión operativa de este tema está en la app híbrida, con `NetworkService`, `NetworkResult` y overlay ([índice 08](08_App-Hibrida-Integrada.md)).

## 2. El modelo de `Connectivity`

Documentado en `Red/Ejemplo_Docs_Red/Readme.md`. Se consulta con:

```csharp
NetworkAccess accessType = Connectivity.Current.NetworkAccess;
```

| Valor de `NetworkAccess` | Significado |
|--------------------------|-------------|
| `Internet` | Acceso local **e** Internet |
| `ConstrainedInternet` | Acceso limitado: hay un **portal cautivo**; tras autenticarse en él se concede Internet |
| `Local` | Solo red local |
| `None` | Sin conectividad |
| `Unknown` | No se puede determinar |

Y los perfiles activos (`ConnectionProfile`): `Bluetooth`, `Cellular`, `Ethernet`, `WiFi`.

## 3. `ConnectivityTest` — el ejemplo del evento

`Utilities/ConnectivityTest.cs` (49 l.) es **la clase de la documentación de Microsoft transcrita al proyecto**, idéntica al bloque de código que reproduce `Ejemplo_Docs_Red/Readme.md`. Muestra:

- Suscripción en el constructor y desuscripción **en el finalizador** (`~ConnectivityTest()`) a `Connectivity.ConnectivityChanged`.
- En el handler: distinguir `ConstrainedInternet` de la pérdida de acceso, y recorrer `e.ConnectionProfiles` para listar las conexiones activas.

⚠️ **Nadie la instancia**: no hay ninguna referencia a `ConnectivityTest` fuera de su propio archivo, y desuscribirse desde un finalizador no es una técnica recomendable (el evento estático mantiene viva la instancia, así que el finalizador puede no ejecutarse nunca). Es material de lectura, no un patrón a copiar.

## 4. Observaciones

- **El nombre del proyecto y el del namespace no coinciden:** la carpeta y el `.csproj` son `Ejemplo_Maui_Connectivity`, pero `<RootNamespace>` es `Ejemplo_Maui_**Conexion**` y todo el código declara `Ejemplo_Maui_Conexion.*`. El `ApplicationTitle` es «Maui Conexion».
- El `.csproj` declara `SupportedOSPlatformVersion` para **windows** (`10.0.17763.0`) y existe `Platforms/Windows/App.xaml.cs`, pero **el TFM de Windows no está en `TargetFrameworks`**: esa condición nunca aplica.
- No tiene workflow de CI ([índice 09](09_CI-CD-y-Build.md)).

## 5. Fuentes

| Ruta | Contenido |
|------|-----------|
| `Red/Ejemplo_Docs_Red/Readme.md` | Los 5 valores de `NetworkAccess` y el ejemplo del evento |
| `Red/Ejemplo_Maui_Connectivity/Utilities/ConnectivityTest.cs` | Suscripción a `ConnectivityChanged` y recorrido de perfiles |
