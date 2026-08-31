
# Tool-Prompt — Evaluar comportamientos

> **Invocación**:
> - `Lee y ejecuta /Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02.md`

---

# Contexto

Lee `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/ia-db/README.md`, la solución a la que indexa ese documento esta en el repositorio `/Ejemplos_Maui/Ejemplos_Maui_Devices`

Estoy probando manualmente la aplicación MAUI .NET hibrida `Ejemplo_Maui_Hibrida` cuyo
servicio web es: `Ejemplo_ws_Blazor` y sirve de las paginas web a la aplicación móvil.

Algunos comportamientos observados en pagina blazor `/Panel` desde la aplicación móvil:

1. **Solicitar coordeandas**

a. En el boton [Tomar Coordenadas] devuelves las coodenadas, las muestra de bajo , no se observa el separador decimal de latitud y longitud.

b. Como comportamiento no deseado, se observa que el dialogo se cierra en seguida cuando debería cerrarse despues de obtener las coordenadas indicando la espera de la consulta al dispositivo. Aparentemente esto parece una logica del overlay en cuanto a como trabaja respecto a la consulta asincrona del dispositivo


2. **LLamar y reporte de posición**
Correcto!

3. **Tomar foto**
3.a Tomar Foto un solo click en botón `[Tomar Foto]`, v
Correcto!
Ver Log maui .net: `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02/Tomar-Foto-Un-Click.md`

3.b Tomar Foto, varios clicks en botón `[Tomar Foto]`, 
Como comportamiento no deseado, se observo que la foto se hizo y luego refrescó la página web. No debió refrescarse la página web
Ver Log maui .net: `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02/Tomar-Foto-Varios-Clicks.md`


4. **Tomar selfie**

4.a  Un solo click en botón `[Tomar Selfie]`
Correcto!
Ver Log maui .net: `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02/Tomar-Foto-Un-Click.md`

4.b Varios clicks en botón  `[Tomar Selfie]`, 
Incorrecto!
Como comportamiento no deseado, se observó que la foto se hizo, se mostró un breve periodo de tiempo y luego refrescó la página web.
Ver Log maui .net: `/Ejemplos_Maui\Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02/Tomar-Selfie-Varios-Clicks.md`

5. **Post API**
Correcto!
Ver log: `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02/Post-API.md` 

6. **Lectura QR**

6.a Un solo click en el botón `[Leer QR]`
Correcto!
Ver Log maui .net:`/Ejemplos_Maui\Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02/Lectura-QR-Un-Click.md` 

6.b Varios clicks en el botón `[Leer QR]`
Incorrecto!
Como comportamiento no deseado, se observó como comportamiento no deseado, que luego de tomar el QR con el diálogo de toma de QR se refresco la página, cuando debio mostrar el valor del QR.
Ver Log maui .net: `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-02\Lectura-QR-Varios-Clicks.md` 


Conclusión
Sobre lo que es tomar coordeandas, tomar foto, leer qr , cuando el usuario solicita varias veces el tomar foto o tomar qr genera comportamientos no deseados. 

---

# Objetivos

Proponer resoluciones a los comportamientos no deseados.

---

# Solicitudes

1. Evaluar, diagnosticar comportamientos no deseados y proponer una planificación de según los objetivos propuestos sobre los proyectos `Ejemplo_Maui_Hibrida` y `Ejemplo_ws_Blazor`.