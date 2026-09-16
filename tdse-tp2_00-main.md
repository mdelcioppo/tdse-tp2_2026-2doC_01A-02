Como profesor, procederé a analizar la estructura técnica de los archivos solicitados y el flujo de ejecución cronológico del sistema, enfocándome estrictamente en el hardware y el comportamiento de los relojes.

### Análisis Funcional de los Archivos

1. **`startup_stm32f103rbtx.s` (Código de Arranque)**:
* Es el primer código en ejecutarse tras un reinicio del microcontrolador.


* Define la tabla de vectores de interrupción (`g_pfnVectors`), asignando direcciones de memoria a las rutinas de servicio (por ejemplo, `Reset_Handler` y `SysTick_Handler`).


* Su función `Reset_Handler` prepara el entorno de ejecución en C: copia las variables inicializadas a la RAM, pone a cero la sección de memoria no inicializada (`.bss`) y finalmente salta a la función `main()`.




2. **`main.c` (Lógica Principal)**:
* Contiene el punto de entrada de la aplicación en C (`main()`).


* Se encarga de inicializar la capa de abstracción de hardware (HAL), configurar el árbol de relojes (`SystemClock_Config`) e inicializar los periféricos (GPIO, USART2).


* Implementa el bucle infinito del programa (`while (1)`) donde se ejecuta la lógica continua de la aplicación (`app_update()`).




3. **`stm32f1xx_it.c` (Gestión de Interrupciones)**:
* Contiene las rutinas de servicio de interrupción (ISR) asociadas a la tabla de vectores.


* Captura eventos asíncronos del hardware, como el temporizador del sistema (`SysTick_Handler`) o interrupciones de pines GPIO (`EXTI15_10_IRQHandler`), derivándolos a las funciones de manejo de la librería HAL.





---

### Evolución de `SystemCoreClock` y `SysTick` (De Reset a while(1))

El flujo de evolución temporal de las variables que gobiernan el tiempo y la frecuencia del sistema es el siguiente:

**1. Fase de Arranque (`startup_stm32f103rbtx.s`)**

* El sistema arranca en `Reset_Handler`.


* Se ejecuta la instrucción `bl SystemInit`. Aunque el código de esta función no está en los adjuntos (generalmente reside en `system_stm32f1xx.c`), aquí es donde la variable global `SystemCoreClock` se inicializa con el reloj base por defecto del hardware (típicamente el oscilador interno HSI a 8 MHz).


* El hardware de SysTick está apagado.

**2. Fase de Inicialización HAL (`main.c`)**

* El código salta a `main()` e invoca `HAL_Init()`.


* Esta función inicializa el temporizador de hardware SysTick para que genere una interrupción cada 1 milisegundo. Lo hace utilizando la frecuencia base actual (los 8 MHz iniciales). En este momento, la cuenta de SysTick comienza a operar.



**3. Configuración del Árbol de Relojes (`SystemClock_Config` en `main.c`)**

* Se configura el oscilador interno (`RCC_OSCILLATORTYPE_HSI`) y se activa el bucle de bloqueo de fase (PLL) con un multiplicador de 16 (`RCC_PLL_MUL16`).


* Al aplicar esta configuración con `HAL_RCC_ClockConfig()`, la variable `SystemCoreClock` cambia drásticamente su valor para reflejar la nueva frecuencia de la CPU (típicamente pasando a 64 MHz, dependiendo del divisor previo).


* Como la frecuencia del procesador ha cambiado, `HAL_RCC_ClockConfig()` recalibra automáticamente el temporizador SysTick subyacente. Esto asegura que la interrupción siga ocurriendo exactamente cada 1 milisegundo a pesar de que el procesador ahora funciona más rápido.



**4. Estado Estacionario e Interrupciones (Bucle `while(1)`)**

* La ejecución alcanza el bucle infinito `while (1)`. La variable `SystemCoreClock` se estabiliza en su valor final y no volverá a cambiar a menos que se reconfigure el hardware.


* En segundo plano, cada 1 ms, el hardware dispara una interrupción que pausa el `while(1)` y salta a `SysTick_Handler()` en `stm32f1xx_it.c`.


* Dentro del manejador, se ejecuta `HAL_IncTick()`. Esta función incrementa una variable global en software (usualmente `uwTick`) que almacena los milisegundos transcurridos desde el inicio del sistema.


* Este ciclo de interrupción e incremento ocurre continuamente mientras la aplicación principal corre.