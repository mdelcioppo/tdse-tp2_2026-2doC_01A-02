# TP2 - Actividad 03: System Statechart

## Modificaciones realizadas en código C
- **task_system_attribute.h**: Definición de la enumeración de estados del sistema (`ST_SYS_WAIT_FOR_CAR_ARRIEVE`, etc.).
- **task_system.c**: Implementación de la máquina de estados de control principal.
  - Procesamiento de la cola de eventos `event_task_system_queue`.
  - Envío de comandos a los actuadores mediante `put_event_task_actuator()`.
  - Además se modificaron los archivos propios del actuador para poder verificar el funcionamiento completo del sistema con los leds incluidos.
  - Actualización del switch con los nuevos eventos y estados.
## Resumen de Estados y Eventos del Sistema
| Estado Actual | Evento Recibido | Estado Siguiente | Eventos Generados (Hacia Actuadores) |
| :--- | :--- | :--- | :--- |
| `ST_SYS_WAIT_FOR_CAR_ARRIEVE` | `EV_SYS_CAMERA` | `ST_SYS_WAIT_FOR_BUTTON_PRESSED` | Ninguno |
| `ST_SYS_WAIT_FOR_BUTTON_PRESSED` | `EV_SYS_BUTTON` | `ST_SYS_WAIT_FOR_BARRIER_OPENED` | `EV_LED_BLINK` (Barrier Open)<br>`EV_LED_OFF` (Barrier Close) |
| `ST_SYS_WAIT_FOR_BARRIER_OPENED` | `tick == 0` | `ST_SYS_WAIT_FOR_CAR_LEAVES` | `EV_LED_ON` (Barrier Open) |
| `ST_SYS_WAIT_FOR_CAR_LEAVES` | `EV_SYS_SENSOR_COIL` | `ST_SYS_WAIT_FOR_BARRIER_CLOSED` | `EV_LED_OFF` (Barrier Open)<br>`EV_LED_BLINK` (Barrier Close) |
| `ST_SYS_WAIT_FOR_BARRIER_CLOSED` | `tick == 0` | `ST_SYS_WAIT_FOR_CAR_ARRIEVE` | `EV_LED_ON` (Barrier Close) |

## Conclusión
Se logro implementar el sistema completo con los cambios en el actuador para dos leds (uno externo y otro de la placa) para simular, llegada, entrada e salida del auto, es decir todos los pasos del sistema completo, incluido la bajada y subida de barrera con blinking de los leds.
