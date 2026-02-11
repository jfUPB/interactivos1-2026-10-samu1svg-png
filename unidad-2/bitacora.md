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

##Actividad semana 4
```.asm
## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 



## Bitácora de reflexión

