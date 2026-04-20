# Unidad 6

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
# Bitácora — Integración Strudel Adapter

## 1. Objetivo

Integrar un sistema de generación musical (Strudel) con un sistema visual existente respetando una arquitectura desacoplada basada en:

* Adapter (recepción y normalización)
* bridgeServer (reenvío)
* bridgeClient (recepción en frontend)
* FSMTask (organización de eventos)
* updateLogic (estado)
* drawRunning (render)

---

## 2. Primer intento: conexión directa con Strudel (WebSocket)

Se intentó conectar Strudel online directamente mediante WebSocket hacia el adapter.

### Problemas encontrados:

* Errores de parseo en Strudel al intentar usar código JavaScript fuera de su sintaxis.
* Strudel online no permite abrir conexiones WebSocket personalizadas.
* No se lograba enviar eventos al bridge.

### Conclusión:

El entorno web de Strudel limita la comunicación externa, por lo que no es viable una conexión directa desde el navegador.

---

## 3. Segundo intento: uso de `fetch` (HTTP)

Se implementó un adapter HTTP para recibir eventos enviados desde Strudel usando `fetch`.

### Problemas encontrados:

* Aunque el código no generaba errores, los eventos no siempre eran enviados.
* `onTrigger` en Strudel no se ejecutaba de forma consistente.
* Dependencia del contexto de audio del navegador.

### Conclusión:

El envío de eventos desde Strudel online mediante HTTP no es confiable para sincronización en tiempo real.

---

## 4. Tercer intento: simulación de eventos

Se decidió implementar un `StrudelAdapter` que genera eventos musicales simulados con:

* sonidos: `bd`, `sd`, `hh`
* parámetro `delta`
* timestamps

### Problemas encontrados:

* Uso de `setInterval` generaba desfase (drift) en el tiempo.
* Los eventos no mantenían un ritmo estable.

### Solución aplicada:

Se reemplazó `setInterval` por un sistema basado en tiempo acumulado (`nextTime`), logrando:

* ritmo constante
* mayor estabilidad temporal

---

## 5. Problema crítico: desincronización por timestamp

Los eventos contenían timestamps incompatibles con el tiempo del frontend.

### Síntoma:

* Los eventos nunca se ejecutaban en la FSM.
* Pantalla sin visualización o respuestas tardías.

### Solución:

Se implementó un sistema de compensación de offset:

```js
timeOffset = Date.now() - ev.timestamp;
ev.timestamp += timeOffset;
```

Esto permitió sincronizar los eventos con el tiempo del navegador.

---

## 6. Integración con FSM

Se mantuvo la arquitectura:

* `bridgeClient.js` solo recibe y reenvía eventos.
* `FSMTask` organiza los eventos.
* `updateLogic` transforma eventos en estado visual.
* `drawRunning` únicamente dibuja.

Se evitó:

* lógica visual en el adapter
* lógica de red en el render

---

## 7. Problemas en visualización

### Problemas:

* Uso de posiciones aleatorias → visual caótico
* Falta de correspondencia clara entre sonido y forma

### Solución:

Se implementó un sistema de carriles:

* `bd` → pulso circular
* `sd` → cuadrado con movimiento vertical
* `hh` → líneas verticales

Esto permitió una visualización clara y legible.

---

## 8. Resultado final

El sistema final logra:

* Generación de eventos musicales simulados
* Normalización en el adapter
* Reenvío correcto por el bridge
* Procesamiento en FSM
* Visualización sincronizada

Se respetó completamente la arquitectura desacoplada.

---

## 9. Conclusiones

* La separación de responsabilidades fue clave para depurar errores.
* Los principales problemas estuvieron en la sincronización temporal.
* La simulación de eventos fue necesaria debido a limitaciones del entorno web.
* El sistema final es estable, entendible y defendible.

---
## parentesis 
Luego de mostrar la simulación el profesor pidio que el sistema fuera realmente funciona y que cumpliera con la rubrica por lo que accedio a darme mas tiempo para el cambio

## 10. Integración final y funcionamiento del sistema

En la fase final se logró la integración real con Strudel, identificando que el problema principal no era de código sino de comunicación: el sistema utilizaba dos bridges en puertos distintos, por lo que los eventos nunca llegaban al frontend. La solución consistió en unificar el flujo en un mismo puerto, permitiendo que Strudel enviara eventos correctamente al bridgeServer, el cual los reenvía al frontend.

El StrudelAdapter cumple el rol de traducir los mensajes entrantes (formato tipo OSC) a un formato normalizado con type, timestamp y payload, evitando que el frontend procese datos crudos. De esta forma se mantiene la arquitectura desacoplada.

Además, se implementó un sistema de cola temporal (eventQueue) donde los eventos no se ejecutan al llegar, sino cuando el tiempo local (Date.now()) alcanza su timestamp. Esto permite separar transporte de ejecución, mejorando la sincronización audiovisual y evitando problemas de latencia.

Como resultado, el sistema recibe eventos reales en tiempo real, los normaliza, los programa temporalmente y los representa visualmente de forma coherente con el ritmo musical, cumpliendo los requisitos funcionales y arquitectónicos de la actividad.

## Bitácora de reflexión
