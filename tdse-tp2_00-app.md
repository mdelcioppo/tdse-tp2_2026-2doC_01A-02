Comencemos el análisis de la arquitectura del software. Este sistema implementa un planificador (scheduler) cooperativo del tipo "Bare Metal" disparado por eventos temporales, comúnmente llamado planificador *Time-Triggered* o *Event-Triggered* en sistemas embebidos.

### Análisis del Funcionamiento de los Archivos

* **`app.c`**: Es el motor principal del sistema. Define la lista de tareas a ejecutar (`task_cfg_list`) y almacena las métricas de rendimiento (profiling) de cada tarea en `task_dta_list`. Contiene el bucle principal que decide cuándo deben correr las tareas basándose en los ticks del sistema temporal.


* **`app_it.c`**: Concentra las rutinas de servicio para las interrupciones específicas de la aplicación. En particular, maneja la interrupción del temporizador del sistema (SysTick) mediante `HAL_SYSTICK_Callback()`, la cual incrementa la base de tiempo de la aplicación.


* **`dwt.h`**: Expone una interfaz en línea (inline) para el periférico de hardware DWT (Data Watchpoint and Trace) del núcleo ARM Cortex. Esto proporciona un contador de ciclos de reloj extremadamente preciso e independiente, permitiendo medir el tiempo de ejecución del código en microsegundos mediante `cycle_counter_get_time_us()`.


* **`logger.c` y `logger.h**`: Implementan una infraestructura de registro de eventos o telemetría. Formatean los mensajes mediante `snprintf` y los envían a través de la interfaz de depuración (semihosting) deshabilitando previamente las interrupciones para crear una sección crítica segura.


* **`systick.c`**: Ofrece una utilidad adicional independiente (`systick_delay_us()`) para generar retardos bloqueantes precisos a nivel de microsegundo leyendo los registros crudos del hardware SysTick.



---

### Evolución de Variables (Desde `app_init` hasta `app_update`)

#### 1. Fase de Inicialización (`app_init()`)

Al arrancar el sistema, las variables se establecen en sus estados seguros:

* **`g_app_tick_cnt`**: Se inicializa en 0 a través de la función `app_it_init()`. Su unidad es adimensional (cantidad de ticks temporales).


* **`index`**: Es la variable local de iteración de bucles. En `app_init()`, itera desde 0 hasta la cantidad total de tareas configuradas menos uno (`TASK_QTY - 1`) para inicializar cada módulo.


* Para cada tarea en el arreglo `task_dta_list[index]`:


* **`.NOE` (Number Of Executions)**: Se inicializa en 0 (`TASK_X_NOE_INI`). Unidad: Adimensional.
* **`.LET` (Last Execution Time)**: Se inicializa en 0 (`TASK_X_LET_INI`). Unidad: Microsegundos (µs).
* **`.BCET` (Best-Case Execution Time)**: Se inicializa en 1000 (`TASK_X_BCET_INI`). Unidad: Microsegundos (µs). Se preasigna un valor alto de cota superior inicial.
* **`.WCET` (Worst-Case Execution Time)**: Se inicializa en 0 (`TASK_X_WCET_INI`). Unidad: Microsegundos (µs). Se preasigna la mínima cota inferior posible.



#### 2. Fase de Ejecución Continua (`app_update()`)

Durante el bucle principal, la dinámica temporal del sistema cobra vida de forma cíclica:

* **`g_app_tick_cnt`**: En segundo plano (asíncronamente), la interrupción de hardware del SysTick la incrementa en 1 en cada llamada a `HAL_SYSTICK_Callback()`. Sincrónicamente, dentro de `app_update()`, la rutina decrementa esta variable (`g_app_tick_cnt--`) bajo una sección crítica (interrupciones deshabilitadas con `CPSID i`). Esto vacía el "buzón" de ticks acumulados, procesándolos uno a uno.


* **`g_app_runtime_us`**: Al iniciar una nueva iteración de tareas de un tick, se reinicia a 0. Luego, acumula secuencialmente el tiempo `LET` de cada tarea mediante `g_app_runtime_us += task_dta_list[index].LET`. Al finalizar el ciclo `for`, refleja el tiempo total de ejecución de *todas* las tareas combinadas en esa iteración. Unidad: Microsegundos (µs).


* **`index`**: Vuelve a iterar de 0 a `TASK_QTY - 1` para ejecutar en secuencia la función de actualización de cada módulo (`task_update`).


* Durante el recorrido (`task_dta_list[index]`):


* **`.NOE`**: Se incrementa secuencialmente en 1 (`NOE++`) indicando que la tarea acaba de ejecutarse un ciclo más.
* **`.LET`**: Se sobreescribe obteniendo la diferencia de tiempo cruda del DWT (`cycle_counter_get_time_us()`). Representa exactamente cuánto tiempo en hardware tomó procesar esa tarea.
* **`.BCET`**: Si el `LET` medido fue estrictamente menor que el `BCET` histórico (`BCET > LET`), se actualiza su valor al de `LET`. Almacena la iteración más rápida histórica.
* **`.WCET`**: Si el `LET` medido fue estrictamente mayor que el `WCET` histórico (`WCET < LET`), se actualiza su valor al de `LET`. Almacena el pico máximo de latencia.



---

### Impacto de utilizar `LOGGER_INFO()` en el perfilado

Insertar una macro `LOGGER_INFO()` dentro de la lógica de una tarea alterará drásticamente la dinámica temporal del sistema, impactando de forma directa en dos variables:

1. **Impacto en `task_dta_list[index].WCET**`: El protocolo de semihosting utilizado en `LOGGER_INFO()` es una operación de Entrada/Salida muy lenta porque requiere intervención del depurador por hardware y formateo de cadenas `snprintf`. Mientras se imprime el log, el contador de ciclos DWT sigue avanzando. En consecuencia, el tiempo de ejecución `LET` de la tarea en ese ciclo disparará un pico de duración anormal. El condicional capturará ese pico, provocando que `WCET` registre un falso peor caso de ejecución, arruinando el perfilado real del algoritmo.


2. **Impacto en `g_app_runtime_us**`: Dado que esta variable acumula secuencialmente los `LET` de todas las tareas ejecutadas en un ciclo, la demora inducida artificialmente por el loguero se sumará linealmente a `g_app_runtime_us`. Esto reflejará una carga de CPU inflada en esa iteración temporal.



Adicionalmente, debido a que `LOGGER_INFO` deshabilita temporalmente las interrupciones (`CPSID i`), un tiempo excesivo de logueo puede bloquear la ejecución asíncrona del `HAL_SYSTICK_Callback()`, resultando en pérdida de métricas de ticks (`g_app_tick_cnt`) y un corrimiento severo en el planificador cooperativo temporal de `app_update()`.

¿Sobre qué sección específica de la inicialización o arquitectura de las tareas de la aplicación te gustaría profundizar para el Trabajo Práctico?