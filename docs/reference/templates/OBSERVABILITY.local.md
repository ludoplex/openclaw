# Local observability and native lobster execution template for OpenClaw

This template implements the local observability specification in `/reference/templates/OBSERVABILITY.local.spec`.

Use this template to fork OpenClaw and bootstrap a local environment with deep observability, native hooks, and continuous project execution.

This template is JavaScript-first and focuses on:

- Process lifecycle
- Terminal and shell execution
- File read, create, append, and delete activity
- Git commit activity
- Network sockets and packet captures
- Native hook execution paths
- User-defined project queues for 24x7 autonomous work
- Source declaration inventory (variables and functions with file paths and line numbers)
- Validation gates for coding and errands

## 1. Fork and clone

```bash
gh repo fork openclaw/openclaw --clone=true --remote=true
cd openclaw
git remote -v
```

## 2. Baseline environment

```bash
export OPENCLAW_HOME="$HOME/.openclaw"
export OPENCLAW_BIN="$(command -v openclaw)"
export OPENCLAW_LOG_ROOT="$HOME/openclaw-observability"
export OPENCLAW_CAPTURE_IFACE="any"
export OPENCLAW_MANIFEST_DIR="$OPENCLAW_LOG_ROOT/manifest"
mkdir -p "$OPENCLAW_LOG_ROOT"/{process,shell,files,git,network,hooks,instructions,projects,events,manifest,coverage}
pnpm install
```

## 3. Process telemetry

```bash
ps -eo pid,ppid,comm,args | rg 'openclaw|openclaw-gateway|node|bun' > "$OPENCLAW_LOG_ROOT/process/initial-snapshot.log"

nohup bash -lc '
  while true; do
    {
      echo "=== $(date -u +%Y-%m-%dT%H:%M:%SZ) ==="
      ps -eo pid,ppid,comm,args | rg "openclaw|openclaw-gateway|node|bun" || true
      echo
    }
    sleep 5
  done
' >> "$OPENCLAW_LOG_ROOT/process/poll.log" 2>&1 &
```

## 4. Shell and lobster-compatible terminal visibility

```bash
mkdir -p "$HOME/bin"
cat > "$HOME/bin/openclaw-observe" <<'WRAP'
#!/usr/bin/env bash
set -euo pipefail
LOG_ROOT="${OPENCLAW_LOG_ROOT:-$HOME/openclaw-observability}"
mkdir -p "$LOG_ROOT/shell"
export PROMPT_COMMAND='history -a'
export HISTTIMEFORMAT='%F %T '
script -q "$LOG_ROOT/shell/tty-$(date +%Y%m%d-%H%M%S).log" -c "openclaw $*"
WRAP
chmod +x "$HOME/bin/openclaw-observe"
```

## 5. File operation tracking

```bash
sudo auditctl -D
sudo auditctl -w "$OPENCLAW_HOME" -p rwxa -k openclaw-home
sudo auditctl -w "$PWD" -p rwxa -k openclaw-repo
sudo ausearch -k openclaw-home -ts recent > "$OPENCLAW_LOG_ROOT/files/audit-openclaw-home.log"
```

Fallback mode:

```bash
inotifywait -mr "$OPENCLAW_HOME" "$PWD" -e open,modify,create,close_write,attrib,delete \
  --format '%T %w%f %e' --timefmt '%Y-%m-%dT%H:%M:%SZ' \
  > "$OPENCLAW_LOG_ROOT/files/inotify.log"
```

## 6. Git commit telemetry

```bash
mkdir -p .githooks
cat > .githooks/post-commit <<'HOOK'
#!/usr/bin/env bash
set -euo pipefail
LOG_ROOT="${OPENCLAW_LOG_ROOT:-$HOME/openclaw-observability}"
mkdir -p "$LOG_ROOT/git"
{
  echo "timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
  echo "commit=$(git rev-parse HEAD)"
  git show --name-status --stat --oneline -1
  echo
} >> "$LOG_ROOT/git/commit-events.log"
HOOK
chmod +x .githooks/post-commit
git config core.hooksPath .githooks
```

## 7. Network and protocol visibility

```bash
sudo tcpdump -i "$OPENCLAW_CAPTURE_IFACE" -nn -s 0 -vvv \
  '(tcp or udp) and (port 80 or port 443 or port 5222 or port 3478 or port 5349)' \
  -w "$OPENCLAW_LOG_ROOT/network/openclaw-traffic.pcap"
```

```bash
while true; do
  {
    echo "=== $(date -u +%Y-%m-%dT%H:%M:%SZ) ==="
    ss -tpuna | rg 'openclaw|node|bun' || true
    echo
  } >> "$OPENCLAW_LOG_ROOT/network/socket-snapshots.log"
  sleep 10
done
```

## 8. Native hook instrumentation

```json
{
  "hooks": {
    "mappings": [
      {
        "match": {
          "channel": "*"
        },
        "action": "wake",
        "template": "[obs] channel={{Channel}} user={{UserName}} text={{Text}}"
      }
    ]
  }
}
```

```bash
cat > "$OPENCLAW_LOG_ROOT/hooks/hook-log.sh" <<'HOOKLOG'
#!/usr/bin/env bash
set -euo pipefail
LOG_ROOT="${OPENCLAW_LOG_ROOT:-$HOME/openclaw-observability}"
mkdir -p "$LOG_ROOT/hooks"
{
  echo "timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
  cat
  echo
} >> "$LOG_ROOT/hooks/events.log"
HOOKLOG
chmod +x "$OPENCLAW_LOG_ROOT/hooks/hook-log.sh"
```

## 9. User-defined project queue and autonomous 24x7 loop

Create a queue of projects/tasks defined by the user:

