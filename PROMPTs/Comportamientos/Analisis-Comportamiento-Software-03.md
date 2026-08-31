
# Tool-Prompt — Evaluar comportamientos

> **Invocación**:
> - `Lee y ejecuta /Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-03.md`

---

# Contexto

Lee `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/ia-db/README.md`, la solución a la que indexa ese documento esta en el repositorio `/Ejemplos_Maui/Ejemplos_Maui_Devices`

Estoy probando manualmente la aplicación MAUI .NET hibrida `Ejemplo_Maui_Hibrida` cuyo
servicio web es: `Ejemplo_ws_Blazor` y sirve de las paginas web a la aplicación móvil.

Algunos comportamientos observados en pagina blazor `/Panel` desde la aplicación móvil:

1. **Solicitar coordeandas**

a. Primera vez. 

Como es la primera vez, Pide los permisos.

`/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-03/Primera.md`

b. Segunda vez. 

Directamente va a tomar las coordeandas.

`/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion\PROMPTs/Comportamientos/Analisis-Comportamiento-Software-03/Segunda.md`

c. Primera vez. Pide los permisos.

Directamente va a tomar las coordeandas.

`/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analisis-Comportamiento-Software-03/Tercera.md`


Pero en todos los casos se cierra el overlay y la coordenada no se ve hasta un rato despues.

El comportamiento aceptable es que mientras no se tenga la coordenada en la pantalla como corresponde no debería salir de la pantalla de espera que provee el overlay. Es razonable que cuando termine el gps de calcular recien ahi se notifique la baja de la pantalla de esper. Si el gps cancelo porque tardo mucho, la pantalla de espera se cae porque en si el gps termino (aunque sea forzado por el timeout)

---

# Solicitudes

1. Evaluar, diagnosticar comportamientos no deseados y proponer una planificación de según los objetivos propuestos sobre los proyectos `Ejemplo_Maui_Hibrida` y `Ejemplo_ws_Blazor`.