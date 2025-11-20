# Maquina expendedora

Este proyecto implementa un controlador para una máquina expendedora utilizando Arduino UNO, incluyendo el manejo de sensores, interacción con el usuario y diferentes modos de funcionamiento mediante una máquina de estados, interrupciones y tareas periódicas con ArduinoThread.

# Funcionamiento

En este apartado, explicaré la ejecución de la máquina expendedora. Tanto los componentes y la actividad hardware, como el funcionamiento software.

## Hardware
El sistema integra varios sensores:

### LCD

Se usa para imprimir información necesaria del sistema:

- Mensajes de arranque
- Valores de sensores como distancia, temperatura y humedad.
- Menú de los productos.
- Progreso de preparación del café.

### Joystick

Permite navegar libremente y selección de funciones:

- Arriba/abajo: navegar por listas.
- Pulsación: seleccionar la opción que se encuentre en el LCD.
- Izquiera: volver atrás en el menú admin.
- Arriba/abajo en `Modificar precios`: ajustar el precio de 5 en 5 centimos.

### Botón

Permite realizar funciones dependiendo del tiempo de pulsación:

- 2-3 segundos: reinicia el modo cliente solo si estas en él.
- +5 segundos: en estado cliente, entra al modo admin, y en modo admin, sale.

### Sensor Temperatura/Humedad DHT11

Permite obtener los valores de la temperatura y humedad:

- Se muestra automáticamente cuando llega un cliente (durante 5 s).
- Se muestra en la sección correspondiente del modo admin.

### Sensor Ultrasonidos

Permite obtener la distancia:

- Espera la presencia de un cliente a menos de 1m.

### LEDs

Permite apreciar de una forma más visual diferentes situaciones.

- LED verde: parpadea 3 veces cuando está cargando en el estado inicial.
- LED rojo: se enciende de forma progresiva cuando se está preparando el café.
- Ambos LEDs: se encienden cuando estás en modo admin.

## Software

El funcionamiento principal está basado en una máquina de estados, donde cada uno representa una funcionalidad en la aplicación. Por ejemplo `PREPARE_COFFEE` que se encarga de realizar todo lo necesario para simular la preparación del café, o `ADMIN`, donde se encuentra el funcionamiento del admin. Para este estado se implementó una máquina de estados interna, puesto que tenía demasiadas funciones y así me permitía estar dentro del bucle principal en el estado `ADMIN` y dentro realizar la función que se pidiese.

### Estados principales

- `START`
- `WAIT_CLIENT`
- `WEATHER_5SEC`
- `CHOOSE_COFFEE`
- `PREPARE_COFFEE`
- `REMOVE_COFFEE`
- `ADMIN`

### Estados admin

- `ADMIN_WEATHER`
- `ADMIN_DISTANCE`
- `ADMIN_COUNTER`
- `ADMIN_MENU_PRICES`
- `ADMIN_CHANGE_PRICES`
- `ADMIN_MENU`


### Watchdogs
Es un mecanismo de seguridad que se usa para evitar el bloqueo de la placa. Si no responde al comportamiento deseado o se bloquea, provoca un reset de la placa:

- Desactiva el watchdog y lo configura
```cpp
void setup() {
  wdt_disable();
  wdt_enable(WDTO_8S);
}
```
- Actualiza el watchdog para que no produzca el reseteo
```cpp
void loop() {
  wdt_reset();
}
```
### Interrupciones
Es un mecanismo del controlador para responder a eventos, suspende temporalmente el flujo del programa para ejecutar una ISR. Es un opción perfecta para el uso de los botones, porque permite el programa atender a las pulsasiones sin descuidar la ejecución del programa principal.

El código de la ISR debe ser lo más rápida posible, lo ideal sería modificar variables, activar un flag, etc. Hacer `Serial.println(...)` no es recomendable porque es un comprotamiento lento.

En este caso las interrupciones se usan para la pulsación de botones, hay que tener en cuenta el rebote de estos, para eso, dentro de la ISR se comprueba si el tiempo entre las dos interrupciones es mayor a `DEBOUNCE` (200ms):

