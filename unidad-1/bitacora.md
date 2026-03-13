# Bitacora de la Unidad 1



## Actividad 01


### 1) Creación del programa microbit
 es una forma de generar experiencias combinando el diseño y las nuevas tecnologias generativas.

### 2) Lectua del boton
 es una rama que puedo utilizar para generar creativamente ideas o diseñosutiles para pulir una estetica en mi portafolio.
   (probando insertar imagenes)
<img width="1517" height="609" alt="image" src="https://github.com/user-attachments/assets/db13af69-9bf6-473b-ab7d-d65640f4b50c" />

## Actividad 02
1) es una sistema creado para generar diseños sin fin que se pueden modificar al gusto del usuario dependiendo de su necesidad.
2) la creacion de patrones para modelos 3D que ayuden a definir nuevas esteticas ej: un sistema que genere patrones que simulen manchas de nieve para modelos ambientados en lugares nevados.
<img width="1154" height="383" alt="image" src="https://github.com/user-attachments/assets/fb9948f6-fde4-450f-81fd-99bd782365a9" />

<img width="1179" height="769" alt="image" src="https://github.com/user-attachments/assets/134ad8eb-23b9-4ad4-8514-e22a78632f88" />

## Actividad 03
-Hecha en clase principios del uso del microbit

## Actividad 04
-Vamos a crear juntos un programa en p5.js que interactúe con un micro:bit. La idea es que el programa en p5.js muestre un cuadrado en la pantalla que cambie de color al presionar un botón del micro:bit y regrese a su color original al soltar el botón.

## 1) Creación del programa del micro:bit

Vamos a crear un programa en el micro:bit que tenga un input (un botón) y un output (enviar un mensaje por serial) que indique que se ha presionado el botón.

Lo primero que debes hacer es abrir el editor de micro:bit. Luego, importa de la biblioteca microbit todas las funciones que necesitas para interactuar con el micro:bit.
```.asm
 from microbit import *

```
## 2)Lectura del boton
Ahora vamos a leer el estado del botón A del micro:bit. Para ello, utilizaremos un bucle que se ejecutará continuamente y verificará si el botón A ha sido presionado.

Observa que utilizamos button_a.was_pressed() para detectar si el botón ha sido presionado. También podrías usar button_a.is_pressed() si quieres saber si el botón está presionado en ese momento, pero was_pressed() es más adecuado para detectar eventos únicos como un clic. Si usas is_pressed(), el programa podría enviar múltiples mensajes si el botón se mantiene presionado. ¿Ves la diferencia?
```.asm
  from microbit import *

  while True:
      if button_a.is_pressed():
// en el original aparece como was pero no funciona por eso lo corregi :)
```
## 3) Lectura del botón y envío del mensaje

 Ahora que sabemos cuándo se presiona el botón, vamos a enviar un mensaje por el puerto serial del micro:bit. Esto nos permitirá recibir el mensaje en p5.js.

Nota que debes inicializar la comunicación serial con uart.init(baudrate=115200) antes de enviar mensajes. El baudrate es la velocidad de transmisión de datos, y 115200 es una velocidad comúnmente utilizada para la comunicación serial. Finalmente, utilizamos uart.write('A') para enviar el mensaje ‘A’ cuando se presiona el botón A.
```.asm
  from microbit import *

  uart.init(baudrate=115200)

  while True:
      if button_a.was_pressed():
          uart.write('A')

```
## 4) Aplicación en p5.js - Biblioteca de conexión serial
Lo primero que debes hacer es añadir la biblioteca que te permite conectar el micro:bit con p5.js. Recuerda que esto lo haces en el archivo index.html de tu proyecto p5.js.
```.asm
<script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script>

```
## 5) Aplicación en p5.js - Configuración inicial
Vas a crear un par de variables globales (estas las creas por fuera de cualquier función). Al ser globales, podrás acceder a ellas desde cualquier parte del código. Estas variables te servirán para almacenar una referencia al objeto que te permitirá manipular el puerto serial y la otra variable te servirá para guardar una referencia al objeto que representa el botón con el cual podrías conectar y desconectar el micro:bit de la aplicación.

