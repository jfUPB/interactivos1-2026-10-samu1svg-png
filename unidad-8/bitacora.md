# Unidad 8

## Bitácora de proceso de aprendizaje
### strudel musica:
```
setcps(0.38)

stack(

  // BOMBO MUY SUAVE
  s("bd ~ ~ ~ ~ ~ ~ ~")
    .gain(0.7)
    .shape(0.12),

  // SNARE LEJANA
  s("~ ~ ~ sd ~ ~ ~ ~")
    .gain(0.55)
    .room(0.5),

  // HI HATS DELICADOS
  s("~ ~ hh ~ ~ ~ hh ~")
    .gain(0.08)
    .hpf(9000)
    .room(0.2)
    .pan(sine.range(-0.15,0.15).slow(8)),

  // PLATILLO AMBIENTAL
  s("~ ~ ~ oh")
    .gain(0.22)
    .room(0.98)
    .slow(4),

  // PLATILLO LARGO
  s("~ ~ ~ hc")
    .gain(0.14)
    .room(1)
    .slow(8),

  // BAJO OSCURO ACOMPAÑANDO
  note("c1 ~ g0 ~")
    .sound("square")
    .gain(0.6)
    .distort(0.12)
    .lpf(220)
    .slow(4),

  // GUITARRA DISTORSIONADA PRINCIPAL
  note("g2 ~ eb2 ~")
    .sound("gm_distortion_guitar")
    .gain(0.55)
    .room(0.92)
    .lpf(1600)
    .slow(5),

  // GUITARRA FLOTANTE MELANCOLICA
  note("g3 ~ f3 ~")
    .sound("gm_overdriven_guitar")
    .gain(0.38)
    .room(1)
    .slow(7),

  // DRONE CASI IMPERCEPTIBLE
  note("c2")
    .sound("triangle")
    .gain(0.035)
    .room(1)
    .slow(40),

  // TEXTURA ANALOGICA
  noise()
    .gain(0.008)
    .lpf(900)

)
```

### Microbit: editor codigo:
```
from microbit import *

uart.init(baudrate=115200)

while True:
    x = accelerometer.get_x()
    y = accelerometer.get_y()

    a = button_a.is_pressed()
    b = button_b.is_pressed()

    uart.write(
        str(x) + "," +
        str(y) + "," +
        str(a) + "," +
        str(b) + "\n"
    )

    sleep(30)
```
## Bitácora de aplicación 
### Diagrama inicial de arquitectura
```
micro:bit ──> MicrobitAsciiAdapter ─┐
                                    │
Strudel ───> StrudelAdapter ────────┼──> bridgeServer ──> bridgeClient ──> FSMTask ──> updateLogic ──> drawRunning
                                    │
OSC ───────> OpenStageControlAdapter┘
```
La idea principal fue mantener la arquitectura del curso sin cambiarla o dañarla como lo hice en las anteriores actividades. Cada fuente de datos entra por su propio adapter, luego bridgeServer solo redistribuye mensajes y el frontend se encarga de interpretar el comportamiento del sistema.

- se utilizaron los siguientes adapters:
- 
#### MicrobitAsciiAdapter

Se utilizó para recibir datos seriales enviados desde micro:bit usando UART. El adapter parsea aceleración X/Y y botones A/B.

contrato de mensaje:
```
{
  "type": "microbit",
  "x": 120,
  "y": -340,
  "btnA": false,
  "btnB": true,
  "t": 123456
}
```

#### StrudelAdapter

Se utilizó para recibir eventos musicales desde Strudel mediante WebSocket. El adapter normaliza sonidos y duración de eventos.

contrato de mensaje:
```
{
  "type": "strudel",
  "timestamp": 123456,
  "payload": {
    "sound": "bd",
    "delta": 0.25
  }
}
```

#### OpenStageControlAdapter

Se utilizó para recibir controles persistentes vía OSC UDP desde Open Stage Control.

contrato de mensaje:
```
{
  "type": "osc",
  "payload": {
    "address": "/size",
    "args": [1.5]
  }
}

```
## Pruebas tecnicas de integración
Se realizaron pruebas independientes y simultáneas con las tres fuentes de entrada.

Se utilizaron console logs en:

bridgeServer;
bridgeClient;
FSMTask.

- lo que ayudo a verificar si realmente estaban llegando datos al frontend


# Problemas encontrados
--
### Problema 1 — Arquitectura incorrecta

- Inicialmente parte de la lógica de Strudel estaba directamente dentro de bridgeServer.js y algunos adapters no heredaban de BaseAdapter.

### Solución

- Se reorganizó la arquitectura:

- cada fuente quedó en su propio adapter;
- todos heredaron de BaseAdapter;
- la lógica de parsing se movió fuera de bridgeServer.js.
--
## Problema 2 — Visuales no aparecían

- Después de reorganizar adapters, el frontend quedaba en negro.

## Solución

- Se agregaron console logs en:

- adapters;
- bridgeServer;
- bridgeClient;
- FSMTask.

Esto permitió detectar que los eventos sí llegaban pero no estaban entrando correctamente a la FSM.

--
## Problema 3 — Error en micro:bit Python

- El código inicial del micro:bit fue escrito usando sintaxis de JavaScript dentro del editor Python.

## Solución

- Se reescribió usando MicroPython y UART para enviar datos CSV compatibles con MicrobitAsciiAdapter.
--
## Problema 4 — Compatibilidad futura con micro:bit

- Durante la reorganización se perdió temporalmente parte de la estructura original del bridge server relacionada con selección de device.

## Solución

- Se restauró la estructura base del curso para mantener compatibilidad con:

- sim;
- micro:bit;
- adapters futuros.
--
