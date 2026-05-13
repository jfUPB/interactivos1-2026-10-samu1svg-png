# Unidad 7

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

# Bitácora — Integración Open Stage Control

## 1. Configuración de Open Stage Control

Open Stage Control se configuró como una fuente de datos externa que envía mensajes OSC por UDP hacia el backend del sistema.

Se estableció la conexión con los siguientes parámetros:

- Protocolo: OSC (UDP)
- Dirección: 127.0.0.1
- Puerto: 9000

El sistema no recibe JSON directamente, por lo que se utilizó la librería `osc` en Node.js para decodificar correctamente los mensajes entrantes.

---

## 2. Widgets utilizados y por qué

Se usaron tres widgets:

- RGB (`/rgb_1`) para cambiar el color
- Fader (`/size`) para el tamaño
- Toggle (`/toggle`) para activar o desactivar

La idea era simple: tener control en tiempo real.  
En la práctica fue más complicado de lo que parecía.

---

## 3. Estructura de mensaje

Se definió así:

```json
{
  "type": "osc",
  "payload": {
    "address": "/rgb_1",
    "args": [255, 120, 30]
  }
}
```
Esto permitió que todo el sistema entendiera los mensajes sin tener que rehacer todo otra vez.

## 4. Conexión del sistema
bridgeClient recibe los mensajes
FSMTask los organiza
updateLogic actualiza el estado
drawRunning dibuja

En teoría todo está separado y ordenado.
En la práctica, si algo falla en un punto, todo deja de tener sentido.

## 5. Integración de Strudel y OSC

Strudel manda eventos con tiempo (ritmo).
OSC cambia valores que se quedan (estado).

Uno dispara cosas.
El otro cambia cómo se ven.

Al final ambos terminan mezclándose en el mismo sistema, aunque claramente no fueron pensados para coexistir fácilmente.

## 6. Pruebas

Se hicieron pruebas básicas:

- mover el RGB → ver si cambia el color
- mover el size → ver si escala
- usar el toggle → ver si se apaga

También se revisaron logs porque nada funcionaba a la primera.

## 7. Problemas y soluciones
OSC no funcionaba → no era JSON
Adapter daba undefined → mal parseo
Visuales negras → datos corruptos
Size rompía todo → mal uso del lerp
Widgets mal configurados → errores mínimos pero destructivos

Todo se fue arreglando uno por uno, pero cada solución llevaba a otro problema.

Conclusión

Se logró hacer funcionar, pero no fue directo ni claro.
Más que implementar, fue ir corrigiendo errores constantemente.

El sistema funciona, pero entender por qué funciona tomó mucho más tiempo del esperado.

## Bitácora de reflexión
