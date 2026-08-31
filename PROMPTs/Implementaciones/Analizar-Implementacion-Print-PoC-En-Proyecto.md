# Tool-Prompt — Analizar implementación PoC en Producción

> **Invocación**:
> - `Lee y ejecuta /Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Implementaciones/Analizar-Implementacion-Print-PoC-En-Proyecto.md`

---

# Contexto


Lee `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/ia-db/README.md`

En el proyecto `Ejemplo_Maui_Hibrida`hay una implementación parcial del comando `action=print`. La implementación de la impresora bluetooth sigue la implementación dada en el proyecto `Ejemplo_MotorDSL_Dialog`.


---

# Objetivo

- Determinar los ajustes finales en `Ejemplo_Maui_Hibrida` para la implementación del comando `action=print`

---

# Solicitudes


- Evaluar, observar y entender la arquitectura el proyecto  `Ejemplo_MotorDSL_Dialog` extrayendo el patron de implementación de la impresora bluetooth enfocado principalmente en los dialogos utilizados para manejar el estado de la impresora.
- Evaluar, observar y entender la arquitectura de implementacion de comandos usado en el proyecto `Ejemplo_Maui_Hibrida` para con los diferentes dispositivos GPS, Camera, entre otros.
- Proponer los ajustes necesarios para completar la funcionalidad del comando `action=print` en base al PoC dado por el proyecto patrón: `Ejemplo_MotorDSL_Dialog` en el proyecto: `Ejemplo_Maui_Hibrida`. El dialogo overlay de la impresión funcionen correctamente siguiendo como patrón ejemplificado en `Ejemplo_MotorDSL_Dialog`


---

# Framework

## Rule Set

- `/IA/IA.Prompts/PromptFramework/RuleSets/RuleSet-Lean.md`




