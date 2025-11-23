<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>SuperOS — serverless, auditable, composable (A+)</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
:root {
  --bg: #0b0e12; --panel: #12161d; --border: #1f2530;
  --text: #c9d1d9; --muted: #8b949e; --primary: #58a6ff; --accent: #f0c674;
  --mono: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
  --sans: system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, Noto Sans, "Helvetica Neue", Arial, "Apple Color Emoji", "Segoe UI Emoji";
}
* { box-sizing: border-box }
html, body { height:100% }
body { margin:0; background:var(--bg); color:var(--text); font:14px/1.5 var(--sans) }
header { position:fixed; top:0; left:0; right:0; z-index:10; background:linear-gradient(180deg, rgba(18,22,29,.95), rgba(18,22,29,.85)); backdrop-filter:blur(6px); border-bottom:1px solid var(--border); padding:8px 12px }
main { position:relative; padding-top:72px }
canvas#main-canvas { position:fixed; inset:0; z-index:0 }
.nav { display:flex; gap:8px; align-items:center; flex-wrap:wrap }
.nav a { color:var(--muted); text-decoration:none; padding:6px 10px; border-radius:6px; border:1px solid transparent }
.nav a:hover { color:var(--text); border-color:var(--border) }
.nav a.active { color:var(--text); border-color:var(--primary) }
.badge { background:#0d1117; border:1px solid var(--border); color:var(--primary); padding:6px 10px; border-radius:8px; margin-right:8px }
#stats { display:flex; gap:12px; align-items:center; margin-top:6px; color:var(--muted) }
.pill, .pill.mono { display:inline-block; background:#0d1117; border:1px solid var(--border); padding:4px 8px; border-radius:999px; margin-right:6px }
.mono { font-family:var(--mono) }
.controls { display:flex; gap:8px; flex-wrap:wrap; margin-top:8px }
.button { background:#0d1117; color:var(--text); border:1px solid var(--border); padding:8px 12px; border-radius:8px; cursor:pointer }
.button.primary { border-color:var(--primary); color:var(--primary) }
.button.accent { border-color:var(--accent); color:var(--accent) }
.button[aria-pressed="true"] { background:rgba(88,166,255,0.12) }
.section { position:relative; z-index:1; padding:16px }
.grid { display:grid; grid-template-columns:repeat(auto-fit, minmax(260px, 1fr)); gap:12px }
.card { background:var(--panel); border:1px solid var(--border); border-radius:10px; padding:12px }
.row { display:flex; gap:8px; align-items:center; flex-wrap:wrap }
.kv { display:grid; grid-template-columns: 140px 1fr; gap:8px 12px; align-items:center }
.dropzone { border:2px dashed var(--border); border-radius:10px; padding:16px; text-align:center }
details summary { cursor:pointer }
hr { border:0; border-top:1px solid var(--border); margin:12px 0 }
pre { background:#0d1117; padding:12px; border-radius:8px; overflow:auto }
textarea, input { background:#0d1117; color:var(--text); border:1px solid var(--border); border-radius:8px; padding:8px }
label, span { color:var(--muted) }
ul { margin:0; padding-left:18px }
a.inline { color:var(--primary); text-decoration:none }
</style>
</head>
<body>
<header id="app-header">
  <div class="nav"></div>
  <div id="stats">
    <span class="pill mono"><strong>Update ms:</strong> <span id="up-ms">0</span></span>
    <span class="pill mono" id="fps">FPS: 0</span>
    <span class="pill mono" id="hw">Virtual HW: GOD MODE 1</span>
    <span class="pill mono"><strong>Particles:</strong> <span id="particleCount">0</span></span>
    <span class="pill mono"><strong>Grid cells:</strong> <span id="gridCells">0</span></span>
  </div>
  <div class="controls">
    <button id="upgradeBtn" class="button primary" title="u">Upgrade HW</button>
    <button id="aiBtn" class="button" aria-pressed="true" title="a">AI Substrate: ON</button>
    <button id="spawnBtn" class="button accent" title="space">Spawn burst</button>
    <button id="clearBtn" class="button" title="c">Clear swarm</button>
  </div>
</header>

<main>
  <canvas id="main-canvas" aria-label="Swarm canvas"></canvas>
  <section id="app-root" class="section"></section>
</main>

<script>
"use strict";

/* =========================
   Utilities and helpers
========================= */
const now = () => performance.now();
const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
const rand = (min, max) => Math.random() * (max - min) + min;
const randInt = (min, max) => Math.floor(rand(min, max + 1));
const distance = (a, b) => Math.hypot(a.x - b.x, a.y - b.y);

function el(tag, attrs = {}, children = []) {
  const node = document.createElement(tag);
  for (const [k, v] of Object.entries(attrs)) {
    if (k === "class") node.className = v;
    else if (k === "text") node.textContent = v;
    else if (k.startsWith("on")) node.addEventListener(k.slice(2).toLowerCase(), v);
    else node.setAttribute(k, v);
  }
  [].concat(children).forEach(c => {
    if (c === null || c === undefined) return;
    node.appendChild(typeof c === "string" ? document.createTextNode(c) : c);
  });
  return node;
}
function mount(parent, node) { parent.appendChild(node); return node; }

/* =========================
   Structured audit bus
========================= */
const Bus = (() => {
  const subs = new Set();
  const emit = (type, payload, level = "info") => {
    const evt = { ts: Date.now(), type, level, payload };
    subs.forEach(fn => {
      try { fn(evt); } catch (err) {
        console.warn("[BUS subscriber error]", err);
      }
    });
  };
  return {
    subscribe: fn => { subs.add(fn); return () => subs.delete(fn); },
    emit,
    error: (type, err, payload) => emit(type, { error: String(err), ...payload }, "error"),
    warn: (type, payload) => emit(type, payload, "warn")
  };
})();

/* =========================
   Immutable store (auditable)
========================= */
const store = (() => {
  let state = {};
  const subs = new Set();
  const clone = o => JSON.parse(JSON.stringify(o));
  return {
    init: s => { state = s; subs.forEach(fn => fn(clone(state))); },
    get: () => clone(state),
    set: p => {
      state = { ...state, ...p };
      Bus.emit("store:update", { keys: Object.keys(p) });
      subs.forEach(fn => fn(clone(state)));
    },
    subscribe: fn => { subs.add(fn); fn(clone(state)); return () => subs.delete(fn); }
  };
})();

/* =========================
   Router (hash-based)
========================= */
function Router({ routes, onChange }) {
  const parse = () => ({ path: (location.hash.replace(/^#/, "") || "/") });
  const notify = () => onChange(parse());
  return {
    start: () => { window.addEventListener("hashchange", notify, { passive: true }); notify(); }
  };
}

/* =========================
   Worker sandbox (capability isolated)
========================= */
function createWorker(fn) {
  const src = `self.onmessage = async (e) => {
    const api = (${fn.toString()})(self);
    try {
      const res = await api.handle(e.data);
      self.postMessage({ ok: true, res });
    } catch (err) {
      self.postMessage({ ok: false, err: String(err) });
    }
  }`;
  const blob = new Blob([src], { type: "application/javascript" });
  const url = URL.createObjectURL(blob);
  const w = new Worker(url);
  w._url = url;
  return w;
}
function callWorker(w, msg, { timeoutMs = 10000 } = {}) {
  return new Promise((resolve, reject) => {
    let done = false;
    const to = setTimeout(() => {
      if (done) return;
      done = true;
      w.removeEventListener("message", handler);
      reject(new Error("Worker timeout"));
    }, timeoutMs);
    const handler = (e) => {
      const { ok, res, err } = e.data || {};
      if (done) return;
      done = true;
      clearTimeout(to);
      w.removeEventListener("message", handler);
      ok ? resolve(res) : reject(new Error(err));
    };
    w.addEventListener("message", handler);
    w.postMessage(msg);
  }).finally(() => {
    try { w.terminate(); } catch {}
    if (w._url) { try { URL.revokeObjectURL(w._url); } catch {} }
  });
}

/* =========================
   Parsers (hardened minimal set)
========================= */
const Parsers = {
  // JSON with safer parse + fallback
  json: async (text) => {
    try { return JSON.parse(text); }
    catch (e) { return { __raw__: text, __error__: String(e) }; }
  },
  // CSV: header + rows, handles quotes, escapes, empty cells
  csv: async (text) => {
    const lines = text.trim().split(/\r?\n/);
    if (!lines.length) return [];
    const splitCSV = (line) => {
      const cells = []; let cur = ""; let inQ = false;
      for (let i = 0; i < line.length; i++) {
        const ch = line[i];
        if (ch === '"' && line[i + 1] === '"') { cur += '"'; i++; continue; }
        if (ch === '"') { inQ = !inQ; continue; }
        if (ch === ',' && !inQ) { cells.push(cur); cur = ""; continue; }
        cur += ch;
      }
      cells.push(cur);
      return cells.map(c => c.trim());
    };
    const header = splitCSV(lines.shift());
    return lines.map(line => {
      const cells = splitCSV(line);
      const obj = {};
      for (let i = 0; i < header.length; i++) obj[header[i]] = cells[i] ?? "";
      return obj;
    });
  },
  // Minimal TOML: sections + key=value for string/number/bool; ignores arrays/dates
  toml: async (text) => {
    const obj = {}; let cur = obj;
    for (const raw of text.split(/\r?\n/)) {
      const s = raw.trim();
      if (!s || s.startsWith("#")) continue;
      if (s.startsWith("[") && s.endsWith("]")) {
        const sec = s.slice(1, -1).trim();
        obj[sec] = obj[sec] || {};
        cur = obj[sec]; continue;
      }
      const m = s.match(/^([^=]+)=(.+)$/);
      if (!m) continue;
      const k = m[1].trim(); let v = m[2].trim();
      if (/^".*"$/.test(v)) v = v.slice(1, -1);
      else if (/^(true|false)$/i.test(v)) v = v.toLowerCase() === "true";
      else if (!Number.isNaN(Number(v))) v = Number(v);
      cur[k] = v;
    }
    return obj;
  },
  // Minimal YAML: key: value, nesting by 2 spaces, lists with "- "
  yaml: async (text) => {
    const root = {}; const stack = [{ indent: -1, obj: root }];
    for (const raw of text.split(/\r?\n/)) {
      const line = raw.replace(/\t/g, " ");
      const trimmed = line.trim();
      if (!trimmed || trimmed.startsWith("#")) continue;
      const indent = (line.match(/^ */)[0] || "").length;
      while (stack.length && indent <= stack[stack.length - 1].indent) stack.pop();
      const ctx = stack[stack.length - 1].obj;
      if (trimmed.startsWith("- ")) {
        const item = trimmed.slice(2).trim();
        const key = "__list__";
        (ctx[key] = ctx[key] || []).push(parseScalar(item));
        continue;
      }
      const m = trimmed.match(/^([^:]+):\s*(.*)$/);
      if (!m) continue;
      const k = m[1].trim(); let v = m[2].trim();
      if (!v) { ctx[k] = {}; stack.push({ indent, obj: ctx[k] }); }
      else { ctx[k] = parseScalar(v); }
    }
    return root;
    function parseScalar(v) {
      if (/^".*"$/.test(v)) return v.slice(1, -1);
      if (/^(true|false)$/i.test(v)) return v.toLowerCase() === "true";
      if (!Number.isNaN(Number(v))) return Number(v);
      return v;
    }
  }
};

/* =========================
   Canonical translator
========================= */
const Translator = {
  inferType: (obj) => {
    if (!obj || typeof obj !== "object") return "unknown";
    if (obj.model || obj.models) return "ai";
    if (obj.rpc || obj.chainId || obj.network) return "blockchain";
    if (obj.pool || obj.pools || obj.amm) return "defi";
    if (obj.dex || obj.orderbook || obj.swap) return "dex";
    if (obj.webrtc || obj.signal || obj.peers) return "p2p";
    if (obj.margin || obj.perp || obj.leverage) return "leverage";
    if (obj.document || obj.content || obj.editor) return "editor";
    return "unknown";
  },
  wrap: (obj) => ({
    type: Translator.inferType(obj),
    meta: { detectedAt: Date.now() },
    data: obj
  })
};

/* =========================
   Capability registry
========================= */
const Capabilities = {
  AI: { id: "ai", ops: ["listModels", "run"], validate: (cfg) => cfg && typeof cfg === "object" },
  Blockchain: { id: "blockchain", ops: ["listNodes", "call"], validate: (cfg) => cfg && typeof cfg === "object" },
  DeFi: { id: "defi", ops: ["listPools", "quote", "swap"], validate: (cfg) => cfg && typeof cfg === "object" },
  DEX: { id: "dex", ops: ["markets", "orderbook", "place"], validate: (cfg) => cfg && typeof cfg === "object" },
  P2P: { id: "p2p", ops: ["peers", "signal", "send"], validate: (cfg) => cfg && typeof cfg === "object" },
  Leverage: { id: "leverage", ops: ["pairs", "position", "open", "close"], validate: (cfg) => cfg && typeof cfg === "object" },
  Editor: { id: "editor", ops: ["open", "format", "export"], validate: (cfg) => cfg && typeof cfg === "object" }
};

const Registry = (() => {
  const adapters = new Map(); // key -> {capability, title, workerFactory, schema?}
  const register = (key, spec) => {
    if (!spec || !spec.capability || !spec.workerFactory) {
      throw new Error("Invalid adapter spec");
    }
    adapters.set(key, spec);
    Bus.emit("registry:add", { key, capability: spec.capability.id, title: spec.title });
  };
  const list = () => Array.from(adapters.entries()).map(([k, v]) => ({ key: k, ...v }));
  const get = (key) => adapters.get(key);
  return { register, list, get };
})();

/* =========================
   Example adapters (improved)
========================= */
Registry.register("ai.local", {
  capability: Capabilities.AI,
  title: "AI (Local Stub)",
  workerFactory: () => createWorker((self) => ({
    handle: async (msg) => {
      const { op, args } = msg || {};
      if (op === "listModels") return [{ id: "llama.cpp", type: "local" }, { id: "mlc", type: "webgpu" }];
      if (op === "run") {
        const prompt = String(args?.prompt || "");
        return { output: `Echo: ${prompt.slice(0, 400)}`, tokens: prompt.length };
      }
      throw new Error("Unsupported op");
    }
  }))
});

Registry.register("chain.eth", {
  capability: Capabilities.Blockchain,
  title: "Blockchain (ETH RPC)",
  workerFactory: () => createWorker((self) => ({
    handle: async ({ op, args }) => {
      if (op === "listNodes") return [{ id: "geth", rpc: "http://localhost:8545", status: "offline" }];
      if (op === "call") {
        const { rpc, method, params = [] } = args || {};
        if (!rpc || !method) throw new Error("rpc and method required");
        // Basic POST JSON-RPC
        let res;
        try {
          res = await fetch(rpc, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ jsonrpc: "2.0", id: 1, method, params })
          }).then(r => r.json());
        } catch (e) {
          res = { error: String(e) };
        }
        return res;
      }
      throw new Error("Unsupported op");
    }
  }))
});

Registry.register("defi.amm", {
  capability: Capabilities.DeFi,
  title: "DeFi (AMM Math)",
  workerFactory: () => createWorker((self) => ({
    handle: async ({ op, args }) => {
      if (op === "listPools") return [{ id: "pool-ETH/USDC", type: "constant-product", x: 1000, y: 2_000_000 }];
      if (op === "quote") {
        const { x, y, dx, fee = 0.003 } = args || {};
        // Validate inputs
        if ([x, y, dx, fee].some(v => typeof v !== "number" || !Number.isFinite(v) || v < 0)) {
          throw new Error("Invalid input");
        }
        const k = x * y;
        const xNew = x + dx * (1 - fee);
        const yNew = k / xNew;
        const dy = y - yNew;
        return { dy, price: dy / dx, fee };
      }
      if (op === "swap") return { tx: "local-simulated", status: "ok" };
      throw new Error("Unsupported op");
    }
  }))
});

Registry.register("dex.stub", {
  capability: Capabilities.DEX,
  title: "DEX (Orderbook Stub)",
  workerFactory: () => createWorker((self) => ({
    handle: async ({ op }) => {
      if (op === "markets") return [{ id: "ETH-USD", tickSize: 0.5 }];
      if (op === "orderbook") return { bids: [[1999.5, 2]], asks: [[2000.5, 1.5]] };
      if (op === "place") return { orderId: "ord_" + Date.now(), status: "accepted" };
      throw new Error("Unsupported op");
    }
  }))
});

Registry.register("p2p.webrtc", {
  capability: Capabilities.P2P,
  title: "P2P (WebRTC Stub)",
  workerFactory: () => createWorker((self) => ({
    handle: async ({ op }) => {
      if (op === "peers") return [{ id: "local", status: "idle" }];
      if (op === "signal") return { status: "stubbed" };
      if (op === "send") return { status: "queued" };
      throw new Error("Unsupported op");
    }
  }))
});

Registry.register("lev.perp", {
  capability: Capabilities.Leverage,
  title: "Leverage (Perp Math)",
  workerFactory: () => createWorker((self) => ({
    handle: async ({ op }) => {
      if (op === "pairs") return [{ id: "ETH-USD", maxLev: 20 }];
      if (op === "position") return { pnl: 12.34, leverage: 5, collateral: 1000 };
      if (op === "open") return { posId: "pos_" + Date.now(), status: "opened" };
      if (op === "close") return { status: "closed" };
      throw new Error("Unsupported op");
    }
  }))
});

Registry.register("editor.basic", {
  capability: Capabilities.Editor,
  title: "Editor (Format/Export)",
  workerFactory: () => createWorker((self) => ({
    handle: async ({ op, args }) => {
      if (op === "open") return { content: args?.content || "" };
      if (op === "format") {
        const s = String(args?.content || "");
        return { content: s.replace(/\s+$/gm, "") };
      }
      if (op === "export") {
        const s = String(args?.content || "");
        return { blobUrl: URL.createObjectURL(new Blob([s], { type: "text/plain" })) };
      }
      throw new Error("Unsupported op");
    }
  }))
});

/* =========================
   Swarm agent + engine (optimized)
========================= */
class SwarmAgent {
  constructor(id, x, y) {
    this.id = id; this.x = x; this.y = y;
    this.vx = 0; this.vy = 0;
    this.energy = 100; this.active = true;
    this.nearby = []; this.env = { width: 0, height: 0 };
  }
  perceive(agents, env, grid, cellSize) {
    this.env = env;
    // Lookup neighbors via spatial grid
    const cx = Math.floor(this.x / cellSize);
    const cy = Math.floor(this.y / cellSize);
    const neighbors = [];
    for (let gy = cy - 1; gy <= cy + 1; gy++) {
      for (let gx = cx - 1; gx <= cx + 1; gx++) {
        const bucket = grid.get(`${gx}:${gy}`) || [];
        for (const a of bucket) if (a !== this && distance(a, this) < 50) neighbors.push(a);
      }
    }
    this.nearby = neighbors;
  }
  decide(aiOn) {
    if (!aiOn) return;
    if (this.nearby.length > 0) {
      let dx = 0, dy = 0;
      for (let a of this.nearby) { dx += this.x - a.x; dy += this.y - a.y; }
      this.vx += dx * 0.04 + rand(-0.6, 0.6);
      this.vy += dy * 0.04 + rand(-0.6, 0.6);
    } else {
      this.vx += rand(-0.4, 0.4);
      this.vy += rand(-0.4, 0.4);
    }
  }
  act() {
    this.x += this.vx; this.y += this.vy;
    this.vx *= 0.9; this.vy *= 0.9;
    this.energy -= 0.05;
    if (this.energy <= 0) this.active = false;
    this.x = clamp(this.x, 0, this.env.width);
    this.y = clamp(this.y, 0, this.env.height);
  }
}

class SwarmEngine {
  constructor(renderer, count = 1000) {
    this.renderer = renderer;
    this.agents = [];
    for (let i = 0; i < count; i++) {
      this.agents.push(new SwarmAgent(i, rand(0, renderer.width), rand(0, renderer.height)));
    }
    this.hwLevel = 1;
    this.aiOn = true;
    this.cellSize = 64; // spatial hashing cell size
    this.grid = new Map();
    this.metrics = { gridCells: 0 };
  }
  rebuildGrid() {
    this.grid.clear();
    for (const a of this.agents) {
      const gx = Math.floor(a.x / this.cellSize);
      const gy = Math.floor(a.y / this.cellSize);
      const key = `${gx}:${gy}`;
      const bucket = this.grid.get(key);
      if (bucket) bucket.push(a);
      else this.grid.set(key, [a]);
    }
    this.metrics.gridCells = this.grid.size;
  }
  update() {
    this.rebuildGrid();
    const env = this.renderer;
    for (let a of this.agents) if (a.active) a.perceive(this.agents, env, this.grid, this.cellSize);
    for (let a of this.agents) if (a.active) a.decide(this.aiOn);
    for (let a of this.agents) if (a.active) a.act();
    const before = this.agents.length;
    this.agents = this.agents.filter(a => a.active);
    const after = this.agents.length;
    if (after !== before) Bus.emit("swarm:prune", { before, after });
  }
  render() {
    const ctx = this.renderer.ctx;
    ctx.fillStyle = `rgba(0,0,0,${this.aiOn ? 0.05 : 0.2})`;
    ctx.fillRect(0, 0, this.renderer.width, this.renderer.height);
    // Batch draw for performance
    ctx.save();
    for (let a of this.agents) {
      ctx.fillStyle = `hsla(${(a.id * 10) % 360}, 100%, 50%, 1)`;
      ctx.fillRect(a.x, a.y, 3, 3);
    }
    ctx.restore();
  }
  spawnBurst(x, y, amt = 50) {
    const scale = clamp(this.hwLevel, 1, 8);
    const n = Math.round(amt * scale * 0.8);
    const baseId = this.agents.length ? Math.max(...this.agents.map(a => a.id)) + 1 : 0;
    for (let i = 0; i < n; i++) {
      const p = new SwarmAgent(baseId + i, x + rand(-10, 10), y + rand(-10, 10));
      this.agents.push(p);
    }
    Bus.emit("swarm:spawn", { n, x, y });
  }
  clear() {
    const n = this.agents.length;
    this.agents = [];
    Bus.warn("swarm:clear", { n });
  }
  upgradeHardware() { this.hwLevel++; Bus.emit("hw:upgrade", { hwLevel: this.hwLevel }); }
  toggleAI() { this.aiOn = !this.aiOn; Bus.emit("ai:toggle", { aiOn: this.aiOn }); }
}

/* =========================
   UI controller
========================= */
class UIController {
  constructor(engine) {
    this.engine = engine;
    this.upEl = document.getElementById("up-ms");
    this.fpsEl = document.getElementById("fps");
    this.hwEl = document.getElementById("hw");
    this.pCountEl = document.getElementById("particleCount");
    this.gridCellsEl = document.getElementById("gridCells");
    this.upBtn = document.getElementById("upgradeBtn");
    this.aiBtn = document.getElementById("aiBtn");
    this.spawnBtn = document.getElementById("spawnBtn");
    this.clearBtn = document.getElementById("clearBtn");
    this._wire();
  }
  _wire() {
    this.upBtn.addEventListener("click", () => {
      this.engine.upgradeHardware();
      this.hwEl.textContent = `Virtual HW: GOD MODE ${this.engine.hwLevel}`;
    });
    this.aiBtn.addEventListener("click", () => {
      this.engine.toggleAI();
      this.aiBtn.setAttribute("aria-pressed", String(this.engine.aiOn));
      this.aiBtn.textContent = `AI Substrate: ${this.engine.aiOn ? "ON" : "OFF"}`;
    });
    this.spawnBtn.addEventListener("click", () => {
      const cx = Math.round(this.engine.renderer.width / 2);
      const cy = Math.round(this.engine.renderer.height / 2);
      this.engine.spawnBurst(cx, cy, 400);
    });
    this.clearBtn.addEventListener("click", () => this.engine.clear());
    window.addEventListener("keydown", e => {
      if (e.key === "u") this.upBtn.click();
      if (e.key === "a") this.aiBtn.click();
      if (e.key === "c") this.clearBtn.click();
      if (e.key === " ") { e.preventDefault(); this.spawnBtn.click(); }
    }, { passive: false });
  }
  updateStats(stats) {
    this.upEl.textContent = stats.upMs.toFixed(0);
    this.fpsEl.textContent = `FPS: ${Math.round(stats.fps)}`;
    this.pCountEl.textContent = stats.particles;
    this.gridCellsEl.textContent = stats.gridCells;
  }
}

/* =========================
   File ingestor
========================= */
function FileIngestor(onItems) {
  const dz = el("div", { class: "dropzone" }, [
    el("p", { text: "Drop files here or use the picker." }),
    el("input", { type: "file", multiple: true }),
  ]);
  const input = dz.querySelector("input");
  const guessType = (type) => (type.includes("json") ? "json"
                      : type.includes("csv") ? "csv"
                      : type.includes("toml") ? "toml"
                      : (type.includes("yaml") || type.includes("yml")) ? "yaml"
                      : null);
  const readFile = (file) => new Promise((resolve) => {
    const reader = new FileReader();
    reader.onerror = () => resolve({ name: file.name, size: file.size, parser: "unknown", parsed: { __error__: String(reader.error) } });
    reader.onload = async () => {
      const text = String(reader.result || "");
      const ext = (file.name.split(".").pop() || "").toLowerCase();
      const byMime = guessType(file.type || "");
      const byExt = ["json","csv","toml","yaml","yml"].includes(ext) ? (ext === "yml" ? "yaml" : ext) : null;
      const parserKey = byMime || byExt || "json";
      let parsed;
      try {
        parsed = await Parsers[parserKey](text);
      } catch (e) {
        parsed = { __raw__: text, __error__: String(e) };
        Bus.error("parser:fail", e, { parserKey, name: file.name });
      }
      resolve({ name: file.name, size: file.size, parser: parserKey, parsed });
    };
    reader.readAsText(file);
  });
  const handleFiles = async (files) => {
    const results = [];
    for (const f of files) results.push(await readFile(f));
    onItems(results);
    Bus.emit("files:ingested", { count: results.length });
  };
  dz.addEventListener("dragover", (e) => { e.preventDefault(); dz.style.borderColor = "var(--primary)"; });
  dz.addEventListener("dragleave", () => { dz.style.borderColor = "var(--border)"; });
  dz.addEventListener("drop", (e) => { e.preventDefault(); dz.style.borderColor = "var(--border)"; handleFiles(e.dataTransfer.files); });
  input.addEventListener("change", () => handleFiles(input.files));
  return dz;
}

/* =========================
   Components
========================= */
function Nav({ route }) {
  const links = [
    ["/", "Home"], ["/files", "Files"], ["/models", "AI Models"], ["/nodes", "Nodes"],
    ["/defi", "DeFi"], ["/dex", "DEX"], ["/p2p", "P2P"], ["/leverage", "Leverage"],
    ["/editor", "Editor"], ["/plugins", "Plugins"], ["/settings", "Settings"]
  ];
  return el("nav", { class: "nav" }, [
    el("strong", { class: "badge" }, "SuperOS"),
    ...links.map(([p,l]) => el("a", { href: "#" + p, class: route.path === p ? "active" : "" }, l))
  ]);
}

function Home() {
  return el("section", { class: "section" }, [
    el("h1", { text: "Welcome to SuperOS" }),
    el("p", { text: "Serverless, framework-free app scaffold for AI + blockchain + DeFi + P2P dashboards." }),
    el("div", { class: "grid" }, [
      el("div", { class: "card" }, [
        el("h3", { text: "Start with files" }),
        el("p", { text: "Ingest and translate configs or datasets." }),
        el("a", { href:"#/files", class:"button primary" }, "Open")
      ]),
      el("div", { class: "card" }, [
        el("h3", { text: "Adapters" }),
        el("p", { text: "Sandboxed plugins for domains." }),
        el("a", { href:"#/plugins", class:"button primary" }, "View")
      ]),
    ])
  ]);
}

function Files() {
  const list = el("div", { class: "grid" });
  const dz = FileIngestor(async (items) => {
    const translated = items.map(it => ({ ...it, translated: Translator.wrap(it.parsed) }));
    const prior = store.get().files || [];
    store.set({ files: [...prior, ...translated] });
  });
  const render = s => {
    list.innerHTML = "";
    (s.files || []).forEach(f => {
      const t = f.translated || {};
      list.appendChild(el("div", { class: "card" }, [
        el("div", { class: "row" }, [
          el("strong", { class: "pill" }, f.name),
          el("span", { class: "pill mono" }, `parser=${f.parser}`),
          el("span", { class: "pill mono" }, `size=${f.size}`)
        ]),
        el("p", { text: `Type: ${t.type}` }),
        el("details", {}, [
          el("summary", { text: "Preview (parsed)" }),
          el("pre", { class: "mono" }, JSON.stringify(f.parsed, null, 2))
        ]),
        el("details", {}, [
          el("summary", { text: "Preview (translated)" }),
          el("pre", { class: "mono" }, JSON.stringify(f.translated, null, 2))
        ])
      ]));
    });
  };
  store.subscribe(render);
  return el("section", { class: "section" }, [
    el("h1", { text: "Files" }),
    el("p", { text: "Drag-and-drop or pick files. Supported: JSON, CSV, TOML, YAML (subset)." }),
    dz, el("hr"), list
  ]);
}

function Models() {
  const list = el("div", { class: "grid" });
  const runBox = el("div", { class: "card" }, [
    el("h3", { text: "Run inference (stub)" }),
    el("textarea", { rows:"4", style:"width:100%;", placeholder:"Type a prompt..." }),
    el("div", { class: "row" }, [
      el("button", {
        class: "button primary",
        onclick: async () => {
          const wSpec = Registry.get("ai.local");
          if (!wSpec) return;
          const ta = runBox.querySelector("textarea");
          try {
            const res = await callWorker(wSpec.workerFactory(), { op:"run", args:{ prompt: ta.value } }, { timeoutMs: 5000 });
            ta.value = String(res.output || "");
          } catch (e) {
            Bus.error("ai:run:error", e);
            ta.value = `Error: ${String(e.message || e)}`;
          }
        }
      }, "Run")
    ])
  ]);
  const render = s => {
    list.innerHTML = "";
    (s.models || []).forEach(m => list.appendChild(el("div", { class: "card" }, [
      el("h3", { text: m.id }),
      el("p", { text: `Type: ${m.type}` })
    ])));
  };
  store.subscribe(render);
  return el("section", { class: "section" }, [ el("h1", { text: "AI Models" }), runBox, list ]);
}

function Nodes() {
  const list = el("div", { class: "grid" });
  const callBox = el("div", { class: "card" }, [
    el("h3", { text: "RPC call" }),
    el("div", { class: "kv" }, [
      el("label", { text: "RPC URL" }),    el("input", { value:"http://localhost:8545" }),
      el("label", { text: "Method" }),     el("input", { value:"eth_blockNumber" }),
      el("label", { text: "Params (JSON)" }), el("input", { value:"[]" }),
    ]),
    el("div", { class: "row" }, [
      el("button", {
        class: "button primary",
        onclick: async () => {
          const [rpcEl, methodEl, paramsEl] = callBox.querySelectorAll("input");
          const wSpec = Registry.get("chain.eth");
          let args;
          try {
            args = { rpc: rpcEl.value, method: methodEl.value, params: JSON.parse(paramsEl.value || "[]") };
          } catch (e) {
            const out = callBox.querySelector("pre") || callBox.appendChild(el("pre", { class:"mono" }));
            out.textContent = `Params parse error: ${String(e)}`;
            return;
          }
          try {
            const res = await callWorker(wSpec.workerFactory(), { op:"call", args }, { timeoutMs: 8000 });
            const out = callBox.querySelector("pre") || callBox.appendChild(el("pre", { class:"mono" }));
            out.textContent = JSON.stringify(res, null, 2);
          } catch (e) {
            const out = callBox.querySelector("pre") || callBox.appendChild(el("pre", { class:"mono" }));
            out.textContent = `RPC error: ${String(e.message || e)}`;
            Bus.error("rpc:call:error", e, { method: args.method });
          }
        }
      }, "Send")
    ])
  ]);
  const render = s => {
    list.innerHTML = "";
    (s.nodes || []).forEach(n => list.appendChild(el("div", { class: "card" }, [
      el("h3", { text: n.id }), el("p", { text: n.rpc }), el("p", { text: n.status })
    ])));
  };
  store.subscribe(render);
  return el("section", { class: "section" }, [ el("h1", { text: "Nodes" }), callBox, el("hr"), list ]);
}

function Defi() {
  const box = el("div", { class: "card" }, [
    el("h3", { text: "AMM quote" }),
    el("div", { class: "kv" }, [
      el("span", { text: "x (base reserve)" }),   el("input", { value:"1000" }),
      el("span", { text: "y (quote reserve)" }),  el("input", { value:"2000000" }),
      el("span", { text: "dx (buy base)" }),      el("input", { value:"1" }),
      el("span", { text: "fee" }),                 el("input", { value:"0.003" }),
    ]),
    el("div", { class: "row" }, [
      el("button", {
        class: "button primary",
        onclick: async () => {
          const inputs = box.querySelectorAll("input");
          const args = { x:+inputs[0].value, y:+inputs[1].value, dx:+inputs[2].value, fee:+inputs[3].value };
          const wSpec = Registry.get("defi.amm");
          try {
            const res = await callWorker(wSpec.workerFactory(), { op:"quote", args }, { timeoutMs: 4000 });
            const out = box.querySelector("pre") || box.appendChild(el("pre", { class:"mono" }));
            out.textContent = JSON.stringify(res, null, 2);
          } catch (e) {
            const out = box.querySelector("pre") || box.appendChild(el("pre", { class:"mono" }));
            out.textContent = `Quote error: ${String(e.message || e)}`;
            Bus.error("defi:quote:error", e);
          }
        }
      }, "Quote")
    ])
  ]);
  return el("section", { class: "section" }, [ el("h1", { text: "DeFi" }), box ]);
}

function Dex() {
  const box = el("div", { class: "card" }, [
    el("h3", { text: "Orderbook" }),
    el("div", { class: "row" }, [
      el("button", {
        class: "button primary",
        onclick: async () => {
          const wSpec = Registry.get("dex.stub");
          try {
            const res = await callWorker(wSpec.workerFactory(), { op:"orderbook" }, { timeoutMs: 3000 });
            const out = box.querySelector("pre") || box.appendChild(el("pre", { class:"mono" }));
            out.textContent = JSON.stringify(res, null, 2);
          } catch (e) {
            const out = box.querySelector("pre") || box.appendChild(el("pre", { class:"mono" }));
            out.textContent = `Orderbook error: ${String(e.message || e)}`;
            Bus.error("dex:orderbook:error", e);
          }
        }
      }, "Load")
    ])
  ]);
  return el("section", { class: "section" }, [ el("h1", { text: "DEX" }), box ]);
}

function P2P() {
  const box = el("div", { class: "card" }, [
    el("h3", { text: "P2P peers" }),
    el("div", { class: "row" }, [
      el("button", {
        class: "button primary",
        onclick: async () => {
          const wSpec = Registry.get("p2p.webrtc");
          try {
            const res = await callWorker(wSpec.workerFactory(), { op:"peers" }, { timeoutMs: 3000 });
            const out = box.querySelector("pre") || box.appendChild(el("pre", { class:"mono" }));
            out.textContent = JSON.stringify(res, null, 2);
          } catch (e) {
            const out = box.querySelector("pre") || box.appendChild(el("pre", { class:"mono" }));
            out.textContent = `Peers error: ${String(e.message || e)}`;
            Bus.error("p2p:peers:error", e);
          }
        }
      }, "List")
    ])
  ]);
  return el("section", { class: "section" }, [ el("h1", { text: "P2P" }), box ]);
}

function Leverage() {
  const box = el("div", { class: "card" }, [
    el("h3", { text: "Perp pairs" }),
    el("div", { class: "row" }, [
      el("button", {
        class: "button primary",
        onclick: async () => {
          const wSpec = Registry.get("lev.perp");
          try {
            const res = await callWorker(wSpec.workerFactory(), { op:"pairs" }, { timeoutMs: 3000 });
            const out = box.querySelector("pre") || box.appendChild(el("pre", { class:"mono" }));
            out.textContent = JSON.stringify(res, null, 2);
          } catch (e) {
            const out = box.querySelector("pre") || box.appendChild(el("pre", { class:"mono" }));
            out.textContent = `Pairs error: ${String(e.message || e)}`;
            Bus.error("lev:pairs:error", e);
          }
        }
      }, "Load")
    ])
  ]);
  return el("section", { class: "section" }, [ el("h1", { text: "Leverage" }), box ]);
}

function Editor() {
  const box = el("div", { class: "card" }, [
    el("h3", { text: "Editor" }),
    el("textarea", { rows:"8", style:"width:100%;", placeholder:"Paste text..." }),
    el("div", { class: "row" }, [
      el("button", {
        class: "button primary",
        onclick: async () => {
          const wSpec = Registry.get("editor.basic");
          const ta = box.querySelector("textarea");
          try {
            const res = await callWorker(wSpec.workerFactory(), { op:"format", args:{ content: ta.value } }, { timeoutMs: 2000 });
            ta.value = String(res.content || ta.value);
          } catch (e) {
            Bus.error("editor:format:error", e);
          }
        }
      }, "Format"),
      el("button", {
        class: "button accent",
        onclick: async () => {
          const wSpec = Registry.get("editor.basic");
          const ta = box.querySelector("textarea");
          try {
            const res = await callWorker(wSpec.workerFactory(), { op:"export", args:{ content: ta.value } }, { timeoutMs: 2000 });
            if (res.blobUrl) {
              const a = el("a", { href: res.blobUrl, download: "export.txt", class:"inline" }, "Download");
              box.appendChild(a);
            }
          } catch (e) {
            Bus.error("editor:export:error", e);
          }
        }
      }, "Export")
    ])
  ]);
  return el("section", { class: "section" }, [ el("h1", { text: "Editor" }), box ]);
}

function Plugins() {
  const list = el("div", { class: "grid" });
  const render = () => {
    list.innerHTML = "";
    Registry.list().forEach(a => {
      list.appendChild(el("div", { class: "card" }, [
        el("h3", { text: a.title }),
        el("p", { text: `Capability: ${a.capability.id}` }),
        el("details", {}, [
          el("summary", { text: "Ops" }),
          el("ul", {}, a.capability.ops.map(op => el("li", {}, op)))
        ]),
        el("div", { class: "row" }, [
          el("button", {
            class: "button",
            onclick: async () => {
              try {
                const res = await callWorker(a.workerFactory(), { op: a.capability.ops[0] }, { timeoutMs: 4000 });
                const out = el("pre", { class: "mono" }, JSON.stringify(res, null, 2));
                list.appendChild(out);
              } catch (e) {
                const out = el("pre", { class: "mono" }, `Adapter error: ${String(e.message || e)}`);
                list.appendChild(out);
                Bus.error("adapter:test:error", e, { key: a.key });
              }
            }
          }, "Test")
        ])
      ]));
    });
  };
  render();
  return el("section", { class: "section" }, [ el("h1", { text: "Plugins" }), list ]);
}

function Settings() {
  const box = el("div", { class: "card" }, [
    el("h3", { text: "Settings" }),
    el("div", { class: "kv" }, [
      el("label", { text: "Dark mode" }),                el("input", { type:"checkbox", checked:true }),
      el("label", { text: "Persist state (localStorage)" }), el("input", { type:"checkbox", id:"persistState" }),
    ]),
    el("div", { class: "row" }, [
      el("button", {
        class: "button primary",
        onclick: () => {
          const persist = document.getElementById("persistState");
          if (!persist.checked) { alert("Persistence disabled"); return; }
          const s = store.get();
          try {
            localStorage.setItem("superos", JSON.stringify(s));
            alert("Settings saved");
            Bus.emit("settings:persist", { keys: Object.keys(s) });
          } catch (e) {
            alert("Persistence failed");
            Bus.error("settings:persist:error", e);
          }
        }
      }, "Save")
    ])
  ]);
  return el("section", { class: "section" }, [ el("h1", { text: "Settings" }), box ]);
}

/* =========================
   Bootstrap
========================= */
const header = document.getElementById("app-header");
const root = document.getElementById("app-root");

const routes = {
  "/": Home, "/files": Files, "/models": Models, "/nodes": Nodes,
  "/defi": Defi, "/dex": Dex, "/p2p": P2P, "/leverage": Leverage,
  "/editor": Editor, "/plugins": Plugins, "/settings": Settings
};

const router = Router({
  routes,
  onChange: (route) => {
    const navEl = header.querySelector(".nav");
    navEl.innerHTML = "";
    mount(navEl, Nav({ route }));
    root.innerHTML = "";
    const comp = routes[route.path] || Home;
    mount(root, comp());
    Bus.emit("route:change", { path: route.path });
  }
});

/* =========================
   Canvas + loop
========================= */
(function bootCanvas() {
  const canvas = document.getElementById("main-canvas");
  const ctx = canvas.getContext("2d", { alpha: true, desynchronized: true });
  function resize() { canvas.width = window.innerWidth; canvas.height = window.innerHeight; }
  window.addEventListener("resize", () => { resize(); renderer.width = canvas.width; renderer.height = canvas.height; }, { passive: true });
  resize();
  const renderer = { ctx, width: canvas.width, height: canvas.height };
  const engine = new SwarmEngine(renderer, 1500);
  const ui = new UIController(engine);

  let lastFrame = now();
  let fpsTracker = { frames: 0, last: now(), fps: 0 };

  // Interaction
  canvas.addEventListener("click", e => engine.spawnBurst(e.clientX, e.clientY, 200));
  canvas.addEventListener("touchstart", e => {
    const t = e.touches?.[0];
    if (t) engine.spawnBurst(t.clientX, t.clientY, 200);
  }, { passive: true });

  // Main loop
  function loop(ts) {
    const dt = ts - lastFrame;
    lastFrame = ts;
    fpsTracker.frames++;
    if (ts - fpsTracker.last >= 1000) {
      fpsTracker.fps = (fpsTracker.frames * 1000) / (ts - fpsTracker.last);
      fpsTracker.frames = 0;
      fpsTracker.last = ts;
    }
    engine.update();
    engine.render();
    ui.updateStats({ upMs: dt, fps: fpsTracker.fps, particles: engine.agents.length, gridCells: engine.metrics.gridCells });
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);

  // Expose engine for modules (optional)
  window.SuperOS = { engine, store, Bus, Registry, Capabilities };
})();

/* =========================
   State init + audit tap
========================= */
const persisted = (() => {
  try { return JSON.parse(localStorage.getItem("superos") || "null"); }
  catch { return null; }
})();
store.init(persisted || {
  files: [],
  models: [{ id: "llama.cpp", type: "local" }],
  nodes: [{ id: "geth", rpc: "http://localhost:8545", status: "offline" }]
});

Bus.subscribe(evt => {
  const iso = new Date(evt.ts).toISOString();
  // Structured console audit
  if (evt.level === "error") console.error("[AUDIT]", iso, evt.type, evt.payload);
  else if (evt.level === "warn") console.warn("[AUDIT]", iso, evt.type, evt.payload);
  else console.debug("[AUDIT]", iso, evt.type, evt.payload);
});

// Start the router
router.start();
</script>
</body>
</html>
