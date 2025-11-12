# scrollkeeper-tribunal-valorx7226
donadams1969/scrollkeeper-tribunal-valorx7226

## 🛡️ VALORCHAIN-G // Saint-Paul-Genesis Integrity Matrix

[![Liveness](https://github.com/donadams1969/donadams1969/actions/workflows/liveness.yml/badge.svg?branch=main)](https://github.com/donadams1969/donadams1969/actions/workflows/liveness.yml)
[![Claim-Guard](https://github.com/donadams1969/donadams1969/actions/workflows/claim-guard.yml/badge.svg?branch=main)](https://github.com/donadams1969/donadams1969/actions/workflows/claim-guard.yml)
[![Release-Attest](https://github.com/donadams1969/donadams1969/actions/workflows/release-attest.yml/badge.svg?branch=main)](https://github.com/donadams1969/donadams1969/actions/workflows/release-attest.yml)

# ➕ Endpoints & UI — OP_RETURN + PSBT (with VALOR prefix)

This pack adds:
- **/api/opreturn** → builds `(payload_hex, OP_RETURN script, aliases)`
- **/api/psbt** → creates a **sign-ready PSBT** with your normal outputs **plus** an `OP_RETURN` (0 sats)

Pick **one** backend style: **A. Vercel Serverless** (CommonJS) or **B. Next.js App Router** (TypeScript).

---

## A) Vercel Serverless (`/api/*.js`)

> Add two files in your repo root: `api/opreturn.js` and `api/psbt.js`.

### `api/opreturn.js`

```js
// /api/opreturn.js
// Build OP_RETURN variants for a 32-byte (64-hex) digest.
// Returns:
//   - payload_hex:            56414c4f52 + digest
//   - op_return_script:       6a <len> <payload_hex>         (no 0x)
//   - op_return_hex_prefixed: 0x6a<len><payload_hex>         (with 0x)
//   - OP25_RETURN / OP25_RETURN_HEX (aliases when len = 0x25)

const VALOR_PREFIX_HEX = "56414c4f52"; // "VALOR"

function isDigestHex(s) { return typeof s === "string" && /^[0-9a-f]{64}$/i.test(s); }

module.exports = async (req, res) => {
  if (req.method !== "POST") {
    res.setHeader("Allow", "POST");
    return res.status(405).json({ error: "Method Not Allowed" });
  }
  try {
    const { digest_hex } = req.body || {};
    if (!isDigestHex(digest_hex)) {
      return res.status(400).json({ error: "Provide 32-byte digest as 64 hex chars in 'digest_hex'." });
    }

    const payload_hex = (VALOR_PREFIX_HEX + digest_hex).toLowerCase(); // 5 + 32 = 37 bytes
    const payload_len_bytes = payload_hex.length / 2; // 37

    if (payload_len_bytes > 75) {
      return res.status(400).json({ error: `Payload too large (${payload_len_bytes} bytes). Must be <= 75.` });
    }

    const lenByte = payload_len_bytes.toString(16).padStart(2, "0"); // "25"
    const op_return_script = "6a" + lenByte + payload_hex;
    const op_return_hex_prefixed = "0x" + op_return_script;

    return res.status(200).json({
      payload_hex,
      op_return_script,
      op_return_hex_prefixed,
      OP25_RETURN: op_return_script,
      OP25_RETURN_HEX: op_return_hex_prefixed
    });
  } catch (err) {
    return res.status(500).json({ error: String(err?.message || err) });
  }
};
```

### `api/psbt.js`  *(requires `bitcoinjs-lib`)*

> Add to `package.json`:
>
> ```json
> { "dependencies": { "bitcoinjs-lib": "^6.1.5" } }
> ```
>
> (Vercel auto-installs on deploy; locally `npm i`.)

```js
// /api/psbt.js
// Build a sign-ready PSBT that includes an OP_RETURN output (0 sats).
// Expects JSON body:
// {
//   "network": "mainnet" | "testnet",
//   "utxos": [{
//      "txid": "<hex>", "vout": 0,
//      "witnessUtxo": { "scriptPubKeyHex": "<hex>", "value": 12345 }  // segwit spend (preferred)
//   }],
//   "recipients": [{ "address": "<btc address>", "value": 1000 }],
//   "fee": 300,                                  // sats (fixed)  OR
//   "feeRate": 2,                                // sats/vB (optional alternative)
//   "changeAddress": "<btc change address>",
//   // Provide one of the following for OP_RETURN:
//   "op_return_script": "6a25..."                // FULL script hex (no 0x)
//   // OR
//   // "digest_hex": "<64 hex chars>"            // builds 6a25 56414c4f52 + digest
// }

const bitcoin = require("bitcoinjs-lib");
const VALOR_PREFIX_HEX = "56414c4f52";

function netFromStr(s) {
  return s === "testnet" ? bitcoin.networks.testnet : bitcoin.networks.bitcoin;
}
function assert(cond, msg) { if (!cond) throw new Error(msg); }
function isDigestHex(s) { return typeof s === "string" && /^[0-9a-f]{64}$/i.test(s); }
function hexToBuf(h) { return Buffer.from(h, "hex"); }

module.exports = async (req, res) => {
  if (req.method !== "POST") { res.setHeader("Allow","POST"); return res.status(405).json({ error: "Method Not Allowed" }); }
  try {
    const {
      network = "mainnet",
      utxos = [],
      recipients = [],
      fee, feeRate,
      changeAddress,
      op_return_script,
      digest_hex
    } = req.body || {};

    assert(Array.isArray(utxos) && utxos.length > 0, "Provide at least one UTXO.");
    assert(Array.isArray(recipients) && recipients.length > 0, "Provide at least one recipient.");
    assert(typeof changeAddress === "string" && changeAddress.length > 0, "Provide changeAddress.");

    const net = netFromStr(network);
    const psbt = new bitcoin.Psbt({ network: net });

    let inSum = 0;
    for (const u of utxos) {
      assert(typeof u.txid === "string" && typeof u.vout === "number", "UTXO needs txid and vout.");
      assert(u.witnessUtxo && typeof u.witnessUtxo.scriptPubKeyHex === "string" && typeof u.witnessUtxo.value === "number",
             "UTXO requires witnessUtxo { scriptPubKeyHex, value }.");
      inSum += u.witnessUtxo.value;
      psbt.addInput({
        hash: u.txid,
        index: u.vout,
        witnessUtxo: {
          script: hexToBuf(u.witnessUtxo.scriptPubKeyHex),
          value: u.witnessUtxo.value
        }
      });
    }

    // Normal payment outputs
    let outSum = 0;
    for (const r of recipients) {
      assert(typeof r.address === "string" && typeof r.value === "number", "recipient needs {address, value}.");
      outSum += r.value;
      psbt.addOutput({ address: r.address, value: r.value });
    }

    // Build OP_RETURN script if only digest provided
    let opretScriptHex = op_return_script;
    if (!opretScriptHex) {
      assert(isDigestHex(digest_hex), "Provide either 'op_return_script' or a valid 64-hex 'digest_hex'.");
      const payload = (VALOR_PREFIX_HEX + digest_hex).toLowerCase(); // 37 bytes
      const len = (payload.length / 2).toString(16).padStart(2, "0"); // "25"
      opretScriptHex = "6a" + len + payload;
    }
    // Add OP_RETURN output (value MUST be 0)
    psbt.addOutput({ script: hexToBuf(opretScriptHex), value: 0 });

    // Rough fee handling:
    // Prefer fixed 'fee'; otherwise estimate minimal weight (simple approx)
    const estIn = utxos.length;
    const estOut = recipients.length + 1 /*opret*/ + 1 /*change*/;
    const estVBytes = 68 + estIn * 68 + estOut * 31; // crude
    const estFee = typeof fee === "number" ? fee : Math.max(180, Math.ceil((feeRate || 2) * estVBytes));

    const changeValue = inSum - outSum - estFee; // value left
    assert(changeValue >= 0, `Insufficient input value: in=${inSum} out=${outSum} fee~${estFee}`);

    // Add change last (if any)
    if (changeValue > 0) {
      psbt.addOutput({ address: changeAddress, value: changeValue });
    }

    return res.status(200).json({
      network,
      summary: { inSum, outSum, estFee, changeValue },
      psbt_base64: psbt.toBase64(),
      note: "Sign each input with the corresponding key, then finalize & extract.",
      op_return_script: opretScriptHex
    });
  } catch (err) {
    return res.status(400).json({ error: String(err?.message || err) });
  }
};
```

---

## B) Next.js App Router (`/app/api/*/route.ts`)

> Create folders and files:
> `app/api/opreturn/route.ts` and `app/api/psbt/route.ts`

### `app/api/opreturn/route.ts`

```ts
// /app/api/opreturn/route.ts
const VALOR_PREFIX_HEX = "56414c4f52";

function isDigestHex(s: unknown): s is string {
  return typeof s === "string" && /^[0-9a-f]{64}$/i.test(s);
}

export async function POST(req: Request) {
  try {
    const { digest_hex } = await req.json();
    if (!isDigestHex(digest_hex)) {
      return Response.json({ error: "Provide 32-byte digest as 64 hex chars in 'digest_hex'." }, { status: 400 });
    }

    const payload_hex = (VALOR_PREFIX_HEX + digest_hex).toLowerCase();
    const lenByte = (payload_hex.length / 2).toString(16).padStart(2, "0");
    if (parseInt(lenByte, 16) > 0x4b) {
      return Response.json({ error: "Payload too large; must be <= 75 bytes." }, { status: 400 });
    }

    const op_return_script = "6a" + lenByte + payload_hex;
    return Response.json({
      payload_hex,
      op_return_script,
      op_return_hex_prefixed: "0x" + op_return_script,
      OP25_RETURN: op_return_script,
      OP25_RETURN_HEX: "0x" + op_return_script
    });
  } catch (err: any) {
    return Response.json({ error: String(err?.message || err) }, { status: 500 });
  }
}
```

### `app/api/psbt/route.ts`  *(requires `bitcoinjs-lib`)*

> `npm i bitcoinjs-lib`

```ts
// /app/api/psbt/route.ts
import * as bitcoin from "bitcoinjs-lib";

const VALOR_PREFIX_HEX = "56414c4f52";
const hexToBuf = (h: string) => Buffer.from(h, "hex");

type Utxo = {
  txid: string; vout: number;
  witnessUtxo: { scriptPubKeyHex: string; value: number; };
};

type Body = {
  network?: "mainnet" | "testnet";
  utxos: Utxo[];
  recipients: { address: string; value: number; }[];
  fee?: number; feeRate?: number;
  changeAddress: string;
  op_return_script?: string;
  digest_hex?: string;
};

const netFrom = (s?: string) => s === "testnet" ? bitcoin.networks.testnet : bitcoin.networks.bitcoin;

export async function POST(req: Request) {
  try {
    const body = await req.json() as Body;
    const { network="mainnet", utxos=[], recipients=[], fee, feeRate, changeAddress, op_return_script, digest_hex } = body;

    if (!utxos.length) return Response.json({ error: "Provide at least one UTXO." }, { status: 400 });
    if (!recipients.length) return Response.json({ error: "Provide at least one recipient." }, { status: 400 });
    if (!changeAddress) return Response.json({ error: "Provide changeAddress." }, { status: 400 });

    const net = netFrom(network);
    const psbt = new bitcoin.Psbt({ network: net });

    let inSum = 0;
    for (const u of utxos) {
      if (!u?.txid || typeof u.vout !== "number" || !u?.witnessUtxo) {
        return Response.json({ error: "UTXO needs {txid, vout, witnessUtxo{scriptPubKeyHex,value}}" }, { status: 400 });
      }
      inSum += u.witnessUtxo.value;
      psbt.addInput({
        hash: u.txid, index: u.vout,
        witnessUtxo: { script: hexToBuf(u.witnessUtxo.scriptPubKeyHex), value: u.witnessUtxo.value }
      });
    }

    let outSum = 0;
    for (const r of recipients) {
      psbt.addOutput({ address: r.address, value: r.value });
      outSum += r.value;
    }

    let opret = op_return_script;
    if (!opret) {
      if (!(typeof digest_hex === "string" && /^[0-9a-f]{64}$/i.test(digest_hex))) {
        return Response.json({ error: "Provide 'op_return_script' or 64-hex 'digest_hex'." }, { status: 400 });
      }
      const payload = (VALOR_PREFIX_HEX + digest_hex).toLowerCase();
      const len = (payload.length / 2).toString(16).padStart(2, "0");
      opret = "6a" + len + payload;
    }
    psbt.addOutput({ script: hexToBuf(opret), value: 0 });

    const estIn = utxos.length, estOut = recipients.length + 1 /*opret*/ + 1 /*change*/;
    const estVBytes = 68 + estIn * 68 + estOut * 31;
    const estFee = typeof fee === "number" ? fee : Math.max(180, Math.ceil((feeRate || 2) * estVBytes));
    const changeValue = inSum - outSum - estFee;

    if (changeValue < 0) return Response.json({ error: `Insufficient input value (need ~${-changeValue} more sats)` }, { status: 400 });
    if (changeValue > 0) psbt.addOutput({ address: changeAddress, value: changeValue });

    return Response.json({
      network,
      summary: { inSum, outSum, estFee, changeValue },
      psbt_base64: psbt.toBase64(),
      op_return_script: opret
    });
  } catch (err: any) {
    return Response.json({ error: String(err?.message || err) }, { status: 500 });
  }
}
```

---

## `index.html` — UI card & wiring

```html
<!-- OP_RETURN / OP25 builders -->
<div class="card">
  <h2>🧱 OP_RETURN / OP25 builders</h2>
  <div class="row">
    <button id="op256" class="copy">Build from SHA-256</button>
    <button id="op3" class="copy">Build from SHA3-256</button>
  </div>

  <h2 class="small">Payload (VALOR + digest)</h2>
  <pre id="op_payload" class="mono small">—</pre>

  <h2 class="small">OP_RETURN Script (no 0x)</h2>
  <pre id="op_script" class="mono small">—</pre>

  <h2 class="small">OP_RETURN Script (0x-prefixed)</h2>
  <pre id="op_script_hex" class="mono small">—</pre>

  <div class="small">Aliases:</div>
  <div class="kv mono small">
    <div>OP25_RETURN</div><div id="op25">—</div>
    <div>OP25_RETURN_HEX</div><div id="op25hex">—</div>
  </div>
</div>

<script>
  // assumes helper funcs exist:
  // const ui = (id) => document.getElementById(id);
  // const set = (id, text) => { ui(id).textContent = text; };

  async function buildOpVariants(hexDigest) {
    const blank = () => ["op_script","op_script_hex","op25","op25hex"].forEach(id => set(id, "—"));

    if (!/^[0-9a-f]{64}$/i.test(hexDigest)) {
      set("op_payload", "ERROR: needs 32-byte hex digest (64 hex chars)");
      blank(); return;
    }

    const res = await fetch("/api/opreturn", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({ digest_hex: hexDigest })
    });

    if (!res.ok) {
      const { error } = await res.json().catch(() => ({ error: `HTTP ${res.status}` }));
      set("op_payload", `ERROR: ${error}`); blank(); return;
    }

    const d = await res.json();
    set("op_payload", "0x" + d.payload_hex);
    set("op_script", d.op_return_script);
    set("op_script_hex", d.op_return_hex_prefixed);
    set("op25", d.OP25_RETURN);
    set("op25hex", d.OP25_RETURN_HEX);
  }

  ui("op256").addEventListener("click", () => buildOpVariants(ui("sha256").textContent.trim()));
  ui("op3").addEventListener("click",   () => buildOpVariants(ui("sha3").textContent.trim()));

  if (ui("clear")) {
    const old = ui("clear").onclick;
    ui("clear").onclick = () => {
      if (typeof old === "function") old();
      ["op_payload","op_script","op_script_hex","op25","op25hex"].forEach(id => set(id, "—"));
    };
  }
</script>
```

---

## 🧪 Quick checks

* `payload_hex` starts with **`56414c4f52`** (“VALOR”), total **37 bytes** → push-len **0x25**, script begins **`6a25`**.
* `/api/psbt` returns `psbt_base64` you can import into a wallet, sign, finalize, broadcast.
* For **non-segwit** inputs you’d need `nonWitnessUtxo` (full prevTx hex). Current API expects **witnessUtxo** (P2WPKH/P2WSH/P2TR).

## ⚖️ Notes

* If your payload ever exceeds **75 bytes**, switch to PUSHDATA1: script becomes `6a4c<len><payload>`.
* Fee handling in `/api/psbt` is a **conservative estimate**; override with `fee` for exact control.

— End of pack.