```bash
cat > "$OPENCLAW_LOG_ROOT/projects/queue.txt" <<'QUEUE'
project: implement spec requirement R1-R11 for local observability
project: validate issue triage artifacts and coverage checkpoints
project: run errands workflow and archive artifacts
QUEUE
```

Create instruction logger:

```bash
cat > "$OPENCLAW_LOG_ROOT/instructions/log-instruction.sh" <<'INSLOG'
#!/usr/bin/env bash
set -euo pipefail
LOG_ROOT="${OPENCLAW_LOG_ROOT:-$HOME/openclaw-observability}"
mkdir -p "$LOG_ROOT/instructions"
{
  echo "timestamp=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
  echo "instruction=$*"
  echo
} >> "$LOG_ROOT/instructions/instructions.log"
INSLOG
chmod +x "$OPENCLAW_LOG_ROOT/instructions/log-instruction.sh"
```

Create continuous worker:

```bash
cat > "$OPENCLAW_LOG_ROOT/projects/run-project-loop.sh" <<'WORKER'
#!/usr/bin/env bash
set -euo pipefail
LOG_ROOT="${OPENCLAW_LOG_ROOT:-$HOME/openclaw-observability}"
QUEUE_FILE="$LOG_ROOT/projects/queue.txt"
mkdir -p "$LOG_ROOT/projects"
touch "$QUEUE_FILE"

while true; do
  if [[ -s "$QUEUE_FILE" ]]; then
    mapfile -t jobs < "$QUEUE_FILE"
    : > "$QUEUE_FILE"
    for job in "${jobs[@]}"; do
      [[ -n "$job" ]] || continue
      "$LOG_ROOT/instructions/log-instruction.sh" "$job"
      openclaw agent --message "$job" --thinking low || true
    done
  fi
  sleep 15
done
WORKER
chmod +x "$OPENCLAW_LOG_ROOT/projects/run-project-loop.sh"
```

Phase policy:

- **Implementation and errands phase**: keep queue populated and process aggressively.
- **Maintenance-only phase**: reduce queue frequency once validation gates pass.

## 10. JavaScript declaration manifest

```bash
cat > "$OPENCLAW_MANIFEST_DIR/generate-js-declaration-manifest.sh" <<'MANIFEST'
#!/usr/bin/env bash
set -euo pipefail
ROOT="${1:-$(pwd)}"
OUT_DIR="${OPENCLAW_MANIFEST_DIR:-$HOME/openclaw-observability/manifest}"
OUT_FILE="$OUT_DIR/javascript-declarations.tsv"
mkdir -p "$OUT_DIR"
: > "$OUT_FILE"

rg -n --pcre2 '^(?:export\s+)?(?:async\s+)?function\s+([A-Za-z_$][\w$]*)|^(?:export\s+)?(?:const|let|var)\s+([A-Za-z_$][\w$]*)' \
  "$ROOT" \
  -g '*.js' -g '*.mjs' -g '*.cjs' \
  -g '!node_modules/**' -g '!dist/**' \
  -r '$file\t$line\t$0' >> "$OUT_FILE" || true

echo "wrote $OUT_FILE"
MANIFEST
chmod +x "$OPENCLAW_MANIFEST_DIR/generate-js-declaration-manifest.sh"
"$OPENCLAW_MANIFEST_DIR/generate-js-declaration-manifest.sh" "$(pwd)"
```

## 11. Validation and coverage gates

Code implementation gate:

```bash
pnpm lint | tee "$OPENCLAW_LOG_ROOT/coverage/lint.log"
pnpm build | tee "$OPENCLAW_LOG_ROOT/coverage/build.log"
pnpm test | tee "$OPENCLAW_LOG_ROOT/coverage/test.log"
```

Assistant-task-equivalent errands gate:

```bash
cat > "$OPENCLAW_LOG_ROOT/coverage/errands-gate.txt" <<'ERRANDS'
- required artifacts collected: process, shell, files, git, network, hooks, instructions
- queue workload completed for current cycle
- timeline regenerated
- safety checks reviewed
ERRANDS
```

## 12. Unified event timeline

```bash
cat > "$OPENCLAW_LOG_ROOT/events/build-timeline.sh" <<'TIMELINE'
#!/usr/bin/env bash
set -euo pipefail
LOG_ROOT="${OPENCLAW_LOG_ROOT:-$HOME/openclaw-observability}"
OUT="$LOG_ROOT/events/timeline.log"
: > "$OUT"
for f in "$LOG_ROOT"/process/*.log "$LOG_ROOT"/shell/*.log "$LOG_ROOT"/files/*.log "$LOG_ROOT"/git/*.log "$LOG_ROOT"/network/*.log "$LOG_ROOT"/hooks/*.log "$LOG_ROOT"/instructions/*.log "$LOG_ROOT"/coverage/*.log; do
  [ -f "$f" ] || continue
  sed "s#^#[${f##*/}] #" "$f" >> "$OUT"
done
sort "$OUT" -o "$OUT"
echo "wrote $OUT"
TIMELINE
chmod +x "$OPENCLAW_LOG_ROOT/events/build-timeline.sh"
```

## 13. Issue triage checklist

1. Process snapshots around the failure window
2. Shell transcript from `openclaw-observe`
3. File events for changed config/session/workspace files
4. Commit event log and patch summary
5. Socket snapshots and packet capture around failure time
6. Hook event logs and instruction logs
7. JavaScript declaration manifest output for affected modules
8. Validation and coverage gate logs
9. Unified `events/timeline.log`

## 14. Safety notes

- Keep observability outputs in private storage.
- Never commit raw secrets, credentials, or personal identifiers.
- Use least-privilege packet capture policies in sensitive environments.
