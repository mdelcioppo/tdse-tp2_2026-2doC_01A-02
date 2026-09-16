Continuando con nuestra dinámica de clases, analizaré la arquitectura de estos módulos, los cuales implementan el patrón de diseño de **Máquinas de Estados Finitos (FSM)** acopladas mediante una **Cola de Mensajes (Message Queue)**.

### Análisis Funcional de los Archivos

1. **`task_sensor_attribute.h` y `task_system_attribute.h**`: Definen la estructura de datos (Data y Config), los estados (ej. `ST_BTN_IDLE`) y los eventos (ej. `EV_BTN_DOWN`) para los módulos Sensor y System, respectivamente. El uso de estructuras estáticas de configuración (`cfg`) separadas de las variables dinámicas de estado (`dta`) es una excelente práctica para ahorrar memoria RAM.


2. **`task_system_interface.c`**: Implementa el mecanismo de comunicación entre tareas (Inter-Task Communication) mediante una cola circular (FIFO) no bloqueante. Permite que el Sensor envíe eventos asíncronos al Sistema sin acoplar lógicamente la ejecución de ambas tareas.


3. **`task_sensor.c`**: Implementa la lógica de la máquina de estados del sensor. Su propósito es leer periódicamente (polling) el estado del hardware, traducir niveles eléctricos lógicos a eventos de software y despachar comandos hacia la cola del sistema únicamente cuando se detectan flancos (cambios de estado).



---

### Evolución de Variables del Sensor

Desde la inicialización (`task_sensor_init`) y durante las sucesivas ejecuciones (`task_sensor_update`), la memoria de cada sensor evoluciona de la siguiente manera:

* **`index`**: Es el iterador del arreglo. En la inicialización y actualización, itera desde 0 hasta `SENSOR_DTA_QTY - 1` (en este caso, solo 0, ya que hay 1 solo sensor configurado).


* **`task_sensor_dta_list[index].tick`**: *Unidad de medida: Ticks del sistema (milisegundos).* Esta variable **no se inicializa explícitamente** en `task_sensor_init()` ni se utiliza durante la operación normal del diagrama de estados provisto. Únicamente toma el valor `DEL_BTN_MIN` (0) si la máquina de estados cae en el caso `default` (un estado de falla) como mecanismo de recuperación. (Nota: el código provisto carece de la lógica Anti-Rebote temporizada que justificaría el uso activo de esta variable).


* **`task_sensor_dta_list[index].state`**:
* En `task_sensor_init()` se inicializa forzosamente en `ST_BTN_IDLE`.


* En `task_sensor_update()`, alterna entre `ST_BTN_IDLE` (botón suelto) y `ST_BTN_ACTIVE` (botón presionado) dependiendo de los eventos generados.




* **`task_sensor_dta_list[index].event`**:
* En `task_sensor_init()` se inicializa en `EV_BTN_UP`.


* En cada `task_sensor_update()`, se sobrescribe inmediatamente evaluando si el pin físico coincide con el nivel lógico de activación (`pressed`). Tomará el valor `EV_BTN_DOWN` si hay coincidencia, o `EV_BTN_UP` en caso contrario.





---

### Comportamiento de `task_sensor_statechart(uint32_t index)`

Esta función es el núcleo de la FSM del sensor y ejecuta dos fases secuenciales:

1. **Traducción de Hardware a Evento:** Lee el estado eléctrico del pin GPIO mediante `HAL_GPIO_ReadPin()` y lo compara con el parámetro `pressed` (nivel activo) configurado para ese sensor específico. Asigna `EV_BTN_DOWN` o `EV_BTN_UP` a la variable de evento dinámico.


2. **Evaluación de Transiciones (Switch-Case):** Evalúa el estado actual (`p_task_sensor_dta->state`):


* **Si está en `ST_BTN_IDLE`:** Evalúa si el evento es `EV_BTN_DOWN`. De ser así, envía el evento de excitación (`signal_down`) a la cola del sistema mediante `put_event_task_system()` y avanza el estado a `ST_BTN_ACTIVE`.


* **Si está en `ST_BTN_ACTIVE`:** Evalúa si el evento es `EV_BTN_UP`. De ser así, envía el evento de relajación (`signal_up`) a la cola y regresa al estado `ST_BTN_IDLE`.


* **Caso `default`:** Maneja corrupciones de memoria reseteando las variables dinámicas del sensor al estado de reposo inicial (`ST_BTN_IDLE`, `EV_BTN_UP`).





---

### Evolución de Variables de la Cola FIFO (`event_task_system_queue`)

Estas variables gestionan la cola circular implementada en `task_system_interface.c`. Su evolución al dispararse las transiciones en el `task_sensor_statechart` es la siguiente:

* **`event_task_system_queue.count`**: Inicia en 0 (asumiendo que se llamó previamente a `init_event_task_system()`). Permanece en 0 durante el polling normal. Cuando el sensor detecta un flanco (ej. se presiona el botón), llama a `put_event_task_system()`, lo que **incrementa `count` en 1**.


* **`event_task_system_queue.queue[i]`**: El arreglo se inicializa con el valor `EMPTY` (255) en todos sus índices. Al ingresar un evento, el índice actual apuntado por `head` (ej. `queue[0]`) recibe el evento disparado por el sensor (ej. `EV_SYS_ACTIVE`).


* **`event_task_system_queue.head`**: Inicia en 0. Cada vez que el sensor empuja un evento válido, `head` se incrementa en 1. Si `head` alcanza el valor `QUEUE_LENGTH` (16), se produce un *wrap-around* y vuelve a 0 de forma circular.


* **`event_task_system_queue.tail`**: Representa el puntero de lectura (consumidor). Inicia en 0 y **no sufre modificaciones** por acción de `task_sensor_init()` ni de `task_sensor_update()`. Permanecerá estático hasta que la tarea `System` (en otro archivo) invoque `get_event_task_system()` para vaciar la cola.



¿Existe alguna ambigüedad respecto a cómo el puntero `tail` interactuará con el resto del sistema una vez que el consumidor comience a procesar estos eventos?