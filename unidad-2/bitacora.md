# Unidad 2
actividad de los timers: 
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


class Pixel:
    def __init__(self,_x,_y,_interval):
       self.event_queue = []
    
       self.timers = []
       self.myTimer = self.createTimer("Timeout",_interval)
       self.posX = _x
       self.posY = _y

       self.estado_actual = None
       self.transicion_a(self.estado_waitInON)
     
    
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

    ## Comienzo como tal de la aplicación
    def estado_waitInON(self,ev):
        if ev == "ENTRY":
            display.set_pixel(self.posX,self.posY,9)
            self.myTimer.start()
        
            
        if ev == "Timeout":
            self.transicion_a(self.estado_waitInOff)
            pass
    def estado_waitInOff(self,ev):
         if ev == "ENTRY":
            display.set_pixel(self.posX,self.posY,0)
            self.myTimer.start()
            
         if ev == "Timeout":
            self.transicion_a(self.estado_waitInOff)


pixel1 = Pixel(0,0,1000)

while True:
    pixel1.update()
    utime.sleep_ms(20)
```

##Actividad 4
```.asm
# ================================
# IMPORTACIONES
# ================================

from microbit import *     # Permite usar display, botones, acelerómetro, imágenes, etc.
import utime               # Permite trabajar con tiempo en milisegundos
import music               # Permite reproducir sonidos en el speaker


# ================================
# FUNCION PARA CREAR IMÁGENES DE RELLENO
# ================================

def make_fill_images(on='9', off='0'):
    """
    Crea una lista de 26 imágenes (0 a 25 pixeles encendidos).
    Los pixeles se llenan desde arriba-izquierda hacia abajo-derecha.
    """
    imgs = []  # Lista donde guardaremos las imágenes

    for n in range(26):  # Desde 0 hasta 25 pixeles encendidos
        rows = []        # Filas de la imagen actual
        k = 0            # Contador de pixeles

        for y in range(5):  # 5 filas
            row = []
            for x in range(5):  # 5 columnas
                # Si el contador es menor que n, enciende pixel
                # Si no, lo deja apagado
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))  # Une los pixeles de la fila

        # Crea la imagen final y la agrega a la lista
        imgs.append(Image(':'.join(rows)))

    return imgs


# Creamos todas las imágenes una sola vez
FILL = make_fill_images()
# Para mostrar una imagen: display.show(FILL[n])


# ================================
# CLASE TIMER (NO USA sleep)
# ================================

class Timer:
    """
    Timer basado en ticks_ms().
    Cuando termina, genera un evento en la máquina de estados.
    """

    def __init__(self, owner, event_to_post, duration):
        self.owner = owner          # Objeto que recibirá el evento
        self.event = event_to_post  # Nombre del evento
        self.duration = duration    # Duración en milisegundos
        self.start_time = 0         # Momento en que inicia
        self.active = False         # Indica si está activo

    def start(self, new_duration=None):
        """
        Inicia el temporizador.
        Puede recibir nueva duración opcional.
        """
        if new_duration is not None:
            self.duration = new_duration

        self.start_time = utime.ticks_ms()  # Guarda tiempo actual
        self.active = True

    def stop(self):
        """Detiene el temporizador."""
        self.active = False

    def update(self):
        """
        Verifica si el tiempo terminó.
        Si terminó, envía evento a la FSM.
        """
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)


# ================================
# CLASE PRINCIPAL (MAQUINA DE ESTADOS)
# ================================

