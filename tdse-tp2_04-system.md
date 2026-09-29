# TP2 - Actividad 04: 1 Actuator Statechart

## Modificaciones realizadas en código C
- **task_actuator_attribute.h**: Definición de los estados (`ST_LED_OFF`, `ST_LED_ON`, `ST_LED_BLINK`) y eventos (`EV_LED_OFF`, `EV_LED_ON`, `EV_LED_BLINK`).
- **task_actuator.c**: Mapeo del LED único `LD2` (`LED_A`).
- **task_actuator_update()**: Control de salidas físicas mediante llamadas a la HAL de STM32 (`HAL_GPIO_WritePin`, `HAL_GPIO_TogglePin`).
- **En este caso no se incluyeron LEDs externos por lo que se modificó el 3 y se utilizó solo el de la placa, sabiendo que varias de las modificaciones de los archivos del actuador ya estaban realizadas para el punto previo, solo se redujo a un LED lo propuesto**

## Resumen de Evolución del Actuador
| Estado Actual | Evento | Estado Siguiente | Operación GPIO (Hardware) |
| :--- | :--- | :--- | :--- |
| `ST_LED_OFF` | `EV_LED_ON` | `ST_LED_ON` | `HAL_GPIO_WritePin("Led Placa, LED_ON)` |
| `ST_LED_OFF` | `EV_LED_BLINK` | `ST_LED_BLINK` | `HAL_GPIO_WritePin(Led Placa, LED_ON)`, `tick = DEL_LED_MAX` |
| `ST_LED_ON` | `EV_LED_OFF` | `ST_LED_OFF` | `HAL_GPIO_WritePin(Led Placa, LED_OFF)` |
| `ST_LED_BLINK` | `tick == 0` | `ST_LED_BLINK` | `HAL_GPIO_TogglePin(Led Placa)`, `tick = DEL_LED_MAX` |
| `ST_LED_BLINK` | `EV_LED_OFF` | `ST_LED_OFF` | `HAL_GPIO_WritePin(Led Placa, LED_OFF)` |

## Conclusión
Se logró reducir el funcionamiento anterior a un solo LED, en esta caso el de la placa, mediante Live Expressions se observó la actualización de los estados del led y del tick.