Observa la función connectBtnClick(). Esta función se ejecuta cuando el usuario hace click en el botón de conexión. A esto se le llama un “event handler” o manejador de eventos.
```.asm
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
}

function draw() {
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}

```
## 6) Aplicación en p5.js - Dibujar en cada frame
En draw() vas a dibujar un cuadrado en la pantalla todo el tiempo, pero tendrás que decidir en cada frame qué color debe tener. Entonces, antes de dibujarlo vas a leer el puerto serial y ver si hay algún mensaje disponible. Si hay un mensaje, lo vas a leer (debes siempre consumir el mensaje para que no se acumule) y cambiarás el color a rojo. Si no hay mensaje, el cuadrado será de color verde.

Antes de terminar el frame vas a actualizar el estado del botón para que refleje el estado de conexión del micro:bit.


```.asm

    let port;
    let connectBtn;

    function setup() {
        createCanvas(400, 400);
        background(220);
        port = createSerial();
        connectBtn = createButton("Connect to micro:bit");
        connectBtn.position(80, 300);
        connectBtn.mousePressed(connectBtnClick);
    }

    function draw() {
        background(220);

        if (port.availableBytes() > 0) {
            let dataRx = port.read(1);
            if (dataRx == "A") {
            fill("red");
            }
        } else {
            fill("green");
        }

        rectMode(CENTER);
        rect(width / 2, height / 2, 50, 50);

        if (!port.opened()) {
            connectBtn.html("Connect to micro:bit");
        } else {
            connectBtn.html("Disconnect");
        }
    }

    function connectBtnClick() {
        if (!port.opened()) {
            port.open("MicroPython", 115200);
        } else {
            port.close();
        }
    }
```
```.asm


```

# Bitácora de aplicación
## Actividad 05
El programa de p5.js.
``` .asm
// Declara la variable que manejará la comunicación serial
let port;

// Botón para conectar o desconectar el micro:bit
let connectBtn;

// Variable que controla la posición horizontal del círculo
let x = 350;

// Variable donde se guardan los datos recibidos por serial
let dataRx;

// Función que se ejecuta UNA SOLA VEZ al iniciar el programa
function setup() {

  // Crea un lienzo de 400x400 píxeles
  createCanvas(400, 400);

  // Pinta el fondo de gris claro
  background(220);

  // Crea el objeto de comunicación serial (WebSerial)
  port = createSerial();

  // Crea un botón HTML con el texto indicado
  connectBtn = createButton("Connect to micro:bit");

  // Posiciona el botón en la pantalla
  connectBtn.position(80, 300);

  // Asocia el click del botón a la función connectBtnClick
  connectBtn.mousePressed(connectBtnClick);

  // Define el color de relleno como rojo
  fill("red");

  // Dibuja un círculo inicial en el centro vertical
  ellipse(x / 2, height / 2, 100, 100);
}

// Función que se ejecuta cada vez que se presiona el botón
function connectBtnClick() {

  // Si el puerto NO está abierto
  if (!port.opened()) {

    // Abre la conexión serial a 115200 baudios
    port.open('MicroPython', 115200);

  } else {

    // Si ya estaba abierto, cierra la conexión
    port.close();
  }
}

// Función que se ejecuta continuamente (60 veces por segundo)
function draw() {

  // Limpia la pantalla en cada frame
  background(220);

  // Dibuja el círculo usando la posición actual de x
  ellipse(x / 2, height / 2, 100, 100);

  // Define el color del texto
  fill('red');

  // Muestra en pantalla el dato recibido por serial
  text(dataRx, x / 2, height / 2);

  // Verifica si hay datos disponibles en el puerto serial
  if (port.availableBytes() > 0) {

    // Lee UN byte proveniente del micro:bit
    dataRx = port.read(1);

    // Si el dato recibido es "A"
    if (dataRx == "A") {

      // Mueve el círculo hacia la izquierda
      x -= 10;

    // Si el dato recibido es "B"
    } else if (dataRx == "B") {

      // Mueve el círculo hacia la derecha
      x += 10;
    }

    // Limita el valor de x para que no salga del lienzo
    x = constrain(x, 0, width);
  }

  // Cambia el texto del botón según el estado del puerto
  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  } else {
    connectBtn.html("Disconnect");
  }
}

```
## El programa de micro:bit.
``` .asm
# Importa todas las librerías básicas del micro:bit
from microbit import *

# Inicializa la comunicación serial UART a 115200 baudios
uart.init(baudrate=115200)

# Muestra una imagen de mariposa al iniciar
display.show(Image.BUTTERFLY)

# Bucle infinito principal
while True:

    # Si el botón A está presionado
    if button_a.is_pressed():

        # Envía el carácter 'A' por el puerto serial
        uart.write('A')

        # Pausa de 500 ms para evitar envíos repetidos
        sleep(500)

    # Si el botón B está presionado
    if button_b.is_pressed():

        # Envía el carácter 'B' por el puerto serial
        uart.write('B')

        # Pausa de 500 ms
        sleep(500)

```