class Task:

    def __init__(self):
        # Cola de eventos
        self.event_queue = []

        # Lista de timers internos
        self.timers = []

        # Creamos un timer que generará evento "Timeout" cada 1000ms
        self.myTimer = self.createTimer("Timeout", 1000)

        # Tiempo inicial obligatorio
        self.tiempo = 20

        # Estado actual
        self.estado_actual = None

        # Estado inicial: configuración
        self.transicion_a(self.estado_configuracion)


    # ================================
    # MÉTODOS GENERALES FSM
    # ================================

    def createTimer(self, event, duration):
        """Crea un nuevo timer y lo guarda."""
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        """Agrega evento a la cola."""
        self.event_queue.append(ev)

    def update(self):
        """
        1. Actualiza todos los timers
        2. Procesa los eventos pendientes
        """
        for t in self.timers:
            t.update()

        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        """
        Maneja transición formal:
        1. Ejecuta EXIT del estado actual
        2. Cambia estado
        3. Ejecuta ENTRY del nuevo estado
        """
        if self.estado_actual:
            self.estado_actual("EXIT")

        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")


    # ================================
    # ESTADO 1: CONFIGURACIÓN
    # ================================

    def estado_configuracion(self, ev):

        if ev == "ENTRY":
            # Mostrar cantidad actual de pixeles
            display.show(FILL[self.tiempo])

        elif ev == "A":
            # Aumentar tiempo hasta máximo 25
            if self.tiempo < 25:
                self.tiempo += 1
            display.show(FILL[self.tiempo])

        elif ev == "B":
            # Disminuir tiempo hasta mínimo 15
            if self.tiempo > 15:
                self.tiempo -= 1
            display.show(FILL[self.tiempo])

        elif ev == "S":
            # Shake arma el temporizador
            self.transicion_a(self.estado_armado)


    # ================================
    # ESTADO 2: ARMADO (CUENTA REGRESIVA)
    # ================================

    def estado_armado(self, ev):

        if ev == "ENTRY":
            # Inicia timer de 1 segundo
            self.myTimer.start(1000)

        elif ev == "Timeout":
            # Cada segundo ocurre esto
            if self.tiempo > 0:
                self.tiempo -= 1
                display.show(FILL[self.tiempo])
                self.myTimer.start(1000)  # Reinicia el timer
            else:
                # Cuando llega a 0 → estado final
                self.transicion_a(self.estado_final)

        elif ev == "EXIT":
            # Detener timer al salir del estado
            self.myTimer.stop()


    # ================================
    # ESTADO 3: FINAL
    # ================================

    def estado_final(self, ev):

        if ev == "ENTRY":
            # Mostrar calavera
            display.show(Image.SKULL)

            # Reproducir sonido
            music.play(music.DADADADUM)

        elif ev == "A":
            # Reinicia a valor inicial
            self.tiempo = 20
            self.transicion_a(self.estado_configuracion)


# ================================
# PROGRAMA PRINCIPAL
# ================================

# Crear instancia de la máquina de estados
task = Task()

# Bucle infinito principal
while True:

    # Generar eventos desde hardware

    if button_a.was_pressed():
        task.post_event("A")

    if button_b.was_pressed():
        task.post_event("B")

    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    # Actualizar FSM
    task.update()

    # Pequeña pausa técnica (NO afecta lógica de tiempo)
    utime.sleep_ms(20)

```
## codigo del diagrama(revisar)
```.asm
@startuml
skinparam monochrome true
skinparam dpi 150
skinparam state {
    BackgroundColor white
    BorderColor black
}

[*] --> Configuracion

state Configuracion {
}

Configuracion : ENTRY / display.show(FILL[tiempo])
Configuracion : A [tiempo < 25] / tiempo++\ndisplay.show(FILL[tiempo])
Configuracion : B [tiempo > 15] / tiempo--\ndisplay.show(FILL[tiempo])

Configuracion --> Armado : S / myTimer.start(1000)


state Armado {
}

Armado : ENTRY / myTimer.start(1000)\n
Armado : Armado : Timeout [tiempo > 0] / tiempo--\ndisplay.show(FILL[tiempo])\nmyTimer.start(1000)\n
Armado : EXIT / myTimer.stop()


Armado --> Final : Timeout [tiempo == 0]


state Final {
}

Final : ENTRY / display.show(SKULL)\nmusic.play(DADADADUM)
Final --> Configuracion : A / tiempo = 20\n display.show(FILL[tiempo])

@enduml
```
##codigo del diagrama documentado
```.asm
DIAGRAMA DE MÁQUINA DE ESTADOS EN PLANTUML
------------------------------------------
Este bloque contiene el código PlantUML que modela
la máquina de estados del temporizador en micro:bit.
"""

plantuml_code = """
@startuml
' Inicia el diagrama UML de estados

skinparam monochrome true
' Fuerza el diagrama en blanco y negro para claridad académica

skinparam dpi 150
' Mejora la resolución de exportación de la imagen

skinparam state {
    BackgroundColor white
    BorderColor black
}
' Define estilo visual de los estados


[*] --> Configuracion
' Estado inicial del sistema apunta a "Configuracion"


state Configuracion {
}
' Declaración del estado de configuración


Configuracion : ENTRY / display.show(FILL[tiempo])
' Al entrar al estado se muestra el tiempo actual en pantalla

Configuracion : A [tiempo < 25] / tiempo++\\ndisplay.show(FILL[tiempo])
' Botón A aumenta el tiempo si es menor que 25 y actualiza la pantalla

Configuracion : B [tiempo > 15] / tiempo--\\ndisplay.show(FILL[tiempo])
' Botón B disminuye el tiempo si es mayor que 15 y actualiza la pantalla


Configuracion --> Armado : S / myTimer.start(1000)
' Al detectar sacudida (shake) pasa al estado Armado
' e inicia el temporizador de 1 segundo



state Armado {
}
' Declaración del estado Armado


