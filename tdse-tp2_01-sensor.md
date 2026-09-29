# TP2 - Actividad 01: 1 Sensor Statechart

## Modificaciones realizadas en código C
- **task_sensor_attribute.h**: Se definen las enumeraciones de estados (`ST_BTN_UP`, `ST_BTN_FALLING`, `ST_BTN_DOWN`, `ST_BTN_RISING`) y eventos (`EV_BTN_UP`, `EV_BTN_DOWN`).
- **task_sensor.c**: Se configura el arreglo `task_sensor_cfg_list[1]` asociando la entrada GPIO a `B1_USER` (Blue Button).
- **task_sensor_update()**: Se codifica el diagrama de estados utilizando un `switch(state)` con filtrado antirrebote (`DEL_BTN_MAX`). Al confirmar pulsación/liberación se llama a `put_event_task_system()`. Todo basado en el statechart proporcionado por la cátedra.

## Resumen de Estados y Transiciones
| Estado Actual | Evento / Condición | Estado Siguiente | Acción / Función llamada |
| :--- | :--- | :--- | :--- |
| `ST_BTN_UP` | `EV_BTN_DOWN` | `ST_BTN_FALLING` | `tick = DEL_BTN_MAX` |
| `ST_BTN_FALLING` | `tick == 0` | `ST_BTN_DOWN` | `put_event_task_system(EV_SYS_BTN_DOWN)` |
| `ST_BTN_DOWN` | `EV_BTN_UP` | `ST_BTN_RISING` | `tick = DEL_BTN_MAX` |
| `ST_BTN_RISING` | `tick == 0` | `ST_BTN_UP` | `put_event_task_system(EV_SYS_BTN_UP)` |

## Conclusión
Se logró verificar al observar task_sensor_dta_list que al pulsar el botón se modificaban los estados así como también la actualización del contador con el anti-rebote incluido.
