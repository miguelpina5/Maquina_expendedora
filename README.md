# Objetivo

Diseñar un controlador para una máquina expendedora con arduino UNO que consiga gestionar la interación entre usuario y sensores.

# Funcionamiento

En este apartado, explicaré la ejecución de la máquina expendedora. Tanto los componentes y la actividad hardware, como el funcionamiento software.

## Hardware
El sistema integra varios sensores:

### LCD

Para imprimir información necesaria y que sea un comportamiento más visual.

Usos:
- Imprimir la distancia, temperatura ...
- Ver el menú de los cafés.
- Indicar el preparado del café y cuando termina.

### Joystick

Nos permite navegar libremente y hacer diferentres funciones.

Usos:
- Navegar por el menú de cafés moviendo para arriba y para abajo.
- Preparar un café pulsando el botón del joystick sobre el producto que queramos.
- En el menú admin:
   - Navegar y entrar en al apartado que queramos (Ver temperatura, Modificar precios...) pulsando el joystick.
   - Si estamos dentro de algún apartado, volvemos al menú moviéndolo a la izquiera.
   - En el menú `Modificar precios`, pulsando el joystick en un producto, podemos modificar el precio de 5 en 5 céntimos, moviendo el joystick para arriba y para abajo, para subir y bajar el precio respectivamente. Para cancelar el cambio, movemos el joystick a la izquierda, y para confirmarlo, pulsamos el botón del joystick.

### Botón

Nos permite realizar funciones dependiendo del tiempo que lo mantenemos pulsado.

Usos:
- Modo cliente:
   - Si el tiempo entre que se pulsa y se suelta el botón es entre 2 y 3 segundos, Se reinicia el estado y vuelve a esperar cliente.
   - Si se pulsa más de 5 segundos entra en modo admin
- Modo admin:
   - Si se pulsa más de 5 segundos, sale del modo admin y espera un cliente.

### Sensor Temperatura/Humedad DHT11

Nos permite obtener e imprimir la temperatura y humedad.

Usos:
- Ver la temperatura en el menú admin y durante 5 segundos al detectar cliente.

### Sensor Ultrasonidos

Nos permite obtener e imprimir la distancia.

Usos:
- Cuando está esperando al cliente, mira constantemente la distancia al objeto más cercano, hasta encontrar alguno a menos de 1m.

### LEDs

Nos permite apreciar de una forma más visual diferentes situaciones.

Usos:
- Parpadea 3 veces cuando está cargando en el estado inicial.
- Se enciende de forma progresiva cuando se está preparando el café.
- Se encienden los dos cuando estás en modo admin.

## Software
### Watchdogs
Es un mecanismo de seguridad que se usa para evitar el bloqueo de la placa. Si no responde al comportamiento deseado o se bloquea, provoca un reset.

Usos:
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
Es un mecanismo del controlador para responder a eventos, suspende temporalmente el flujo del programa para ejecutar un ISR. Es un opción perfecta para el uso de los botones, porque permite el programa atender a las pulsasiones sin descuidar la ejecución del programa principal.

El código de la ISR debe ser lo más rápida posible, lo ideal sería modidficar variables, activar un flag, etc. Hacer Serial es un comprotamiento lento.

Usos:
- Detectar cuando de pulsa y se suelta el botón
```cpp
attachInterrupt(digitalPinToInterrupt(BUTTON), press_button, CHANGE);

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












# Vídeo de demostración

https://github.com/user-attachments/assets/5e29b49d-456f-45ec-8ca6-4c9a614feddb


# Esquema del circuito

<img width="1109" height="718" alt="Screenshot from 2025-11-19 17-44-21" src="https://github.com/user-attachments/assets/58fbc2c3-37fc-45a1-b7ed-e4143c9ab390" />

