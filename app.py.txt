from flask import Flask, request, jsonify
from datetime import datetime
import threading, json, os

app = Flask(__name__)

# Credenciais Polarium via variáveis de ambiente
POLARIUM_EMAIL = os.getenv("POLARIUM_EMAIL")
POLARIUM_PASSWORD = os.getenv("POLARIUM_PASSWORD")

CANDLES = {}
CANDLES_LOCK = threading.Lock()
MAX_CANDLES = 3000

def normalize_pair(p):
    if not p: return "UNKNOWN"
    s = str(p).upper().replace("_","").replace("-","")
    if "/" in s: return s
    if len(s) == 6: return s[:3] + "/" + s[3:]
    return s

def try_extract_candles_from_payload(payload):
    out = []
    if isinstance(payload, str):
        try:
            payload = json.loads(payload)
        except:
            return out
    if isinstance(payload, dict):
        if "pair" in payload and "data" in payload:
            pair = normalize_pair(payload.get("pair"))
            for c in payload.get("data", []):
                candle = {
                    "ts": int(c.get("t") or c.get("time") or c.get("from") or 0),
                    "open": float(c.get("o") or c.get("open") or 0),
                    "high": float(c.get("h") or c.get("high") or 0),
                    "low": float(c.get("l") or c.get("low") or 0),
                    "close": float(c.get("c") or c.get("close") or 0)
                }
                out.append((pair, candle))
            return out
    return out

@app.route("/_ws_in", methods=["POST"])
def ws_in():
    body = request.json or {}
    payload = body.get("payload", body)
    items = try_extract_candles_from_payload(payload)
    count = 0
    with CANDLES_LOCK:
        for pair, candle in items:
            if pair not in CANDLES:
                CANDLES[pair] = []
            if not any(c['ts'] == candle['ts'] for c in CANDLES[pair]):
                CANDLES[pair].append(candle)
                CANDLES[pair].sort(key=lambda x: x['ts'])
                if len(CANDLES[pair]) > MAX_CANDLES:
                    CANDLES[pair] = CANDLES[pair][-MAX_CANDLES:]
                count += 1
    return jsonify({"stored": count})

def analyze_pair_candles(candles):
    wins=losses=signals=0
    last_signal=None
    for i in range(2, len(candles)):
        v1 = candles[i-2]; v2 = candles[i-1]; conf = candles[i]
        vela1Alta = v1['close'] > v1['open']; vela1Baixa = v1['close'] < v1['open']
        vela2Alta = v2['close'] > v2['open']; vela2Baixa = v2['close'] < v2['open']
        corpoMax = max(v1['open'], v1['close']); corpoMin = min(v1['open'], v1['close'])
        dentroVela1 = (v2['close'] <= corpoMax) and (v2['close'] >= corpoMin)
        condCALL = vela1Baixa and vela2Alta and dentroVela1
        condPUT  = vela1Alta  and vela2Baixa and dentroVela1
        if condCALL or condPUT:
            signals += 1
            if condCALL:
                last_signal = "CALL"
                if conf['close'] > conf['open']: wins += 1
                else: losses += 1
            else:
                last_signal = "PUT"
                if conf['close'] < conf['open']: wins += 1
                else: losses += 1
    return {"wins": wins, "losses": losses, "signals": signals, "last_signal": last_signal, "win_rate": (wins/signals) if signals else 0.0}

@app.route("/api/scan", methods=["POST"])
def api_scan():
    body = request.json or {}
    pairs = body.get("pairs")
    result = []
    with CANDLES_LOCK:
        target_pairs = pairs if pairs else list(CANDLES.keys())
        for p in target_pairs:
            pnorm = normalize_pair(p)
            candles = sorted(CANDLES.get(pnorm, []), key=lambda x: x['ts'])
            analysis = analyze_pair_candles(candles)
            result.append({"pair": pnorm, **analysis, "candles_count": len(candles)})
    return jsonify({"results": result})

@app.route("/api/status", methods=["GET"])
def status():
    with CANDLES_LOCK:
        return jsonify({"pairs_count": len(CANDLES), "pairs": list(CANDLES.keys())})

if __name__ == "__main__":
    print("Backend iniciado. Polarium credentials via env vars.")
    app.run(host="0.0.0.0", port=5000)
