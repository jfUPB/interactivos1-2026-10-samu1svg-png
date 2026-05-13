# Unidad 8

## Bitácora de proceso de aprendizaje
### strudel musica2:
```
// ============================================
// VARIACION 3
// MÁS POST-PUNK / ROCK
// ============================================

setcps(0.5)

let kick =
  s("bd bd ~ bd ~ bd bd ~")
    .gain(0.75)
    .shape(0.1)

let snare =
  s("~ ~ sd ~ ~ ~ sd sd")
    .gain(0.7)
    .room(0.4)

let hats =
  s("hh*8")
    .gain(0.025)
    .hpf(12000)

let bass =
  note("c1 c1 g0 bb0")
    .sound("square")
    .gain(0.65)
    .distort(0.1)
    .lpf(300)

let guitar1 =
  note("g2 bb2 eb2 f2")
    .sound("gm_distortion_guitar")
    .gain(0.6)
    .room(0.7)

let guitar2 =
  note("g3 ~ d3 ~")
    .sound("gm_overdriven_guitar")
    .gain(0.28)
    .room(1)
    .slow(4)

let noiseLayer =
  noise()
    .gain(0.004)
    .lpf(700)

$: stack(kick, kick.osc())
$: stack(snare, snare.osc())
$: stack(hats, hats.osc())
$: stack(bass, bass.osc())
$: stack(guitar1, guitar1.osc())
$: stack(guitar2, guitar2.osc())
$: stack(noiseLayer)
```
### strudel musica1:
```
// ========================================
// LIVE CONTROLS
// ========================================

let tempo = 0.38

let kickGain = 0.7
let snareGain = 0.55
let hatsGain = 0.08

let bassSpeed = 4
let guitarSpeed = 5

let ambience = 1

// ========================================

setcps(tempo)

// ========================================
// DRUMS
// ========================================

let kick =
  s("bd ~ ~ ~ ~ ~ ~ ~")
    .gain(kickGain)
    .shape(0.12)

let snare =
  s("~ ~ ~ sd ~ ~ ~ ~")
    .gain(snareGain)
    .room(0.5 * ambience)

let hats =
  s("~ ~ hh ~ ~ ~ hh ~")
    .gain(hatsGain)
    .hpf(9000)
    .room(0.2)
    .pan(
      sine.range(-0.15,0.15).slow(8)
    )

// ========================================
// CYMBALS
// ========================================

let cymbal1 =
  s("~ ~ ~ oh")
    .gain(0.22)
    .room(0.98 * ambience)
    .slow(4)

let cymbal2 =
  s("~ ~ ~ hc")
    .gain(0.14)
    .room(1 * ambience)
    .slow(8)

// ========================================
// BASS
// ========================================

let bass =
  note("c1 ~ g0 ~")
    .sound("square")
    .gain(0.6)
    .distort(0.12)
    .lpf(220)
    .slow(bassSpeed)

// ========================================
// GUITARS
// ========================================

let guitar1 =
  note("g2 ~ eb2 ~")
    .sound("gm_distortion_guitar")
    .gain(0.55)
    .room(0.92 * ambience)
    .lpf(1600)
    .slow(guitarSpeed)

let guitar2 =
  note("g3 ~ f3 ~")
    .sound("gm_overdriven_guitar")
    .gain(0.38)
    .room(1 * ambience)
    .slow(guitarSpeed + 2)

// ========================================
// DRONE
// ========================================

let drone =
  note("c2")
    .sound("triangle")
    .gain(0.035)
    .room(1)
    .slow(40)

// ========================================
// OUTPUTS
// ========================================

$: stack(
  kick,
  kick.osc()
)

$: stack(
  snare,
  snare.osc()
)

$: stack(
  hats,
  hats.osc()
)

$: stack(
  cymbal1,
  cymbal1.osc()
)

$: stack(
  cymbal2,
  cymbal2.osc()
)

$: stack(
  bass,
  bass.osc()
)

$: stack(
  guitar1,
  guitar1.osc()
)

$: stack(
  guitar2,
  guitar2.osc()
)

$: stack(
  drone,
  drone.osc()
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
