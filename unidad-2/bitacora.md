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

[*] --> Configuracion

Configuracion : tiempo = 20 \n
Configuracion : if/\n \n A [tiempo < 25]\n / tiempo++\n display.show(FILL[tiempo])\n
Configuracion : if/\n \n B [tiempo > 15]\n / tiempo--\n display.show(FILL[tiempo])
Configuracion --> Armado : S\n / myTimer.start(1000)

Armado : timer activo
Armado --> Armado : Timeout [tiempo > 0]\n / tiempo--\n display.show(FILL[tiempo])\n myTimer.start(1000)
Armado --> Final : Timeout [tiempo == 0]\n / display.show(SKULL)\n music.play(...)
Armado --> Configuracion : EXIT\n / myTimer.stop()

Final --> Configuracion : A\n / tiempo=20\n display.show(FILL[tiempo])

@enduml

```
## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 



## Bitácora de reflexión


