# Unidad 3
### Actividad 1
semaforo con botones codigo:
```.asm
from microbit import *
import utime

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration

        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)


class Semaforo:
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        self.event_queue = []
        self.timers = []
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.createTimer("Timeout",self.timeInRed)

        self.estado_actual = None
        self.transicion_a(self.estado_waitInRed)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)

    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transicion_a(self.estado_waitInGreen)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)

        if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transicion_a(self.estado_waitInYellow)

        if ev == "A":
            self.transicion_a(self.estado_waitInYellow)
        if ev == "B":
            self.transicion_a(self.estado_Nocturno)

    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transicion_a(self.estado_waitInRed)
        if ev == "B":
            self.transicion_a(self.estado_Nocturno)


    
    def estado_Nocturno(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
           if display.get_pixel(self.x,self.y+1) ==0:
                display.set_pixel(self.x,self.y+1,9)
           else:
                display.set_pixel(self.x,self.y+1,0)
           self.myTimer.start(self.timeInYellow)
        
        if ev == "A":     
            self.transicion_a(self.estado_waitInYellow)
            

semaforo1 = Semaforo(0,0,2000,1000,500)

while True:
    if button_a.was_pressed():
        semaforo1.post_event("A")
    if button_b.was_pressed():
        semaforo1.post_event("B")
    semaforo1.update()
    utime.sleep_ms(20)

```
### Actividad 2 se uso el temporizador del estado armado se agrego un boton  que dice si se pausa el timer, y otro que al ingresar ABA ocurre un estado( el resto del codigo esta en la parte del notion de la act 2)
```.asm
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music

class Temporizador(FSMTask):
    def __init__(self):
        super().__init__()
        self.secuencia =[]
        self.mypassword =["A","B","A"]
        self.counter = 20
        self.myTimer = self.add_timer("Timeout",1000)
        self.estado_actual = None
        self.transition_to(self.estado_config)


    def estado_config(self, ev):
        if ev == ENTRY:
            self.counter = 20
            display.show(FILL[self.counter])
            self.myTimer.start()
        if ev == "A":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])
        if ev == "B":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])
        if ev == "S":
            self.transition_to(self.estado_armed)

    def estado_armed(self, ev):
        if ev == ENTRY:
            self.secuencia.clear()
            self.myTimer.start()
        if ev == "Timeout":
            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])
                if self.counter == 0:
                    self.transition_to(self.estado_timeout)
                else:
                    self.myTimer.start()
        if ev == "A":
            if self.myTimer.active == False:
                self.myTimer.start()
            else:
                self.myTimer.stop()
        if ev == "A" or ev == "B":
            self.secuencia.append(ev)
            if len(self.secuencia) == 3:
                if self.secuencia == self.mypassword:
                    self.transition_to(self.estado_config)
                else :
                    self.secuencia.clear()
        

    def estado_timeout(self, ev):
        if ev == ENTRY:
            display.show(Image.SKULL)
            music.play(music.FUNERAL)
        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)

temporizador = Temporizador()

while True:

    if button_a.was_pressed():
        temporizador.post_event("A")
    if button_b.was_pressed():
        temporizador.post_event("B")
    if accelerometer.was_gesture("shake"):
        temporizador.post_event("S")

    temporizador.update()
    utime.sleep_ms(20)
```
## Bitácora de proceso de aprendizaje
#### codigo documentado de la aplicación
```.asm
// ===============================
// CONSTANTES INTERNAS DE LA FSM
// ===============================

// Evento automático que se dispara cuando se entra a un estado
const ENTRY = "entry";

// Evento automático que se dispara cuando se sale de un estado
const EXIT = "exit";


// ===============================
// CLASE FSMTimer
// Maneja temporizadores asociados a eventos
// ===============================

class FSMTimer {

  // Constructor del timer
  // eventName → nombre del evento que se enviará
  // interval → tiempo en milisegundos
  // task → referencia a la máquina de estados
  constructor(eventName, interval, task) {
    this.eventName = eventName;      // Nombre del evento a disparar
    this.interval = interval;       // Tiempo de espera
    this.task = task;               // FSM asociada
    this.timeoutId = null;          // ID interno del setTimeout
  }

  // Inicia el temporizador
  start() {
    this.stop(); // Evita duplicados

    this.timeoutId = setTimeout(() => {
      // Cuando termina el tiempo, envía evento a la FSM
      this.task.postEvent(this.eventName);
    }, this.interval);
  }

  // Detiene el temporizador si está activo
  stop() {
    if (this.timeoutId !== null) {
      clearTimeout(this.timeoutId);
      this.timeoutId = null;
    }
  }
}


// ===============================
// CLASE BASE FSMTask
// Motor general de máquina de estados
// ===============================

class FSMTask {

  constructor() {
    this.state = null;        // Estado actual (función)
    this.eventQueue = [];     // Cola de eventos pendientes
    this.timers = [];         // Lista de timers asociados
  }

  // Crea un temporizador y lo asocia a la FSM
  addTimer(eventName, interval) {
    const timer = new FSMTimer(eventName, interval, this);
    this.timers.push(timer);
    return timer;
  }

  // Agrega un evento a la cola
  postEvent(event) {
    this.eventQueue.push(event);
  }

  // Cambia de estado
  transitionTo(newState) {

    // Ejecuta lógica de salida del estado actual
    if (this.state) {
      this.state(EXIT);
    }

    // Asigna nuevo estado
    this.state = newState;

    // Ejecuta lógica de entrada del nuevo estado
    if (this.state) {
      this.state(ENTRY);
    }
  }

  // Procesa todos los eventos pendientes
  update() {
    while (this.eventQueue.length > 0) {
      const ev = this.eventQueue.shift();

      if (this.state) {
        this.state(ev);
      }
    }
  }
}
```
#### Sketch.js 
```.asm
// ===============================
// CONFIGURACIÓN DEL TEMPORIZADOR
// ===============================

// Valores mínimo, máximo y valor inicial
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

// Eventos que entiende la FSM
const EVENTS = {
  DEC: "A",         // Disminuir / Pausar
  INC: "B",         // Incrementar
  START: "S",       // Iniciar
  TICK: "Timeout",  // Evento automático del timer
};

// Variables globales
let serial;            // Comunicación serial
let secuencia = [];    // Guarda combinación secreta
let temporizador;      // Instancia principal FSM


// ===============================
// CLASE TEMPORIZADOR
// Hereda de FSMTask
// ===============================

class Temporizador extends FSMTask {

  constructor(minValue, maxValue, defaultValue) {
    super(); // Llama constructor base FSM

    this.pausado = false;

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;

    this.configValue = defaultValue;      // Valor en modo configuración
    this.totalSeconds = defaultValue;     // Tiempo total configurado
    this.remainingSeconds = defaultValue; // Tiempo restante

    // Crea timer que dispara evento cada 1000 ms
    this.myTimer = this.addTimer(EVENTS.TICK, 1000);

    // Estado inicial
    this.transitionTo(this.estado_config);
  }

  // Devuelve estado actual
  get currentState() {
    return this.state;
  }

  // ===============================
  // ESTADO CONFIGURACIÓN
  // ===============================

  estado_config = (ev) => {

    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
    }

    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    }

    else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    }

    else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };


  // ===============================
  // ESTADO EN CUENTA REGRESIVA
  // ===============================

  estado_armed = (ev) => {

    if (ev === ENTRY) {
      this.pausado = false;
      this.myTimer.start();
    }

    else if (ev === EVENTS.TICK) {

      if (!this.pausado && this.remainingSeconds > 0) {

        this.remainingSeconds--;

        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      } else {
        this.myTimer.start();
      }
    }

    else if (ev === EVENTS.DEC) {
      this.pausado = !this.pausado;
    }

    else if (ev === EXIT) {
      this.myTimer.stop();
    }
  };


  // ===============================
  // ESTADO FINAL (TIMEOUT)
  // ===============================

  estado_timeout = (ev) => {
    if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };
}


// ===============================
// SETUP
// ===============================

function setup() {

  createCanvas(windowWidth, windowHeight);
  textAlign(CENTER, CENTER);

  // Inicializa serial
  serial = createSerial();

  // Botón de conexión
  let connectButton = createButton("Conectar micro:bit");
  connectButton.position(20, 20);
  connectButton.mousePressed(() => serial.requestPort());

  // Crea instancia FSM
  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
}


// ===============================
// DRAW
// ===============================

function draw() {

  background(30);

  // Lectura serial
  if (serial.available()) {
    let str = serial.readLine();
    if (str) {
      manejarEntrada(str.trim());
    }
  }

  // Actualiza FSM
  temporizador.update();

  // Render según estado
  if (temporizador.currentState === temporizador.estado_config) {
    textSize(100);
    fill(255);
    text(temporizador.configValue, width / 2, height / 2);
  }

  if (temporizador.currentState === temporizador.estado_armed) {
    textSize(100);
    fill(255);
    text(temporizador.remainingSeconds, width / 2, height / 2);

    if (temporizador.pausado) {
      textSize(30);
      text("PAUSADO", width / 2, height / 2 + 80);
    }
  }

  if (temporizador.currentState === temporizador.estado_timeout) {
    textSize(80);
    fill(255, 0, 0);
    text("¡TIEMPO!", width / 2, height / 2);
  }
}


// ===============================
// MANEJO DE ENTRADAS
// ===============================

function manejarEntrada(valor) {
  if (valor === "S") temporizador.postEvent(EVENTS.START);
  if (valor === "A") procesarA();
  if (valor === "B") procesarB();
}

function procesarA() {
  guardarSecuencia("A");
  temporizador.postEvent(EVENTS.DEC);
}

function procesarB() {
  guardarSecuencia("B");
  temporizador.postEvent(EVENTS.INC);
}


// ===============================
// SECUENCIA SECRETA
// ===============================

function guardarSecuencia(letra) {

  secuencia.push(letra);

  if (secuencia.length > 3) secuencia.shift();

  if (
    secuencia.join("") === "ABA" &&
    temporizador.currentState === temporizador.estado_armed
  ) {
    temporizador.pausado = false;
    temporizador.transitionTo(temporizador.estado_config);
    secuencia = [];
  }
}


// ===============================
// TECLADO
// ===============================

function keyPressed() {
  if (key === "a" || key === "A") procesarA();
  if (key === "b" || key === "B") procesarB();
  if (key === "s" || key === "S") temporizador.postEvent(EVENTS.START);
}
```
#### index.html
```.asm
<!DOCTYPE html>
<!-- Declara que el documento usa HTML5 -->

<html lang="es">
<!-- Inicio del documento HTML y especifica que el idioma es español -->

<head>
  <meta charset="UTF-8">
  <!-- Permite usar caracteres especiales como tildes y ñ -->

  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <!-- Hace que la página se adapte correctamente a pantallas pequeñas -->

  <title>Proyecto FSM con micro:bit</title>
  <!-- Título que aparece en la pestaña del navegador -->

  <!-- Librería p5.js para gráficos y manejo de teclado -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.0/p5.min.js"></script>

  <!-- Enlace al archivo principal donde está toda la lógica del proyecto -->
  <script src="sketch.js"></script>

</head>

<body>
  <!-- Cuerpo de la página -->

  <h1>Simulación FSM con micro:bit</h1>
  <!-- Título visible en la página -->

  <p>Presiona A, B o S para probar los estados.</p>
  <!-- Instrucciones para el usuario -->

</body>
</html>
```
#### microbit editor
```.asm
from microbit import *
# Importa todas las funciones y objetos de la librería microbit
# (botones, acelerómetro, display, uart, etc.)

import utime
# Importa el módulo de tiempo para poder hacer pausas en milisegundos

uart.init(baudrate=115200)
# Inicializa la comunicación serial (UART)
# baudrate=115200 define la velocidad de transmisión de datos
# Debe coincidir con la velocidad configurada en el programa receptor (p5.js o PC)

while True:
# Bucle infinito → el micro:bit ejecuta esto constantemente

    if button_a.was_pressed():
        # Verifica si el botón A fue presionado

        uart.write("A\n")
        # Envía la letra A seguida de salto de línea
        # El "\n" es importante para que el receptor sepa que terminó el mensaje

    if button_b.was_pressed():
        # Verifica si el botón B fue presionado

        uart.write("B\n")
        # Envía la letra B seguida de salto de línea

    if accelerometer.was_gesture("shake"):
        # Detecta si el micro:bit fue agitado

        uart.write("S\n")
        # Envía la letra S (Start)

        utime.sleep_ms(500)
        # Espera 500 milisegundos para evitar múltiples envíos por un solo shake

    utime.sleep_ms(20)
    # Pequeña pausa de 20 ms para:
    # - Reducir consumo
    # - Evitar sobrecargar la comunicación
    # - Hacer el loop más estable
```
## Bitácora de aplicación 
### fsm.js
```.asm

const ENTRY = "entry";
const EXIT = "exit";


class FSMTimer {
  constructor(eventName, interval, task) {
    this.eventName = eventName;
    this.interval = interval;
    this.task = task;
    this.timeoutId = null;
  }

  start() {
    this.stop();
    this.timeoutId = setTimeout(() => {
      this.task.postEvent(this.eventName);
    }, this.interval);
  }

  stop() {
    if (this.timeoutId !== null) {
      clearTimeout(this.timeoutId);
      this.timeoutId = null;
    }
  }
}



class FSMTask {
  constructor() {
    this.state = null;
    this.eventQueue = [];
    this.timers = [];
  }

  addTimer(eventName, interval) {
    const timer = new FSMTimer(eventName, interval, this);
    this.timers.push(timer);
    return timer;
  }

  postEvent(event) {
    this.eventQueue.push(event);
  }

  transitionTo(newState) {
    if (this.state) {
      this.state(EXIT);
    }

    this.state = newState;

    if (this.state) {
      this.state(ENTRY);
    }
  }

  update() {
    while (this.eventQueue.length > 0) {
      const ev = this.eventQueue.shift();
      if (this.state) {
        this.state(ev);
      }
    }
  }
}
```
### Sketch.js
```.asm
const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};

let serial;
let secuencia = [];
let temporizador;

class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();

    this.pausado = false;
    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;

    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);
    this.transitionTo(this.estado_config);
  }

  get currentState() {
    return this.state;
  }

  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
    }
    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    }
    else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    }
    else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.pausado = false;
      this.myTimer.start();
    }
    else if (ev === EVENTS.TICK) {
      if (!this.pausado && this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      } else {
        this.myTimer.start();
      }
    }
    else if (ev === EVENTS.DEC) {
      this.pausado = !this.pausado;
    }
    else if (ev === EXIT) {
      this.myTimer.stop();
    }
  };

  estado_timeout = (ev) => {
    if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };
}

function setup() {
  createCanvas(windowWidth, windowHeight);
  textAlign(CENTER, CENTER);

  // Serial compatible
  serial = createSerial();

  let connectButton = createButton("Conectar micro:bit");
  connectButton.position(20, 20);
  connectButton.mousePressed(() => serial.requestPort());

  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );
}

function draw() {
  background(30);

  // Leer datos serial correctamente
  if (serial.available()) {
    let str = serial.readLine();
    if (str) {
      str = str.trim();
      manejarEntrada(str);
    }
  }

  temporizador.update();

  if (temporizador.currentState === temporizador.estado_config) {
    textSize(100);
    fill(255);
    text(temporizador.configValue, width / 2, height / 2);
  }

  if (temporizador.currentState === temporizador.estado_armed) {
    textSize(100);
    fill(255);
    text(temporizador.remainingSeconds, width / 2, height / 2);

    if (temporizador.pausado) {
      textSize(30);
      text("PAUSADO", width / 2, height / 2 + 80);
    }
  }

  if (temporizador.currentState === temporizador.estado_timeout) {
    textSize(80);
    fill(255, 0, 0);
    text("¡TIEMPO!", width / 2, height / 2);
  }
}

function manejarEntrada(valor) {
  if (valor === "S") temporizador.postEvent(EVENTS.START);
  if (valor === "A") procesarA();
  if (valor === "B") procesarB();
}

function procesarA() {
  guardarSecuencia("A");
  temporizador.postEvent(EVENTS.DEC);
}

function procesarB() {
  guardarSecuencia("B");
  temporizador.postEvent(EVENTS.INC);
}

function guardarSecuencia(letra) {
  secuencia.push(letra);
  if (secuencia.length > 3) secuencia.shift();

  if (
    secuencia.join("") === "ABA" &&
    temporizador.currentState === temporizador.estado_armed
  ) {
    temporizador.pausado = false;
    temporizador.transitionTo(temporizador.estado_config);
    secuencia = [];
  }
}

function keyPressed() {
  if (key === "a" || key === "A") procesarA();
  if (key === "b" || key === "B") procesarB();
  if (key === "s" || key === "S") temporizador.postEvent(EVENTS.START);
}
```
### index.html
```.asm
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>Temporizador</title>

  <!-- p5 -->
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.1/lib/p5.js"></script>

  <!-- p5 WebSerial compatible con createSerial() -->
  <script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script>
</head>

<body>

  <!-- PRIMERO fsm -->
  <script src="fsm.js"></script>

  <!-- DESPUÉS sketch -->
  <script src="sketch.js"></script>

</body>
</html>
```
### microbit
```.asm
from microbit import *
import utime

uart.init(baudrate=115200)

while True:
    if button_a.was_pressed():
        uart.write("A\n")
    if button_b.was_pressed():
        uart.write("B\n")
    if accelerometer.was_gesture("shake"):
        uart.write("S\n")
        utime.sleep_ms(500)

    utime.sleep_ms(20)
```


## Bitácora de reflexión


