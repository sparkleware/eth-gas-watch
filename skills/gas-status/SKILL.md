---
type: Skill
mode: read-only
name: ETH Gas Status
category: crypto
description: Queries Etherscan gas oracle and prints current Safe / Standard / Fast gwei with traffic-light status; flags when gas drops below the configured low-threshold
var: "20"
tags: [crypto, ethereum, gas, sparkleware]
requires: [ETHERSCAN_API_KEY?]
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
URL="https://api.etherscan.io/api?module=gastracker&action=gasoracle"

OUT_FILE="output/gas-status-${today}.md"
mkdir -p output
RUN_STATUS="STATUS_OK"
```

### 2. Fetch + parse

A JSON API read — keyless by default, so plain `curl` is the primary path. When `ETHERSCAN_API_KEY` is set (higher rate limits), route the call through `./secretcurl` with an `{ENV_NAME}` placeholder so the key never appears on a command line. See the [Network note](#network-note) below.

```bash
if [ -n "${ETHERSCAN_API_KEY:-}" ]; then
  RESP="$(./secretcurl "${URL}&apikey={ETHERSCAN_API_KEY}" 2>/dev/null || echo '')"
else
  RESP="$(curl -fsSL -H 'User-Agent: sparkleware-eth-gas-watch/0.1' "$URL" 2>/dev/null || echo '')"
fi
```

If `$RESP` is empty or its `.status` is not `1` (flaky GET or rate limit), fall back: call **WebFetch** on the same keyless `$URL`, extract the JSON body from the response, and continue with that as `$RESP`. Whenever the fallback is used (or data is still unavailable after it), set `RUN_STATUS="STATUS_DEGRADED"`.

```bash
STATUS="$(echo "$RESP" | jq -r '.status // "0"')"
if [ "$STATUS" != "1" ]; then
  # flaky GET or rate limit — retry the SAME keyless URL with WebFetch (see Network note),
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

### 3.5 Trend vs the last reading (from this skill's own ledger)

The historical ledger written in Step 7 doubles as a trend source — compare against the previous run so the digest says *where gas is heading*, not just where it is:

```bash
LEDGER="${AEON_ROOT:-.}/memory/logs/gas-watch.log"
TREND=""
if [ -f "$LEDGER" ] && [ "$STD" != "?" ]; then
  PREV_STD="$(tail -1 "$LEDGER" | sed -n 's/.* std=\([0-9.]*\) .*/\1/p')"
  if [ -n "$PREV_STD" ]; then
    DELTA=$(( $(printf '%.0f' "$STD") - $(printf '%.0f' "$PREV_STD") ))
    if   [ "$DELTA" -gt 0 ]; then TREND="↑ +${DELTA} gwei since last check"
    elif [ "$DELTA" -lt 0 ]; then TREND="↓ ${DELTA} gwei since last check"
    else                          TREND="→ flat since last check"
    fi
    ONE_LINER="${ONE_LINER}   (${TREND})"
  fi
fi
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

## Network note

There is no network sandbox — `curl` works normally. The keyless Etherscan GET is the primary path. When `ETHERSCAN_API_KEY` is set, the call goes through `./secretcurl` with an `{ETHERSCAN_API_KEY}` placeholder — never interpolate the key into a plain `curl` command line. If a GET returns empty or a non-`1` status (flaky or rate-limited), re-fetch the identical keyless `$URL` with the built-in **WebFetch** tool, take the JSON from its response, and continue parsing as normal.

## Notes

- Etherscan's gas oracle endpoint works without an API key for casual use. For production / higher cadence, register a free key at https://etherscan.io/apis and set `ETHERSCAN_API_KEY` as a repo secret.
- Default schedule is every 4 hours — catches typical low-gas windows (often early UTC mornings on weekends).
- `${var}` is your gwei threshold for the 🟢 alert. Default 20. Set lower (e.g. 8-10) if you only want signal on genuinely cheap moments.
- The Step-7 ledger accumulates a grep-able gas-price history; Step 3.5 reads it back for the trend line, so the skill's signal compounds the longer it runs.
- All read, no writes to chain — safe to schedule indefinitely.
- Requires `curl` and `jq`; `WebFetch` covers flaky public GETs.
