# TP2 - Actividad 05: 2 Actuator Statecharts

## Modificaciones realizadas en código C
- **task_actuator_attribute.h**: Declaración de identificadores `ID_LED_BARRIER_OPEN` e `ID_LED_BARRIER_CLOSE`.
- **task_actuator.c**:
  - Ajuste de estructuras a 2 instancias: `task_actuator_cfg_list[2]` y `task_actuator_dta_list[2]`.
  - Asignación de pines GPIO individuales configurados en CubeMX para cada LED.
  - Modificación de `task_actuator_update()` para procesar de forma concurrente ambos actuadores mediante un bucle de iteración.

## Conclusión
Se implementó el mismo funcionamiento que en la parte 3 (system completo), pero en este caso se utilizaron dos leds externos, para los cuáles en live expressions se siguió la actualización de estados al hacer funcionar el sistema mediante apretar los botones, y se verificó el abrir y cerrar de barrera implementado cada uno con un led distinto.
