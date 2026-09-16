Continuando con el análisis de la arquitectura del Trabajo Práctico, estos archivos implementan la lógica central de control (la tarea "System") y su comunicación con los actuadores. El sistema actúa como un intermediario: procesa las señales encoladas por el sensor y decide qué comandos despachar hacia el hardware de salida.

A continuación, te explico técnicamente su funcionamiento y la evolución de sus variables.

### Análisis Funcional de la Arquitectura

1. **`task_system_attribute.h` y `task_actuator_attribute.h**`: Definen las estructuras de datos, eventos (ej. `EV_SYS_ACTIVE`, `EV_LED_ACTIVE`) y estados (ej. `ST_SYS_IDLE`, `ST_LED_IDLE`) para el controlador central y el actuador, respectivamente.


2. **`task_system_interface.c`**: Gestiona la cola circular (FIFO) de entrada para la tarea System. Las funciones definidas aquí permiten inicializar la cola, insertar eventos (usado por el sensor) y extraer eventos (usado por el sistema).


3. **`task_system.c`**: Contiene la máquina de estados principal. Su rol es consumir los eventos de la cola (desacoplando el tiempo de lectura del tiempo de ejecución) y ejecutar las transiciones lógicas.


4. **`task_actuator_interface.c`**: A diferencia de la cola del sistema, implementa un paso por mensaje directo (sin cola) hacia el actuador mediante la función `put_event_task_actuator()`.



---

### Evolución de Variables del Sistema (System)

Desde la inicialización en `task_system_init()` hasta las ejecuciones de `task_system_update()`:

* **`index`**: En la inicialización, itera desde 0 hasta `SYSTEM_DTA_QTY - 1` (en este caso 0, ya que solo hay un modo `NORMAL` configurado). Durante `task_system_update()` no se utiliza un iterador `index` de forma explícita, sino que se accede directamente al índice `NORMAL`.


* **`task_system_dta_list[index].tick`**: *Unidad de medida: Ticks temporales (milisegundos)*. Esta variable solo se asigna al valor `DEL_SYS_MIN` (0) si la FSM cae en el caso `default` para recuperarse de un error. No se utiliza para conteo activo en este código.


* **`task_system_dta_list[index].state`**:
* En `task_system_init()` arranca forzosamente en `ST_SYS_IDLE`.


* En `task_system_update()`, alterna entre `ST_SYS_IDLE` y `ST_SYS_ACTIVE` al procesar los eventos extraídos de la cola.




* **`task_system_dta_list[index].event`**:
* En la inicialización comienza en `EV_SYS_IDLE`.


* En la actualización, se sobrescribe con el valor retornado por `get_event_task_system()` cada vez que se detecta un nuevo evento en la cola.




* **`task_system_dta_list[index].flag`**:
* En la inicialización se fija en `false`.


* En la actualización, toma el valor `true` cuando se extrae un nuevo evento de la cola. Luego, la máquina de estados evalúa la transición y, si consume el evento, resetea la bandera a `false` inmediatamente.





---

### Comportamiento de la Función Statechart del Sistema

(Nota: En el código proporcionado, la función no recibe un índice por parámetro, sino que se llama `void task_system_normal_statechart(void)` y usa directamente el índice `NORMAL`).

Su ejecución consta de dos etapas:

1. **Lectura de Cola:** Consulta `any_event_task_system()`. Si hay datos (es decir, el sensor dejó un mensaje), extrae el evento llamando a `get_event_task_system()`, lo guarda en la estructura y levanta la bandera `flag = true`.


2. **Evaluación de Máquina de Estados:**
* Si está en `ST_SYS_IDLE`, hay un evento pendiente (`flag == true`) y este es `EV_SYS_ACTIVE`: baja la bandera, despacha un evento al LED mediante `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)` y avanza al estado `ST_SYS_ACTIVE`.


* Si está en `ST_SYS_ACTIVE`, hay un evento pendiente y es `EV_SYS_IDLE`: baja la bandera, envía `put_event_task_actuator(EV_LED_IDLE, ID_LED_A)` y regresa a `ST_SYS_IDLE`.





---

### Evolución de la Cola FIFO (`event_task_system_queue`)

Estas variables controlan el búfer circular donde el sistema lee sus eventos:

* **`i`**: Es una variable local en `init_event_task_system()` que itera de 0 a 15 (`QUEUE_LENGTH - 1`) para limpiar el búfer.


* **`event_task_system_queue.queue[i]`**: En la inicialización, todos los elementos se llenan con el valor `EMPTY` (255). Cuando el sistema lee un evento mediante `get_event_task_system()`, extrae el valor útil y vuelve a escribir `EMPTY` en esa posición de memoria.


* **`event_task_system_queue.head`**: Puntero de escritura. No se modifica durante la ejecución de las funciones del sistema; su valor solo crece cuando un productor (como el sensor) inserta datos.


* **`event_task_system_queue.tail`**: Puntero de lectura. Inicia en 0. Durante las actualizaciones, si la máquina de estados consume un evento (llamando a `get_event_task_system()`), `tail` se incrementa en 1 para apuntar al siguiente mensaje. Si llega a 16 (`QUEUE_LENGTH`), hace *wrap-around* y vuelve a 0.


* **`event_task_system_queue.count`**: Inicia en 0. Cuando `task_system_normal_statechart()` extrae un evento, esta variable se decrementa en 1.



---

### Evolución de las Variables de Interfaz del Actuador

Estas variables se modifican exclusivamente cuando el Sistema dispara una transición y llama a la interfaz del actuador:

* **`identifier`**: En la llamada `put_event_task_actuator()`, la FSM del sistema pasa el parámetro fijo `ID_LED_A`.


* **`task_actuator_dta_list[identifier].event`**: Se sobrescribe de forma directa y asíncrona con el evento enviado por el sistema (`EV_LED_ACTIVE` o `EV_LED_IDLE`).


* **`task_actuator_dta_list[identifier].flag`**: Cada vez que se ejecuta `put_event_task_actuator()`, esta bandera se impone en `true`, avisándole a la tarea Actuator (que correrá en el futuro) que tiene un nuevo evento sin procesar.



¿Te resulta claro cómo este diseño garantiza que el sistema no se quede bloqueado esperando la respuesta del actuador?