## codigo limpio (NO comentado)
``` .asm
let port;
let connectBtn;
let x= 350;
let dataRx;

function setup() {
  createCanvas(400, 400);
  background(220);
  port = createSerial();
 
  connectBtn = createButton("Connect to micro:bit");
  connectBtn.position(80, 300);
 connectBtn.mousePressed(connectBtnClick);

  
  fill("red");
  ellipse(x / 2, height / 2, 100, 100);
}
  function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
  }
function draw() {
  
  background(220);
        ellipse(x / 2, height / 2, 100, 100);
        fill('red');
        text(dataRx, x / 2, height / 2);
  
  if (port.availableBytes() > 0) {
    
     dataRx = port.read(1);
    if (dataRx == "A") {
      x -= 10;
    } else if (dataRx == "B") {
      x += 10;
    }
    x= constrain(x,0,width);
  
  }
 
    

  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  } else {
    connectBtn.html("Disconnect");
  }

}
```
## microbit:
``` .asm
# Imports go at the top
from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')


        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
       
```
# Bitácora de reflexión
-Lo aprendido en esta unidad son apenas las ideas basicas de como utlizar los programas,todavia no comprendo el nivel al que puede crear sin embargo lo aprendido puede servirme al futuro si logro comprender a mas profundidad las proximas unidades, por ahora quedo con las ideas fundamentales.
## Actividad 06
### Codigo de microbit
```.asm
from microbit import *// importa toda la libreria de herramientas de microbit para usar el microchip

uart.init(baudrate=115200)// es la inicializacion con la conexion serial el numero es un tipo de velocidad comunmente utilizada

while True:

    if button_a.is_pressed(): si el boton es presionado envia "A" en caso contrario envia "n"
        uart.write('A')
    else:
        uart.write('N')

    sleep(100)

```
### Codigo de p5.js
#### Index.Html
```.asm
 <script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script> // permite la conexión con el microbit
```
#### Sketch.js
```.asm
  let port;
  let connectBtn;
  let connectionInitialized = false;// variables libres 

  function setup() { // funcion que se ejecuta una sola vez
    createCanvas(400, 400);
    background(220); crea el canvas 
    port = createSerial();                 // crea el puerto serial,
    connectBtn = createButton("Connect to micro:bit"); //crea el boton que dice connect to microbit
    connectBtn.position(80, 300);    
    connectBtn.mousePressed(connectBtnClick);   //cuando se presiona el boton lleva a que acciones hace
  }

  function draw() {  // funcion que se ejecuta constantemente
    background(220);

    if (port.opened() && !connectionInitialized) { 
      port.clear();
      connectionInitialized = true;
    }

    if (port.availableBytes() > 0) { revisa si en el puerto hay algun mensaje disponible
      let dataRx = port.read(1);
      if (dataRx == "A") {
        fill("red");
      } else if (dataRx == "N") {
        fill("green");
      }
    }

    rectMode(CENTER);                   // crea el cuadrado
    rect(width / 2, height / 2, 50, 50);

    if (!port.opened()) {
      connectBtn.html("Connect to micro:bit");  // cambia el texto del boton dependiendo si es presionado
    } else {
      connectBtn.html("Disconnect");
    }
  }

  function connectBtnClick() { 
    if (!port.opened()) {
      port.open("MicroPython", 115200);   // se revisa si la conexion esta abierta entre el mirobit edito y p5.js
      connectionInitialized = false;
    } else {
      port.close();
    }
  }
```

colorin colorado esta unidad se ha acabado





