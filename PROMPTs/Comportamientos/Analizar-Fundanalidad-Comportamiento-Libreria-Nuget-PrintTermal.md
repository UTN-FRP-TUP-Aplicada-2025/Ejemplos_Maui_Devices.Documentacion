# Tool-Prompt — Analizar capacidades de librería 

> **Invocación**:
> - `Lee y ejecuta /Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Comportamientos/Analizar-Fundanalidad-Comportamiento-Libreria-Nuget-PrintTermal.md`

---

# Contexto

Lee `/Librerias/PrintThermal_Motor_Maui.Documentacion/ia-db/README.md`, es todo lo relacionado a las capacidades de los comandos Esc-Pos y las posibilidades que pueden ofrecer estas impresoras en valuar manejo de fallos. Por ejemplo, atasco de papel, falta de papal, bloqueo, o problemas de buffer, entre otros.

Lee `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/ia-db/README.md` en lo relacionado con los proyectos `Ejemplo_Maui_Hibrida` y `Ejemplo_MotorDSL_Dialog`. Estos dos proyectos usan las librerias nuget `MotorDsl.*`. y son dos aplicaciones PoC (Proff of Concept)
Centrate en estas dos aplicaciónes en la gestión de la impresión, manejos de erroes de impresión, flujo de usuario con respecto a la impresión actules. 

En el proyecto `Ejemplo_Maui_Hibrida` es un PoC (Proof of Concept) y es una aplicación hibrida que que mediante ciertos parametros, con el parametro `action=print` inicio una impresión. Actualmente la parte web es una aplicación blazord con páginas interative service. Esta aplicación web es `Ejemplo_ws_Blazor` y está indexada en `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/ia-db/README.md`

En `Ejemplo_Maui_Hibrida` integra el uso de multiples dispositivo del dipositivo móvil manejado con un patrón de dialogos overlay. Es permite manejar la comunicación con el usuario ante situaciones de solicitud y denegación de estos de acceso a dispositivos, mensajes de esperas, y errores dentro del patrón mvvm de maui.


# Objetivo


Verificar los flujos de usuarios y experiencias de usuarios actuales (UX, UI)
Relevar comportamientos contemplados y evaluar posibles comportamientos no previstos que puedan ocurrir en este tipo de impresiones.
Evaluar mediante la UI actual resolver los comportamientos contemplados y a contemplar mejorando la experiencia de usuario.
Evaluar flujos de usuarios alternativos convenientes a proponer.
Documentar la implementación ilustrando con definiciones y patrones utilizados con el objetivo de que un desarrollador tome la experiencia desde los documentos (usar diagramas, snipped, descripciones, explicaciones)
Proponer mejoras a aplicar en los proyectos PoC para la mejora de la experiencia de usuario y flujo de usuario en materia de los dialogos de impresión en el patrón de dialogos propuestos como overlay.
Generar la información que responda las siguientes preguntas:
¿Situaciones de fallos posibles?
¿Comó se identifica el dispositivo? Por su modelo o por su mac. Como se resuelve la selección de una impresora en caso de haber varias
del mismo modelo.

Se relevó de forma manual, se obtuvo la siguiente información:

1. **Casos de usos**

1.1. **Listar impresoras disponibles**
Lista los dispositivos apareados o sincronizados

Ejemplo en el que se listan dos impresoras registradas:

```
# Elegí una impresora

  Se detectaron varias
 impresoras disponibles.

       [MTP-II]
       [58HB6]
       [Cerrar]	   
``` 

1.a **Flujos alternativos**

- **No se han solicitado los permisos en android aún**

Android solicita los permisos:
```
¿Permitir que Web 
Hibrida encuentre
dispositivos
cercanos, se 
conecte a ellos
y determine su
ubicación relativa?
```

- **Se ha denegado la primera ver los permisos (Solo en android)**

Android solicita los permisos:
```
# Permiso Bluetooth necesario

Para imprimir necesitamos 
acceso al bluetooth. Podés 
intentar concederlo.

[Pedir Permiso]
[Cerrar]

```
El botón [Pedir Permiso] abre ajustes de la aplicación.
El botón [Cerrar] vuelve a la pantalla origen.


- **Se ha denegado permanente**

Android solicita los permisos:
```
# Acceso Bluetooth denegado

Habilitá el permiso de 
Bluetooth desde los ajustes
de la aplicación

[Abrir Configuración]
[Cerrar]

```

El botón [Abrir Configuración] abre ajustes de la aplicación.
El botón [Cerrar] vuelve a la pantalla origen


- **No está encendido el bluetooth**

```
# No se encontraron Impresoras
Encendé la impresora y volvé a intentar.
           [Reintentar]
            [Cerrar]
```
Nota: debería informar que no está encendido bluetooth,  debería mostrar un botón adicional: [Activar bluetooth] y con esto abrir ajustes del SO en la parte de dispositivos bluetooth

- No hay impresoras sincronizada. Bluetooth encendido.
```
# No se encontraron Impresoras
Encendé la impresora y volvé a intentar.
           [Reintentar]
            [Cerrar]
```
**Nota**: es el mismo mensaje que el que cuando no encontró impresoras.
Eso esta bien, pero podría sugerir que no encuentra sincronizado impresoras y tambien agregar el botón adicional [Sincronizar impresora] y con esto abrir ajustes del SO en la parte de dispositivos bluetooth.

2. **Configurar impresora**
Usa el caso de uso #1 para mostrar las impresoras disponibles. El usuario elige una impresora. 

3. **Imprimir**

- **La aplicación se conecta al dispositivo configurada: **

```
# Conectando 

Conectando a 
58HB6...

```

- **La aplicación envía el documento a la impresora**:
```
# Imprimiendo 

Enviando el 
documento ...
58HB6...

```

El dispositivo imprime el documento. Sino terminó por falta de papel o algun tema no se verifica. La inspección es visual, el usuario debe verificar cual fue el problema y volver a iniciar la impresión. Esto esta bien. La causa de reeimprimir, puede ser la falta de papel o que la impresión quedó incompleta, fallos en la comunicación, etc.



# Solicitudes

1. Evaluar la aplicación dada por el proyecto conforme en los objetivos planteados.

2. Generar un documento que reuna los puntos planteados en el documento anterior en `/Librerias/PrintThermal_Motor_Maui.Documentacion/Analisis/Analisis-UX-UI.md`

---

# Framework

## Profile

Aplicar:

- `/IA/IA.Prompts/PromptFramework/Profiles/Code-Review.md`




