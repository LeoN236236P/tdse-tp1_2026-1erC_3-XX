### Análisis del Código Fuente: Tareas y Comunicación

Estos cuatro archivos conforman la capa lógica de entrada de tu sistema embebido. Implementan una **Máquina de Estados Finitos (FSM)** para leer un botón (Sensor) y una **Cola Circular (FIFO)** para enviarle mensajes a otra parte del programa (el Sistema), logrando un diseño desacoplado y no bloqueante.

#### 1. Función de los Archivos
* **`task_sensor_attribute.h` y `task_system_attribute.h`:** Son los diccionarios del sistema. Definen los tipos de datos enumerados (`enum`) para los estados (ej. `ST_BTN_IDLE`) y los eventos (ej. `EV_BTN_DOWN`) de cada tarea, además de las estructuras donde se guardará la configuración estática y las variables dinámicas (`_dta_t`).
* **`task_sensor.c`:** Contiene la lógica ejecutada cíclicamente para leer el hardware (el botón).
* **`task_system_interface.c`:** Implementa un mecanismo de comunicación entre tareas (*Inter-Task Communication*). Es un "buzón" donde el sensor deposita eventos que luego el sistema leerá a su propio ritmo.

---

### Evolución de las Variables de la Tarea Sensor (`task_sensor.c`)

Durante la ejecución de `task_sensor_init()` y el bucle continuo de `task_sensor_update()`, las variables evolucionan de la siguiente manera:

* **`index`** (Unidad: **Adimensional**).
    * Es el iterador del array de sensores. Como la constante `SENSOR_DTA_QTY` es 1 (solo está configurado `BTN_A`), `index` siempre vale **0** dentro del bucle.
* **`task_sensor_dta_list[index].tick`** (Unidad: **Milisegundos / Ticks del sistema**).
    * *Nota importante sobre tu código:* Aunque la variable está definida en la estructura y existen macros de tiempo (`DEL_BTN_MAX`), en la implementación actual de la FSM **esta variable no se está utilizando**. Su valor inicial es basura (o 0 si se limpió la RAM) y solo se le asigna el valor `DEL_BTN_MIN` (0) si la máquina de estados cae en el caso de error (`default`). El código actual *no tiene implementado un mecanismo de anti-rebote (debouncing)* por tiempo.
* **`task_sensor_dta_list[index].state`** (Estado actual).
    * *En `init`:* Arranca forzosamente en `ST_BTN_IDLE` (reposo).
    * *En `update`:* Alternará a `ST_BTN_ACTIVE` cuando se presione el botón y volverá a `ST_BTN_IDLE` cuando se suelte.
* **`task_sensor_dta_list[index].event`** (Evento detectado).
    * *En `init`:* Se inicializa en `EV_BTN_UP`.
    * *En `update`:* En cada milisegundo, esta variable es **sobrescrita** por la lectura física del pin (`HAL_GPIO_ReadPin`). Tomará el valor `EV_BTN_DOWN` si el usuario está presionando el botón físicamente en ese instante, o `EV_BTN_UP` si no lo está tocando.

---

### Comportamiento de `task_sensor_statechart(uint32_t index)`

Esta función es el núcleo de la lógica del botón. Trabaja en dos etapas en cada ejecución (cada 1 ms):

1.  **Traducción de Hardware a Lógica:** Lee el estado físico del pin configurado en `task_sensor_cfg_list`. Compara si el voltaje leído coincide con el nivel lógico de "presionado" y actualiza la variable `event` (`EV_BTN_DOWN` o `EV_BTN_UP`).
2.  **Evaluación de la Máquina de Estados (`switch`):**
    * Si está en **`ST_BTN_IDLE`** y el evento es `EV_BTN_DOWN` (recién presionado): Llama a la interfaz del sistema para guardar un mensaje (`put_event_task_system`) indicando que el sistema debe activarse (`EV_SYS_ACTIVE`). Luego, cambia su estado a `ST_BTN_ACTIVE` para no volver a enviar el mensaje en el siguiente milisegundo.
    * Si está en **`ST_BTN_ACTIVE`** y el evento es `EV_BTN_UP` (recién soltado): Guarda un mensaje en la interfaz ordenando al sistema apagarse (`EV_SYS_IDLE`) y vuelve al estado `ST_BTN_IDLE`.

---

### Evolución de las Variables de la Cola del Sistema (`task_system_interface.c`)

Esta estructura de datos es una **Cola Circular (FIFO: First-In, First-Out)** de 16 posiciones. Así evolucionan sus variables cuando el Sensor (produce) y el Sistema (consume) interactúan:

* **`event_task_system_queue.queue[i]`** (El búfer de datos).
    * Al inicializarse (`init_event_task_system`), las 16 posiciones del array se llenan con el valor `EMPTY` (255).
    * A medida que el sensor detecta pulsaciones y llama a `put_event_task_system()`, las posiciones vacías se van reemplazando secuencialmente con los eventos `EV_SYS_ACTIVE` o `EV_SYS_IDLE`.
    * Cuando el sistema lee un evento mediante `get_event_task_system()`, extrae el valor y vuelve a escribir `EMPTY` en esa posición.
* **`event_task_system_queue.head`** (El puntero de escritura).
    * Arranca en **0**.
    * Cada vez que la tarea Sensor inserta un evento, la variable guarda el evento en la posición `head` y luego **incrementa en +1**.
    * Si `head` llega a 16 (`QUEUE_LENGTH`), vuelve a **0** automáticamente (comportamiento circular).
* **`event_task_system_queue.tail`** (El puntero de lectura).
    * Arranca en **0**.
    * Cada vez que la tarea Sistema extrae un evento, lo lee de la posición `tail` y luego **incrementa en +1**.
    * Al igual que la cabeza, si llega a 16, da la vuelta a **0**.
* **`event_task_system_queue.count`** (Cantidad de eventos pendientes).
    * Arranca en **0**.
    * Suma +1 con cada `put` (el usuario presionó o soltó el botón).
    * Resta -1 con cada `get` (el sistema procesó la acción).
    * Permite saber cuántos mensajes se han acumulado si el sistema está procesando tareas más lento de lo que el usuario presiona el botón.