Armado : ENTRY / myTimer.start(1000)
' Al entrar se inicia el temporizador
Armado : Timeout [tiempo > 0] / tiempo--\\ndisplay.show(FILL[tiempo])\\nmyTimer.start(1000)
' Cada segundo reduce el tiempo, actualiza pantalla
' y reinicia el temporizador mientras tiempo > 0
Armado : EXIT / myTimer.stop()
' Al salir del estado se detiene el temporizador




Armado --> Final : Timeout [tiempo == 0]
' Cuando el tiempo llega a cero se pasa al estado Final



state Final {
}
' Declaración del estado Final


Final : ENTRY / display.show(SKULL)\\nmusic.play(DADADADUM)
' Al entrar muestra una calavera y reproduce sonido

Final --> Configuracion : A / tiempo = 20\\ndisplay.show(FILL[tiempo])
' Botón A reinicia el tiempo a 20 y vuelve a Configuración


@enduml
' Fin del diagrama UML
```
## Bitácora de proceso de aprendizaje
### Actividad 5
codigo nuevo de microbit
```.asm
# ================================
# IMPORTACIONES
# ================================

from microbit import *
import utime
import music

# ================================
# ACTIVAR UART (COMUNICACIÓN SERIAL)
# ================================

uart.init(baudrate=115200)

# ================================
# FUNCION PARA CREAR IMÁGENES DE RELLENO
# ================================

def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()

# ================================
# TIMER
# ================================

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

# ================================
# MAQUINA DE ESTADOS
# ================================

class Task:

    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.myTimer = self.createTimer("Timeout", 1000)
        self.tiempo = 20
        self.estado_actual = None
        self.transicion_a(self.estado_configuracion)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()

        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    # ================================
    # ESTADO CONFIGURACIÓN
    # ================================

    def estado_configuracion(self, ev):

        if ev == "ENTRY":
            display.show(FILL[self.tiempo])

        elif ev == "A":
            if self.tiempo < 25:
                self.tiempo += 1
            display.show(FILL[self.tiempo])

        elif ev == "B":
            if self.tiempo > 15:
                self.tiempo -= 1
            display.show(FILL[self.tiempo])

        elif ev == "S":
            self.transicion_a(self.estado_armado)

    # ================================
    # ESTADO ARMADO
    # ================================

    def estado_armado(self, ev):

        if ev == "ENTRY":
            self.myTimer.start(1000)

        elif ev == "Timeout":
            if self.tiempo > 0:
                self.tiempo -= 1
                display.show(FILL[self.tiempo])
                self.myTimer.start(1000)
            else:
                self.transicion_a(self.estado_final)

        elif ev == "EXIT":
            self.myTimer.stop()

    # ================================
    # ESTADO FINAL
    # ================================

    def estado_final(self, ev):

        if ev == "ENTRY":
            display.show(Image.SKULL)
            music.play(music.DADADADUM)

        elif ev == "A":
            self.tiempo = 20
            self.transicion_a(self.estado_configuracion)


# ================================
# PROGRAMA PRINCIPAL
# ================================

task = Task()

while True:

    # EVENTOS DESDE BOTONES FÍSICOS
    if button_a.was_pressed():
        task.post_event("A")

    if button_b.was_pressed():
        task.post_event("B")

    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    # EVENTOS DESDE p5.js (SERIAL)
    # EVENTOS DESDE p5.js (SERIAL)
    if uart.any():
        raw = uart.read()
    
        if raw is not None:
            data = raw.decode().strip()
    
            for char in data:
                if char == "A":
                    task.post_event("A")
                elif char == "B":
                    task.post_event("B")
                elif char == "S":
                    task.post_event("S")

    task.update()
    utime.sleep_ms(20)

```
##Codigo de p5.js
```.asm
let port;
let connectBtn;

function setup() {
  createCanvas(400, 200);
  background(30);

  connectBtn = createButton("Conectar micro:bit");
  connectBtn.mousePressed(connectSerial);

  textAlign(CENTER, CENTER);
  textSize(18);
}

function draw() {
  background(30);
  fill(255);
  text("Teclas:\nA = UP\nB = DOWN\nS = ARM", width/2, height/2);
}

async function connectSerial() {
  port = await navigator.serial.requestPort();
  await port.open({ baudRate: 115200 });
}

function keyPressed() {
  if (!port) return;

  const writer = port.writable.getWriter();

  if (key === 'A' || key === 'a') {
    writer.write(new TextEncoder().encode("A"));
  }

  if (key === 'B' || key === 'b') {
    writer.write(new TextEncoder().encode("B"));
  }

  if (key === 'S' || key === 's') {
    writer.write(new TextEncoder().encode("S"));
  }

  writer.releaseLock();
}

```
## Bitácora de aplicación 



## Bitácora de reflexión






