# Unidad 4

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
## MicrobitASCIIAdaptersf1.js
```
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error { }

function parseCsvLine(line) {
  const values = line.trim().split(",");
  if (values.length !== 4) throw new ParseError(`Expected 4 values, got ${values.length}`);

  const x = Number(values[0]);
  const y = Number(values[1]);
  const btnA = String(values[2]).trim().toLowerCase();
  const btnB = String(values[3]).trim().toLowerCase();

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  if (!["true", "false"].includes(btnA) || !["true", "false"].includes(btnB)) throw new ParseError("Invalid button data");

  return { x: x | 0, y: y | 0, btnA: btnA === "true", btnB: btnB === "true" };
}


class MicrobitAsciiAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = "";
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    this.buf += chunk.toString("utf8");

    let idx;
    while ((idx = this.buf.indexOf("\n")) >= 0) {
      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue;

      try {
        const parsed = parseCsvLine(line);
        this.onData?.(parsed);
      } catch (e) {
        if (e instanceof ParseError) {
          if (this.verbose) console.log("Bad data:", e.message, "raw:", line);
        } else {
          this._fail(e);
        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitAsciiAdapter;

```
## BridgServer.js
```
const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");

const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitAsciiAdapterSF1 = require("./adapters/MicrobitASCIIAdaptersf1");
const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};

function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() {
  return Date.now();
}

function safeJsonParse(s) {
  try {
    return JSON.parse(s);
  } catch {
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) {
      client.send(text);
    }
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();

  const microbit = ports.find(
    (p) => p.vendorId && parseInt(p.vendorId, 16) === 0x0d28
  );

  return microbit?.path ?? null;
}

async function createAdapter() {

  if (DEVICE === "microbit") {

    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found. Use --serialPort");
      process.exit(1);
    }

    log.info(`micro:bit found at ${path}`);

    return new MicrobitAsciiAdapter({
      path,
      baud: BAUD,
      verbose: VERBOSE
    });
  }

  if (DEVICE === "microbitsf1") {

    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found. Use --serialPort");
      process.exit(1);
    }

    log.info(`micro:bitsf1 found at ${path}`);

    return new MicrobitAsciiAdapterSF1({
      path,
      baud: BAUD,
      verbose: VERBOSE
    });
  }

  return new SimAdapter({ hz: SIM_HZ });
}

async function main() {

  const wss = new WebSocketServer({ port: WS_PORT });

  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected = (detail) => {
    log.info(`[ADAPTER] Connected: ${detail}`);
    status(wss, "connected", detail);
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Error: ${detail}`);
    status(wss, "error", detail);
  };

  adapter.onData = (d) => {
    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs()
    });
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {

    log.info(`Client connected from ${req.socket.remoteAddress}`);

    const state = adapter.connected ? "connected" : "ready";

    ws.send(
      JSON.stringify({
        type: "status",
        state,
        detail: `bridge (${DEVICE})`,
        t: nowMs()
      })
    );

    ws.on("message", async (raw) => {

      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        try {
          await adapter.connect();
        } catch (e) {
          status(wss, "error", e.message);
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        try {
          await adapter.disconnect();
        } catch (e) {
          status(wss, "error", e.message);
        }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        await adapter.handleCommand(msg);
        return;
      }

      if (msg.cmd === "setLed") {
        try {
          await adapter.handleCommand?.(msg);
        } catch (e) {
          status(wss, "error", e.message);
        }
      }
    });

    ws.on("close", () => {
      log.info("Client disconnected");

      if (wss.clients.size === 0) {
        adapter.disconnect();
      }
    });
  });

  if (DEVICE === "sim") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```
## indexsf1.html
```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Painter – Actividad 02</title>
  <link rel="stylesheet" href="style.css" />
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  <script src="fsm.js"></script>
  <script src="bridgeClient.js"></script>
  <script src="sketchsf1.js"></script>
</head>
<body>
</body>
</html>
```
## sketchsf1.js
```
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        this.c = color(181, 157, 0);
        this.lineSize = 100;
        this.angle = 0;
        this.clickPosX = 0;
        this.clickPosY = 0;

        this.circleResolution = 2;
        this.radius = 0;
        this.fillShape = false;

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false
        };

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            strokeWeight(2);
            background(255);
            stroke(0, 25);
            console.log("Microbit ready to draw");
            this.rxData = {
                x: 0,
                y: 0,
                btnA: false,
                btnB: false,
                prevA: false,
                prevB: false,
                ready: false
            };
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys?.(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease?.(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        this.rxData.ready = true;

        this.rxData.x = data.x;
        this.rxData.y = data.y;
        this.rxData.btnA = data.btnA;
        this.rxData.btnB = data.btnB;

        // Eje Y del acelerómetro -> resolución del polígono
        this.circleResolution = int(map(data.y, -2048, 2047, 2, 10));
        this.circleResolution = constrain(this.circleResolution, 2, 10);

        // Eje X del acelerómetro -> radio
        this.radius = map(data.x, -2048, 2047, -width / 2, width / 2);

        // Botón B -> relleno
        this.fillShape = data.btnB;
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(720, 720);
    background(255);
    painter = new PainterTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {
        painter.postEvent({
            type: EVENTS.DATA,
            payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
        });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    let mb = painter.rxData;

    if (!mb.ready) return;

    // Botón A -> dibujar
    if (mb.btnA) {
        push();
        translate(width / 2, height / 2);

        let circleResolution = painter.circleResolution;
        let radius = painter.radius;
        let angle = TAU / circleResolution;

        // Botón B -> relleno
        if (painter.fillShape) {
            fill(34, 45, 122, 50);
        } else {
            noFill();
        }

        beginShape();
        for (let i = 0; i <= circleResolution; i++) {
            let x = cos(angle * i) * radius;
            let y = sin(angle * i) * radius;
            vertex(x, y);
        }
        endShape();

        pop();
    }
}

function windowResized() {
    resizeCanvas(720, 720);
}
```
## microbit
```
from microbit import *

uart.init(115200)

display.set_pixel(0,0,9)

while True:

    x = accelerometer.get_x()
    y = accelerometer.get_y()

    btnA = button_a.is_pressed()
    btnB = button_b.is_pressed()

    data = "{},{},{},{}\n".format(
        x,
        y,
        str(btnA).lower(),
        str(btnB).lower()
    )

    uart.write(data)

    sleep(100)
```
## Bitácora de reflexión
