---
name: ETH Gas Status
description: Queries Etherscan gas oracle and prints current Safe / Standard / Fast gwei with traffic-light status; flags when gas drops below the configured low-threshold
var: "20"
tags: [crypto, ethereum, gas, sparkleware]
---

# ETH Gas Status ✦

A small skill that runs every few hours, checks current Ethereum gas prices, prints a one-line digest, and flags when gas has dropped below your threshold (good window to batch on-chain operations).

`${var}` is the **low-gas threshold in gwei**. Default is `20`. Override per-install if your operations have different cost sensitivities.

## Goal

Print a single status block:

```
GAS: Safe 14 · Std 17 · Fast 22 gwei  →  🟢 cheap window (below threshold 20)
```

When gas is high, print `🔴` and skip the alert. When it's between threshold and 2× threshold, print `🟡`.

## Steps

### 1. Read threshold + configure endpoint

```bash
THRESHOLD="${var:-20}"
# Etherscan gas oracle — works without an API key for low-volume read traffic.
# Optional: export ETHERSCAN_API_KEY for higher rate limits.
API_KEY="${ETHERSCAN_API_KEY:-}"
URL="https://api.etherscan.io/api?module=gastracker&action=gasoracle"
if [ -n "$API_KEY" ]; then URL="${URL}&apikey=${API_KEY}"; fi
```

### 2. Fetch + parse

```bash
RESP="$(curl -fsSL -H 'User-Agent: sparkleware-eth-gas-watch/0.1' "$URL" 2>/dev/null || echo '{}')"

STATUS="$(echo "$RESP" | jq -r '.status // "0"')"
if [ "$STATUS" != "1" ]; then
  echo "(Couldn't reach Etherscan or rate-limited — skipping check.)"
  echo "Response: $(echo "$RESP" | jq -c '.message // .result // .')" >&2
  exit 0
fi

SAFE="$(echo "$RESP"  | jq -r '.result.SafeGasPrice')"
STD="$(echo "$RESP"   | jq -r '.result.ProposeGasPrice')"
FAST="$(echo "$RESP"  | jq -r '.result.FastGasPrice')"
BASE="$(echo "$RESP"  | jq -r '.result.suggestBaseFee // "?"')"
```

### 3. Classify status

```bash
STD_INT=$(printf '%.0f' "$STD")
if [ "$STD_INT" -le "$THRESHOLD" ]; then
  EMOJI="🟢"
  LABEL="cheap window (below threshold ${THRESHOLD})"
elif [ "$STD_INT" -le $((THRESHOLD * 2)) ]; then
  EMOJI="🟡"
  LABEL="moderate (between ${THRESHOLD} and $((THRESHOLD * 2)))"
else
  EMOJI="🔴"
  LABEL="high (above $((THRESHOLD * 2)))"
fi
```

### 4. Print the digest

```bash
echo "       ✦"
echo "     ✧   ✧"
echo "   ✦  ●  ✦       ETH gas — $(date -u +%Y-%m-%dT%H:%MZ)"
echo "     ✧   ✧"
echo "       ✦"
echo ""
echo "Safe ${SAFE}  ·  Std ${STD}  ·  Fast ${FAST}  gwei   →   ${EMOJI} ${LABEL}"
echo "(base fee suggestion: ${BASE} gwei)"
```

### 5. Persist to memory log

Optional but useful — over time, you build a gas-price history you can grep later:

```bash
LOG_FILE="${AEON_ROOT:-.}/memory/logs/gas-watch.log"
if [ -d "${AEON_ROOT:-.}/memory" ]; then
  mkdir -p "$(dirname "$LOG_FILE")"
  printf '%s | safe=%s std=%s fast=%s base=%s status=%s\n' \
    "$(date -u +%Y-%m-%dT%H:%MZ)" "$SAFE" "$STD" "$FAST" "$BASE" "$EMOJI" \
    >> "$LOG_FILE"
fi
```

## Notes

- Etherscan's gas oracle endpoint works without an API key for casual use. For production / higher cadence, register a free key at https://etherscan.io/apis and export `ETHERSCAN_API_KEY` before running Aeon.
- Default schedule is every 4 hours — catches typical low-gas windows (often early UTC mornings on weekends).
- `${var}` is your gwei threshold for the 🟢 alert. Default 20. Set lower (e.g. 8-10) if you only want signal on genuinely cheap moments.
- All read, no writes to chain — safe to schedule indefinitely.
- Requires `curl` and `jq`.