- Detecta cuando se pulsa y se suelta el botón, se configura en modo `CHANGE` para que ejecute la ISR cuando cambia de estado, de LOW a HIGH o viceversa. Esto se necesita porque para reiniciar el estado cliente el tiempo desde que se pulsa hasta que se deja de pulsar es entre 2-3 segundos, entonces hay que saber cuando ocurren las dos situaciones.
```cpp
void setup() {
  attachInterrupt(digitalPinToInterrupt(BUTTON), press_button, CHANGE);
}

void press_button() {
  unsigned long interrupt_time = millis();

  if (interrupt_time - last_interrupt_time_button > DEBOUNCE) {
    if (button_pressed == false) {
      start_time_button = millis();
      button_pressed = true;
      button_rissing = false;

    } else {
      last_time_button = millis();
      button_pressed = false;
      button_rissing = true;
    }
  }
  last_interrupt_time_button = interrupt_time;
}
```

- Detectar cuando de pulsa el botón del joystick, se configura en modo `FALLING` porque solo nos interesa cuando se pulsa.
```cpp
void setup() {
  attachInterrupt(digitalPinToInterrupt(SW), press_joystick, FALLING);
}

void press_joystick() {
  unsigned long interrupt_time = millis();

  if (interrupt_time - last_interrupt_time_joystick > DEBOUNCE) {
    if (state == CHOOSE_COFFEE)
      joystick_pressed = true;
    
    if (state == ADMIN) {
      if (admin_state == ADMIN_MENU || admin_state == ADMIN_MENU_PRICES || admin_state == ADMIN_CHANGE_PRICES)
        joystick_pressed = true;
    }
  }

  last_interrupt_time_joystick = interrupt_time;
}
```

### Threads
Los threads en arduino no ejecutan en paralelo, pero pero permiten ejecutar tareas periodicas, esto facilita la realización de funciones que dependan de tiempo.

Existe un `Controller` que permite manejar, activar y ejecutar numerosos threads al mismo tiempo. El controlador y los threads se crean de la siguiente manera:
```cpp 
ThreadController controller = ThreadController();
Thread thread_start = Thread();
Thread thread_weather = Thread();
Thread thread_remove_coffee = Thread();
```

Usos:
- Para hacer parpadear el LED cada segundo. `SEC_TO_MS` = 1000ms
```cpp
void setup() {
  thread_start.enabled = true;
  thread_start.setInterval(SEC_TO_MS);
  thread_start.onRun(callback_thread_start);
  controller.add(&thread_start);
}

void callback_thread_start() {

  digitalWrite(LED_GREEN, !digitalRead(LED_GREEN));
  waited_seconds++;
  
  if (waited_seconds >= 6) {
    thread_start.enabled = false;
    change_state(WAIT_CLIENT);
  }
}
```
- Para enseñar la temperatura y humedad, espera 5 segundos y cambia de estado.
```cpp
void setup() {
  thread_weather.enabled = false;
  thread_weather.setInterval(5*SEC_TO_MS);
  thread_weather.onRun(callback_thread_weather);
  controller.add(&thread_weather);
}

void callback_thread_weather() {

  if (waited_seconds == 5)
    change_state(CHOOSE_COFFEE);

  else
    waited_seconds = 5;
}
```
- Para la retirada del café, espera 3 segundos y cambia de estado.
```cpp
void setup() {
  thread_remove_coffee.enabled = false;
  thread_remove_coffee.setInterval(3*SEC_TO_MS);
  thread_remove_coffee.onRun(callback_thread_remove_coffee);
  controller.add(&thread_remove_coffee);
}

void callback_thread_remove_coffee() {

  if (waited_seconds == 3)
    change_state(WAIT_CLIENT);

  else 
    waited_seconds = 3;
}
```

# Esquema del circuito

<img width="1109" height="718" alt="Screenshot from 2025-11-19 17-44-21" src="https://github.com/user-attachments/assets/58fbc2c3-37fc-45a1-b7ed-e4143c9ab390" />

# Vídeo de demostración

https://github.com/user-attachments/assets/5e29b49d-456f-45ec-8ca6-4c9a614feddb


