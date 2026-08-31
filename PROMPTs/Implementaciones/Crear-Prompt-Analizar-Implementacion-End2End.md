# Tool-Prompt — Analizar implementación PoC en Producción

> **Invocación**:
> - `Lee y ejecuta Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Implementaciones/Crear-Prompt-Analizar-Implementacion-End2End.md`

---

# Contexto

Lee `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/ia-db/README.md`
Lee `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/Ejemplos_Maui_Devices-docs/README.md`

Se cuentan las siguientes piezas relaciondas:
1. `Ejemplo_Maui_Hibrida` es una aplicación maui.
2. `/Ejemplos_Maui\Ejemplos_Maui_Devices\.github\workflows\cd-ios-Integrada.Ejemplo_Maui_Hibrida.yml`. Es el workflow para compilar en github actions.
3. `/Ejemplos_Maui\Ejemplos_Maui_Devices\Utilities\simular_ui.sh` es el script llamado por una regla del workflow de github actions.
4. `Ejemplos_Maui\Ejemplos_Maui_Devices\Utilities\end2end\com.ejemplos.devices.integrada.hibrida.yaml` es el flujo de trabajo que seguirá `simular_ui.sh`

Es una forma de hacer una prueba end2end sobre la aplicación

---

# Objetivo

- Crear un prompt para que sirva para diseñar el flujo de prueba end2end similar a `Ejemplos_Maui\Ejemplos_Maui_Devices\Utilities\end2end\com.ejemplos.devices.integrada.hibrida.yaml`.

---

# Solicitudes


1. Evaluar, observar y entender esta tecnica de simular y capturar el video.

2. Propone alternativas para que un prompt construya el flujo de trabajo explorando por si mismo la aplicación. Una posibilidad es que se conecte a un telefono como desarrollador usb y haga un barrido en la UI de la aplicación y genere el story board por ponerle un nombre de la historia. Otra forma , sería que el prompt lance la aplicación android y vaya capturando la navegación del usuario, así luego la ejecuta sobre el simualdor por medio de este flujo de trabajo. O bien tal vez el prompt podría proponer ambos como opción y ahi se elige cual

3. El prompt resultante lo podes dejar en `/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/PROMPTs/Implementaciones/Crear-Flow-End2End-App.md`

4. Realimenta con esta tecnica end2end, sus posiblidades, herramientas en ``/Ejemplos_Maui/Ejemplos_Maui_Devices.Documentacion/Ejemplos_Maui_Devices-docs/README.md``

---

# Framework

## Rule Set

- `/IA/IA.Prompts/PromptFramework/RuleSets/RuleSet-Lean.md`
