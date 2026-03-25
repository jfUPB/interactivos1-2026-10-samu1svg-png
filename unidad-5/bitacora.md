# Unidad 5

## Bitácora de aplicación 
## Microbit:editor
``` javascript
from microbit import *

import struct
 
uart.init(115200)
 
HEADER = b'\xAA'
 
while True:

    xValue = accelerometer.get_x()

    yValue = accelerometer.get_y()

    aState = button_a.is_pressed()

    bState = button_b.is_pressed()
 
    payload = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
 
    checksum = sum(payload) % 256

    packet = HEADER + payload + bytes([checksum])

    uart.write(packet)

    sleep(100)
 
``` 
## **MicrobitBinaryAdapter.js**
``` javascript
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class MicrobitBinaryAdapter extends BaseAdapter {

  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;

    // 🔥 CAMBIO 1: ahora es buffer binario, no string
    this.buf = Buffer.alloc(0);

    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required");

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
        this.port.close((err) => (err ? reject(err) : resolve()));
      });
    }

    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    console.log(chunk);

    // 🔥 CAMBIO 2: acumular bytes
    this.buf = Buffer.concat([this.buf, chunk]);

    // 🔥 LOOP de parsing binario
    while (this.buf.length >= 8) {

      // 🔥 CAMBIO 3: buscar header 0xAA
      if (this.buf[0] !== 0xAA) {
        this.buf = this.buf.slice(1);
        continue;
      }

      // 🔥 CAMBIO 4: asegurar paquete completo
      if (this.buf.length < 8) return;

      const packet = this.buf.slice(0, 8);

      // 🔥 CAMBIO 5: checksum
      const checksum =
        (packet[1] +
          packet[2] +
          packet[3] +
          packet[4] +
          packet[5] +
          packet[6]) % 256;

      if (checksum !== packet[7]) {
        console.warn("Bad checksum, discarding frame");
        this.buf = this.buf.slice(1); // 🔥 importante: no 8
        continue;
      }

      // 🔥 CAMBIO 6: leer datos binarios
      const x = packet.readInt16BE(1);
      const y = packet.readInt16BE(3);
      const btnA = packet[5] === 1;
      const btnB = packet[6] === 1;

      // 🔥 CAMBIO 7: emitir igual que ASCII
      this.onData?.({ x, y, btnA, btnB });

      // 🔥 CAMBIO 8: consumir paquete
      this.buf = this.buf.slice(8);
    }

    // 🔥 limpieza buffer
    if (this.buf.length > 4096) {
      this.buf = Buffer.alloc(0);
    }
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = Buffer.alloc(0);
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

module.exports = MicrobitBinaryAdapter;
```
# --**Cambios en el Bridgeserver**--
## Importar el adapter binario
``` javascript
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
``` 
## Añadir caso en createAdapter()
``` javascript
if (DEVICE === "microbitbinary") {
  const path = SERIAL_PATH ?? await findMicrobitPort();

  if (!path) {
    log.error("micro:bit not found. Use --serialPort");
    process.exit(1);
  }

  log.info(`micro:bit (binary) found at ${path}`);

  return new MicrobitBinaryAdapter({
    path,
    baud: BAUD,
    verbose: VERBOSE
  });
}
```
## Bridge server completo
``` javascript
// Uso:
// node bridgeServer.js
// node bridgeServer.js --device microbit
// node bridgeServer.js --device microbitbinary

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");

const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

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

function nowMs() {
  return Date.now();
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
const VERBOSE = process.argv.includes("--verbose");

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
      log.error("micro:bit not found");
      process.exit(1);
    }

    return new MicrobitAsciiAdapter({
      path,
      baud: BAUD,
      verbose: VERBOSE
    });
  }

  if (DEVICE === "microbitbinary") {
    const path = SERIAL_PATH ?? await findMicrobitPort();

    if (!path) {
      log.error("micro:bit not found");
      process.exit(1);
    }

    return new MicrobitBinaryAdapter({
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

    log.info(`Client connected`);

    const state = adapter.connected ? "connected" : "ready";

    ws.send(JSON.stringify({
      type: "status",
      state,
      detail: `bridge (${DEVICE})`,
      t: nowMs()
    }));

    ws.on("message", async (raw) => {

      let msg;
      try {
        msg = JSON.parse(raw.toString("utf8"));
      } catch {
        return;
      }

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

## Bitácora de reflexión
