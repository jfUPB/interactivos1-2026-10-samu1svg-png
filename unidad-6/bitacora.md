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

# (rompio la arquitectura se comenzo desde cero)
# Bitácora — Integración Strudel

## 1. ¿Cómo configuré Strudel para emitir eventos?

Se configuró Strudel para funcionar como generador de eventos musicales en el navegador, enviando mensajes mediante WebSocket hacia el puerto `8080`.

Para las pruebas, se utilizó un cliente WebSocket que enviaba mensajes con la estructura esperada por Strudel:

- Dirección del evento (`/dirt/play`)
- Argumentos en formato clave-valor (`args`)
- Marca temporal (`timestamp`)

Ejemplo de mensaje emitido:

```json
{
  "address": "/dirt/play",
  "args": ["s", "bd", "delta", 0.5],
  "timestamp": 1710000000000
}
```
## 2. ¿Qué estructura final de mensaje decidí usar?

Se definió un contrato normalizado para desacoplar el sistema del formato crudo de Strudel:
```json
{
  "type": "strudel",
  "timestamp": 1710000000000,
  "payload": {
    "sound": "bd",
    "delta": 0.5
  }
}
```
Este formato:

Es claro y consistente
Evita exponer el formato interno de Strudel al frontend
Facilita la extensión futura del sistema
## 3. ¿Cómo conecté bridgeClient.js, FSMTask, updateLogic y drawRunning?

La conexión se realizó respetando la arquitectura por capas:

bridgeClient.js
Recibe los mensajes desde bridgeServer.js
Identifica mensajes de tipo "strudel"
Dispara eventos hacia la FSM mediante postEvent
FSMTask (StrudelTask)
Recibe eventos ya normalizados
Maneja estados (esperando → corriendo)
Encola eventos musicales
updateLogic
Procesa la cola de eventos (eventQueue)
Ejecuta eventos cuando el tiempo actual alcanza su timestamp
Actualiza el estado de las animaciones (progreso temporal)
drawRunning
Solo dibuja
No interpreta eventos ni lógica de red
Usa el estado calculado para renderizar
## 4. ¿Cómo separé recepción, cola temporal y renderizado?

Se separaron claramente las responsabilidades:

Recepción
Ocurre en bridgeClient.js
Solo recibe y reenvía eventos a la FSM
Cola temporal (Scheduling)
Implementada en StrudelTask mediante eventQueue
Los eventos se ordenan por timestamp
Se ejecutan únicamente cuando corresponde en el tiempo
Renderizado
Implementado en drawRunning
Solo dibuja animaciones activas
No contiene lógica de red ni parsing

Esto permite mantener un sistema desacoplado y fácil de mantener.

## 5. ¿Qué pruebas hice para verificar la sincronización?

Se realizaron varias pruebas:

Envío manual de eventos con timestamp futuro para verificar ejecución retardada
Uso de console.log en:
recepción de mensajes
normalización
entrada a la cola
ejecución de eventos
Comparación visual entre ritmo de eventos y aparición de animaciones
Pruebas variando delta para verificar duración de animaciones

Se comprobó que:

Los eventos no se ejecutan al llegar
Se respetan los tiempos definidos por timestamp
Las animaciones siguen el ritmo correctamente
## 6. Problemas encontrados y soluciones

Durante la integración de Strudel se presentaron varios problemas relacionados con arquitectura, sincronización y configuración del sistema.

Inicialmente, el sistema no reconocía el `StrudelAdapter` debido a una ruta incorrecta en el `require`. Esto se solucionó corrigiendo la ruta hacia la carpeta `adapters`.

Otro problema importante fue que los eventos se ejecutaban inmediatamente al llegar, ignorando su `timestamp`. Esto ocurría porque no existía una lógica de scheduling. Se resolvió implementando una cola de eventos (`eventQueue`) ordenada por tiempo y ejecutando cada evento solo cuando `Date.now()` alcanzaba su `timestamp`.

También se detectó que las visuales no coincidían con el comportamiento del repositorio de referencia. La causa era el uso de decrementos artificiales (`life--`) en lugar de un progreso basado en tiempo real. Se solucionó calculando un `progress` usando `(tiempo actual - tiempo inicial) / duración`, lo que permitió sincronizar correctamente las animaciones con el ritmo.


Finalmente, Strudel no se conectaba automáticamente porque el sistema estaba diseñado para dispositivos como micro:bit. Se solucionó creando un servidor WebSocket independiente en el puerto 8080 para recibir los eventos de Strudel sin interferir con el resto del sistema.


## Bitácora de reflexión
