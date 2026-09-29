# TP2 - Actividad 02: 3 Sensor Statecharts

## Modificaciones realizadas en código C
- **task_sensor_attribute.h**: Se actualizan las constantes para soportar un identificador para cada sensor (`ID_SENSOR_0`, `ID_SENSOR_1`, `ID_SENSOR_2`).
- **task_sensor.c**: 
  - Se redimensionan los arreglos `task_sensor_cfg_list[3]` y `task_sensor_dta_list[3]`.
  - Se mapean los puertos/pines GPIO asignados a las entradas externas en `.ioc` determinando si los botones tiene configuración pull up, pull down, etc, se verifica en la asignación mediante la datasheet evitar pines confictivos.
  - Se modifica `task_sensor_init()` y `task_sensor_update()` para iterar sobre los 3 índices mediante un bucle `for`.

## Matriz de Estados por Instancia
| Sensor Index | GPIO Pin | Estado Inicial | Evento Enviado al Sistema |
| :---: | :---: | :---: | :---: |
| 0 | B1_USER (PC13) | `ST_BTN_UP` | `EV_SYS_CAMERA` |
| 1 | BTN_B (PB10)| `ST_BTN_UP` | `EV_SYS_CAMERA` |
| 2 | BTN_C (PA10)| `ST_BTN_UP` | `EV_SYS_BUTTON` |
| 3 | BTN_D (PB5) | `ST_BTN_UP` | `EV_SYS_SENSOR_COIL` |

## Conclusión
Se logró simular el statechart solicitado para 3 sensores de manera correcta y se configuró para que sea utilizable también con 3 botones externos, siendo el B reemplazo del btn local de la placa.
