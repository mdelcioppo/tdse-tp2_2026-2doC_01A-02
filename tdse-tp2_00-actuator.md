Continuando con nuestra dinámica de análisis, llegamos a la etapa final de tu arquitectura: el subsistema actuador. Este módulo se encarga de traducir las decisiones lógicas tomadas por el módulo "System" en acciones físicas en el hardware.

### Análisis Funcional de los Archivos

* **`task_actuator_attribute.h`**: Define el diccionario de datos del actuador, separando la configuración estática de solo lectura (`task_actuator_cfg_t`, como el puerto y pin GPIO) de la memoria dinámica (`task_actuator_dta_t`). Define los estados (`ST_LED_IDLE`, `ST_LED_ACTIVE`) y eventos esperables.


* **`task_actuator_interface.c`**: Expone un mecanismo de comunicación directo (sin cola FIFO) para inyectar eventos en el actuador mediante `put_event_task_actuator()`.


* **`task_actuator.c`**: Contiene la máquina de estados del hardware de salida, resolviendo cuándo y cómo cambiar los niveles lógicos del microcontrolador basándose en los comandos recibidos.



---

### Evolución de Variables (Desde `init` hasta `update`)

Al ejecutar el código, la memoria dinámica del actuador evoluciona de la siguiente forma:

* **`index`**: Tanto en `task_actuator_init()` como en `task_actuator_update()`, itera desde 0 hasta `ACTUATOR_DTA_QTY - 1` (es decir, solo el índice 0, correspondiente a `ID_LED_A`).


* **`task_actuator_dta_list[index].tick`**: *Unidad de medida: Ticks temporales (milisegundos)*. Al igual que en el módulo System, no se inicializa de forma explícita ni se usa para demoras, sino que se asigna a 0 (`DEL_LED_MIN`) únicamente como medida de seguridad si la máquina cae en el caso `default`.


* **`task_actuator_dta_list[index].state`**: Inicia forzosamente en `ST_LED_IDLE` durante el `init`. Cambiará dinámicamente a `ST_LED_ACTIVE` en el bucle principal solo cuando se procese un comando válido de encendido.


* **`task_actuator_dta_list[index].event`**: Se inicializa en `EV_LED_IDLE`. Su valor se mantendrá hasta que el módulo externo System lo sobrescriba.


* **`task_actuator_dta_list[index].flag`**: Comienza en `false`. Indica si hay un evento sin procesar.



---

### Comportamiento de `task_actuator_statechart(uint32_t index)`

Esta función es netamente reactiva y se ejecuta en cada iteración del planificador:

1. **Lectura de estado:** Evalúa en qué estado lógico se encuentra el LED (`ST_LED_IDLE` o `ST_LED_ACTIVE`).


2. **Evaluación de Transición (`ST_LED_IDLE`):** Si el sistema está inactivo, revisa la bandera de nuevo evento (`flag == true`) y si el evento exigido es `EV_LED_ACTIVE`. De cumplirse ambas, limpia la bandera (`flag = false`), escribe el pin físico a nivel alto (`led_on`) mediante la librería HAL, y avanza al estado activo.


3. **Evaluación de Transición (`ST_LED_ACTIVE`):** De manera inversa, si el sistema está activo, exige `flag == true` y `EV_LED_IDLE` para limpiar la bandera, apagar el pin físico (`led_off`) y volver a reposo.


4. **Manejo de Fallas:** Cualquier estado corrupto es capturado por `default`, forzando al actuador a un estado de reposo seguro.



---

### Evolución de las Variables de Interfaz (Comunicación Inter-Tarea)

Cuando la tarea System toma la decisión de accionar el LED, invoca `put_event_task_actuator()` en `task_actuator_interface.c`. Esto altera las variables de la siguiente manera:

* **`identifier`**: El sistema provee la constante `ID_LED_A` (0) para direccionar los datos hacia el LED correcto en el arreglo de actuadores.


* **`task_actuator_dta_list[identifier].event`**: Se sobrescribe inmediatamente con el nuevo comando a ejecutar (ej. `EV_LED_ACTIVE`).


* **`task_actuator_dta_list[identifier].flag`**: Pasa a valer `true`. Esto actúa como un "semáforo" que avisa a `task_actuator_statechart` que debe procesar la variable `event` en su próxima ejecución programada.



Considerando que el sensor usa una cola FIFO y el actuador usa un paso de mensaje directo con bandera, ¿comprendes por qué se eligieron estrategias de acoplamiento distintas para las señales de entrada respecto a las de salida en este sistema embebido?