Este análisis detalla el flujo de ejecución de un firmware para el microcontrolador **STM32F103RBTx** (Cortex-M3), desde el arranque en ensamblador hasta el bucle principal en C.

---

## 1. Análisis de los Archivos

### **startup_stm32f103rbtx.s (El punto de partida)**
Es el archivo de ensamblador que se ejecuta inmediatamente después de un reset. Sus funciones principales son:
* **Definir la Tabla de Vectores (`g_pfnVectors`):** Establece la dirección inicial del puntero de pila (`_estack`) y las direcciones de los manejadores de interrupciones, empezando por `Reset_Handler`.
* **`Reset_Handler`:** 1.  Llama a `SystemInit` para una configuración inicial de registros.
    2.  **Copia la sección `.data`**: Mueve las variables globales inicializadas desde la Flash a la RAM.
    3.  **Limpia la sección `.bss`**: Pone a cero las variables globales no inicializadas en la RAM.
    4.  Llama a los constructores estáticos (`__libc_init_array`) y finalmente salta a la función `main()`.

### **main.c (Lógica de la aplicación)**
Contiene la estructura principal del programa:
* **Inicialización:** Configura la comunicación semihosting (si está activa), el HAL (Capa de Abstracción de Hardware), el reloj del sistema (`SystemClock_Config`), y los periféricos (UART2 y GPIO).
* **Estructura App:** Utiliza un diseño modular llamando a `app_init()` antes del bucle y `app_update()` dentro del `while(1)`.
* **Configuración de Reloj:** Establece el uso del oscilador interno (HSI) con el PLL para alcanzar la frecuencia de trabajo deseada.

### **stm32f1xx_it.c (Manejador de Interrupciones)**
Contiene las funciones que se ejecutan cuando ocurre un evento externo o interno:
* **`SysTick_Handler`:** Se ejecuta periódicamente (usualmente cada 1ms) para incrementar el contador de tiempo del sistema mediante `HAL_IncTick()`.
* **`EXTI15_10_IRQHandler`:** Maneja la interrupción del botón de usuario (`B1_Pin`), llamando al gestor de GPIO del HAL.

---

## 2. Evolución de Variables Críticas

A continuación se describe cómo cambian `SystemCoreClock` y el estado del `SysTick` durante el arranque.


### **Fase 1: Reset_Handler (startup)**
* **`SystemCoreClock`**: Por defecto, tras el reset, el STM32F1 arranca con el oscilador interno **8 MHz** (HSI).
* **`SysTick`**: El temporizador está **desactivado**. Su registro de valor actual es indeterminado o cero, y no genera interrupciones.

### **Fase 2: SystemClock_Config (main.c)**
En esta función, el código realiza lo siguiente:
1.  Configura el PLL multiplicando el HSI (8 MHz / 2 = 4 MHz) por 16.
2.  **`SystemCoreClock`**: Al final de esta función, la variable se actualiza para reflejar la nueva frecuencia de **64 MHz** ($4 \text{ MHz} \times 16 = 64 \text{ MHz}$).
    * *Nota:* Según el código: `PLLSource = HSI_DIV2` (4MHz) y `PLLMUL = 16`.

### **Fase 3: HAL_Init / MX_..._Init**
* **`SysTick`**: Dentro de `HAL_Init()`, se configura el temporizador SysTick para generar una interrupción cada **1 milisegundo**.
    * Se carga el registro de recarga (`LOAD`) basado en el valor de `SystemCoreClock`.
    * Se activa la interrupción y el contador empieza a funcionar.
* **Variable de Ticks**: La variable interna `uwTick` (gestionada en `stm32f1xx_it.c`) comienza a incrementar de 0 en adelante cada vez que el contador llega a cero.

### **Fase 4: Loop Principal (while (1))**
Al llegar aquí:
* **`SystemCoreClock`**: Se mantiene estable en **64,000,000** (64 MHz).
* **`SysTick`**: Está en un ciclo continuo:
    1.  El hardware cuenta hacia atrás.
    2.  Al llegar a 0, se dispara `SysTick_Handler`.
    3.  `HAL_IncTick()` incrementa el contador de milisegundos.
    4.  El proceso se repite indefinidamente, permitiendo que funciones como `HAL_Delay()` funcionen correctamente dentro de `app_update()`.

| Momento | `SystemCoreClock` | Estado SysTick |
| :--- | :--- | :--- |
| **Reset** | 8 MHz (Valor inicial) | Desactivado |
| **Después de `SystemClock_Config`** | 64 MHz | Configurado pero esperando activación |
| **Después de `HAL_Init`** | 64 MHz | **Activo** (Incrementando cada 1ms) |
| **En `while(1)`** | 64 MHz | Activo y estable |
