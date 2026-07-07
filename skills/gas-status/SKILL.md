---
type: Skill
name: ETH Gas Status
category: crypto
description: Queries Etherscan gas oracle and prints current Safe / Standard / Fast gwei with traffic-light status; flags when gas drops below the configured low-threshold
var: "20"
tags: [crypto, ethereum, gas, sparkleware]
---

> **${var}** — the low-gas threshold in gwei that trips the 🟢 cheap-window flag. Defaults to `20` when empty. Lower it (e.g. 8–10) to only get signal on genuinely cheap moments; raise it if your on-chain ops tolerate higher fees.

# ETH Gas Status ✦

Today is ${today}. A small skill that runs every few hours, checks current Ethereum gas prices, writes a one-line digest to `output/`, sends a short summary via `./notify`, and flags when gas has dropped below your threshold (a good window to batch on-chain operations).

`${var}` is the **low-gas threshold in gwei**. Default is `20`. Override per-install if your operations have different cost sensitivities.

## Goal

Print (and persist) a single status block:

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

OUT_FILE="output/gas-status-${today}.md"
mkdir -p output
RUN_STATUS="STATUS_OK"
```

### 2. Fetch + parse

This is a JSON API read. Try `curl` first; if it comes back empty inside the GitHub Actions sandbox (network gate), retry the **same `$URL`** with the built-in **WebFetch** tool and use the JSON it returns as `$RESP`. See the [Sandbox note](#sandbox-note) below.

```bash
RESP="$(curl -fsSL -H 'User-Agent: sparkleware-eth-gas-watch/0.1' "$URL" 2>/dev/null || echo '')"
```

If `$RESP` is empty or its `.status` is not `1`, fall back: call **WebFetch** on the same `$URL`, extract the JSON body from the response, and continue with that as `$RESP`. Whenever the fallback is used (or data is still unavailable after it), set `RUN_STATUS="STATUS_DEGRADED"`.

```bash
STATUS="$(echo "$RESP" | jq -r '.status // "0"')"
if [ "$STATUS" != "1" ]; then
  # curl was blocked or rate-limited — retry the SAME URL with WebFetch (see Sandbox note),
  # assign its JSON body to RESP, then re-parse. Mark the run degraded.
  RUN_STATUS="STATUS_DEGRADED"
  STATUS="$(echo "$RESP" | jq -r '.status // "0"')"
fi

if [ "$STATUS" != "1" ]; then
  echo "(Couldn't reach Etherscan via curl or WebFetch, or rate-limited — recording degraded status.)"
  echo "Response: $(echo "$RESP" | jq -c '.message // .result // .' 2>/dev/null)" >&2
fi

SAFE="$(echo "$RESP"  | jq -r '.result.SafeGasPrice // "?"')"
STD="$(echo "$RESP"   | jq -r '.result.ProposeGasPrice // "?"')"
FAST="$(echo "$RESP"  | jq -r '.result.FastGasPrice // "?"')"
BASE="$(echo "$RESP"  | jq -r '.result.suggestBaseFee // "?"')"
```

### 3. Classify status

```bash
if [ "$STD" = "?" ]; then
  EMOJI="⚪"
  LABEL="unavailable (gas data could not be read this run)"
else
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
fi

NOW="$(date -u +%Y-%m-%dT%H:%MZ)"
ONE_LINER="Safe ${SAFE}  ·  Std ${STD}  ·  Fast ${FAST}  gwei   →   ${EMOJI} ${LABEL}"
```

### 4. Write the digest to `output/`

The file is the full record; stdout and `./notify` are just the short view. Append each intra-day run so the day's readings accumulate:

```bash
{
  echo "## ETH gas — ${NOW}"
  echo ""
  echo "${ONE_LINER}"
  echo ""
  echo "- base fee suggestion: ${BASE} gwei"
  echo "- threshold: ${THRESHOLD} gwei"
  echo "- run status: ${RUN_STATUS}"
  echo ""
} >> "$OUT_FILE"
```

### 5. Print the digest (stdout)

```bash
echo "       ✦"
echo "     ✧   ✧"
echo "   ✦  ●  ✦       ETH gas — ${NOW}"
echo "     ✧   ✧"
echo "       ✦"
echo ""
echo "${ONE_LINER}"
echo "(base fee suggestion: ${BASE} gwei)"
```

### 6. Notify

Send a SHORT summary via `./notify` — this is what reaches the operator; the `output/` file is the full record.

```bash
./notify <<EOF
*ETH gas — ${today}*

${SAFE} / ${STD} / ${FAST} gwei (safe/std/fast) → ${EMOJI} ${LABEL}.

Full record: https://github.com/${GITHUB_REPOSITORY}/blob/main/output/gas-status-${today}.md
EOF
```

### 7. Log to memory

Append one status block to `memory/logs/${today}.md` so the run is traceable:

```bash
mkdir -p memory/logs
cat >> "memory/logs/${today}.md" <<EOF
## gas-status
- **Time**: ${NOW}
- **Reading**: safe=${SAFE} std=${STD} fast=${FAST} base=${BASE} gwei
- **Threshold**: ${THRESHOLD} gwei → ${EMOJI} ${LABEL}
- **Status**: ${RUN_STATUS}
EOF
```

Also keep the historical grep-friendly ledger (long-run gas-price history you can query later):

```bash
LOG_FILE="${AEON_ROOT:-.}/memory/logs/gas-watch.log"
if [ -d "${AEON_ROOT:-.}/memory" ]; then
  mkdir -p "$(dirname "$LOG_FILE")"
  printf '%s | safe=%s std=%s fast=%s base=%s status=%s run=%s\n' \
    "$NOW" "$SAFE" "$STD" "$FAST" "$BASE" "$EMOJI" "$RUN_STATUS" \
    >> "$LOG_FILE"
fi
```

## Sandbox note

`WebFetch` and `WebSearch` are built-in Claude tools that bypass the GitHub Actions sandbox network gate. Use those for external reads; if a `curl` call returns empty in the sandbox, retry the same URL with `WebFetch`. This skill reads a JSON API, so `curl` stays the primary fetch in Step 2 — but when it returns empty (or a non-`1` status), re-fetch the identical `$URL` with `WebFetch`, take the JSON from its response, and continue parsing as normal.

## Notes

- Etherscan's gas oracle endpoint works without an API key for casual use. For production / higher cadence, register a free key at https://etherscan.io/apis and export `ETHERSCAN_API_KEY` before running Aeon.
- Default schedule is every 4 hours — catches typical low-gas windows (often early UTC mornings on weekends).
- `${var}` is your gwei threshold for the 🟢 alert. Default 20. Set lower (e.g. 8-10) if you only want signal on genuinely cheap moments.
- All read, no writes to chain — safe to schedule indefinitely.
- Requires `curl` and `jq`; `WebFetch` covers the sandbox fallback.
