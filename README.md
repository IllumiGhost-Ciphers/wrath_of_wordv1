import logging
import json
import random
import time
import uuid
from datetime import datetime
from collections import deque

# ---------- Logging setup ----------
logger = logging.getLogger("long_term_poetic_syntax")
logger.setLevel(logging.INFO)
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(message)s"))
logger.addHandler(handler)

def log_event(event_type: str, payload: dict):
    entry = {
        "ts": datetime.utcnow().isoformat() + "Z",
        "id": str(uuid.uuid4()),
        "event": event_type,
        **payload
    }
    logger.info(json.dumps(entry, ensure_ascii=False))

# ---------- Configuration ----------
PORT_POOL = deque([2222, 8080, 443, 31337, 5000, 7001])
HONEY_BANNER = "mirror://pride>memory :: archive_unread :: elders_speaking"
MAX_DEPTH = 64

# System delay variables (mutable by consequence)
SYSTEM_DELAY = {
    "base_ms": 120,       # baseline delay
    "jitter_ms": 250,     # random jitter range
    "scale": 1.0,         # multiplicative scale influenced by consequence
    "silence_threshold": 0.85  # triggers protective silence
}

# ---------- State ----------
state = {
    "depth": 0,
    "consequence": 0.10,         # starts low; grows with ego-before-memory
    "archive_heard_ratio": 0.50, # probabilistic measure of elders heard
    "entropy": 0.15,             # system unpredictability
    "port_epoch": 0,             # counts rotations
    "sessions": 0,
    "failures": 0
}

# ---------- Honeypot (simulated) ----------
def honeypot_probe(port: int, depth: int, consequence: float):
    # Simulate an inbound probe with attributes
    probe = {
        "banner": HONEY_BANNER,
        "port": port,
        "depth": depth,
        "vector": random.choice(["mirror", "memory", "ego", "archive", "static"]),
        "amplitude": round(random.uniform(0.0, 1.0), 3),
        "cadence": random.choice(["rapid", "staggered", "flat"]),
    }
    # Interpret probe: ego-before-memory increases consequence
    ego_bias = 1.0 if probe["vector"] == "ego" else 0.0
    memory_bias = 1.0 if probe["vector"] == "memory" else 0.0
    delta = (ego_bias - memory_bias) * (0.05 + probe["amplitude"] * 0.1)
    new_consequence = max(0.0, min(1.0, consequence + delta))

    # Archive heard ratio trends inversely to consequence
    heard_delta = -delta * 0.6
    return probe, new_consequence, heard_delta

# ---------- Port switching (simulated) ----------
def switch_port():
    # Rotate the port pool
    PORT_POOL.rotate(-1)
    current = PORT_POOL[0]
    state["port_epoch"] += 1
    log_event("port_switch", {
        "port": current,
        "epoch": state["port_epoch"],
        "pool": list(PORT_POOL)
    })
    return current

# ---------- Sleep timer ----------
def adaptive_sleep():
    # Delay is base + random jitter, scaled by consequence and entropy
    base = SYSTEM_DELAY["base_ms"]
    jitter = random.uniform(0, SYSTEM_DELAY["jitter_ms"])
    scale = SYSTEM_DELAY["scale"] * (1.0 + state["consequence"] * 0.7 + state["entropy"] * 0.3)
    delay_ms = (base + jitter) * scale
    time.sleep(delay_ms / 1000.0)
    return round(delay_ms, 2)

# ---------- Consequence modulation ----------
def modulate_consequence(delta_consequence: float, heard_delta: float):
    # Update consequence
    before = state["consequence"]
    state["consequence"] = max(0.0, min(1.0, state["consequence"] + delta_consequence))
    # Update archive heard ratio
    state["archive_heard_ratio"] = max(0.0, min(1.0, state["archive_heard_ratio"] + heard_delta))
    # Update entropy slightly based on mismatch
    mismatch = abs(state["consequence"] - (1.0 - state["archive_heard_ratio"]))
    state["entropy"] = max(0.0, min(1.0, state["entropy"] * 0.9 + mismatch * 0.1))

    # Adjust system delay scale when consequence grows
    SYSTEM_DELAY["scale"] = 1.0 + state["consequence"] * 0.8

    log_event("consequence_update", {
        "consequence_before": round(before, 3),
        "consequence_after": round(state["consequence"], 3),
        "archive_heard_ratio": round(state["archive_heard_ratio"], 3),
        "entropy": round(state["entropy"], 3),
        "delay_scale": round(SYSTEM_DELAY["scale"], 3)
    })

# ---------- Protective silence ----------
def protective_silence():
    # Trigger when heard ratio falls below threshold (i.e., elders unheard)
    unheard = 1.0 - state["archive_heard_ratio"]
    if unheard >= SYSTEM_DELAY["silence_threshold"]:
        log_event("protective_silence", {
            "reason": "elders_unheard",
            "unheard_ratio": round(unheard, 3),
            "depth": state["depth"]
        })
        return True
    return False

# ---------- Recursive loop ----------
def cycle():
    if state["depth"] >= MAX_DEPTH:
        log_event("halt", {"reason": "max_depth", "depth": state["depth"]})
        return

    if protective_silence():
        log_event("halt", {"reason": "protective_silence", "depth": state["depth"]})
        return

    try:
        port = switch_port()
        probe, new_consequence, heard_delta = honeypot_probe(
            port=port,
            depth=state["depth"],
            consequence=state["consequence"]
        )

        log_event("honeypot_capture", {
            "port": probe["port"],
            "vector": probe["vector"],
            "amplitude": probe["amplitude"],
            "cadence": probe["cadence"],
            "banner": probe["banner"]
        })

        # Update consequence and ratios
        modulate_consequence(new_consequence - state["consequence"], heard_delta)

        # Sleep with adaptive delay
        slept_ms = adaptive_sleep()
        log_event("sleep", {
            "slept_ms": slept_ms,
            "depth": state["depth"],
            "scale": round(SYSTEM_DELAY["scale"], 3)
        })

        # Recurse
        state["depth"] += 1
        state["sessions"] += 1
        cycle()

    except Exception as e:
        state["failures"] += 1
        # On failure, increase entropy and delay scale slightly
        state["entropy"] = min(1.0, state["entropy"] + 0.05)
        SYSTEM_DELAY["scale"] = min(2.5, SYSTEM_DELAY["scale"] + 0.1)
        log_event("failure", {
            "error": str(e),
            "failures": state["failures"],
            "entropy": round(state["entropy"], 3),
            "delay_scale": round(SYSTEM_DELAY["scale"], 3)
        })
        # Backoff before retry
        slept_ms = adaptive_sleep()
        log_event("retry_backoff", {"slept_ms": slept_ms, "depth": state["depth"]})
        cycle()

if __name__ == "__main__":
    log_event("boot", {
        "message": "long_term_poetic_syntax starting",
        "max_depth": MAX_DEPTH,
        "ports": list(PORT_POOL)
    })
    cycle()
    log_event("shutdown", {
        "sessions": state["sessions"],
        "failures": state["failures"],
        "final_consequence": round(state["consequence"], 3),
        "final_archive_heard_ratio": round(state["archive_heard_ratio"], 3)
    })
