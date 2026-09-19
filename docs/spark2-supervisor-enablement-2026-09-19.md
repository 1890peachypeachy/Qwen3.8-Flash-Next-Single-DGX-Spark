# Spark2 supervisor enablement + watchdog calibration — 2026-09-19

Fleet node: spark2 (spark-bdce, 100.71.248.116). Lane: `qwen3.8-flash-next-spark12`
@ :8000, container `vllm-fn-tp1`, image `vllm/vllm-openai:qwen38-flash-next`,
checkpoint `Mia-AiLab/Qwen3.8-Flash-Next-NVFP4`, max_model_len 262144.

## What shipped

- Upstream #41 (24/7 supervisor) + #51 merged (9032181), plus fork fix f8bf70a
  (heartbeat.service — upstream #41 shipped heartbeat.timer without its unit).
- systemd user units installed with path adaptation (`%h/Qwen3.8-Flash-Next-Single`,
  upstream units hardcode `%h/qwen38-flash-next`): supervisor + maintenance.timer +
  heartbeat.timer enabled, linger ON.
- Supervisor turned ON with Victor's go. Verified end state: smoke 8/8 PASS,
  supervisor steady (`launch_failures=0`, adopt/probe clean), watchdog armed.

## Incident ledger (own the mistakes)

The first enable attempt at 11:14 armed memwatch with kit-default floors
(MemAvailable < 6 GiB). Spark2 runs large watermark tunables
(min_free_kbytes=4MiB, wsf=300) under which warm MemAvailable reads ~0-2 GiB
while the host is healthy (warm steady state measured: MemFree 12.0 GiB,
MemAvailable 1.6 GiB). The 6 GiB avail floor fired at 11:17 — emergency-stopped
a perfectly healthy lane 3 minutes after "go". The old watchdog had been
running DISARMED (MIN_GIB=0) since the 2026-09-17 MemAvailable protective
shutdown precisely because of this accounting shift; arming defaults without
re-checking that calibration was the error.

The supervisor then relaunch-looped into the GB10 boot-memory wall: two weeks
of node uptime grew boot-phase overhead past the HOST_RESERVE_GIB=30 budget
(request_memory saw 88.5 vs 91.63 GiB desired). The documented remedy class
existed — the 2026-09-18 26->30 bump for the same failure — but was applied
late, after ~45 min of failed launches (11:17 stop -> 12:02 ready).
Recovery rule for next time: consult the lane's own remedy history FIRST when
a previously-working boot stops fitting.

## Final calibrated deltas (committed evidence, node .env)

- `HOST_RESERVE_GIB=36` (was 30; 26->30 on 09-18, 30->36 on 09-19). Cost:
  KV pool ~790K -> ~423K tokens at 262K ctx (still ~1.6x concurrency);
  warm MemFree went 4.6 (09-18, HR=30) -> 12.0 GiB (HR=36).
- Watchdog floors: `MEMWATCH_MIN_GIB=0` (avail branch OFF — unreadable under
  watermark shift), `MEMWATCH_MIN_FREE_GIB=2` armed (matches the 09-17/09-18
  incident signal), `MEMWATCH_FREE_GATE_GIB=10`.
- Supervisor restart required after .env edits: a long-running supervise.sh
  snapshotted .env at unit start and passes it down (environment wins over
  file). Not spelled out upstream; systemd restart re-reads cleanly.

## Verification (2026-09-19 ~12:05 PDT)

- Boot from supervisor relaunch: ready ~420s after launch.
- `scripts/smoke-test.sh`: 8 passed, 0 failed (health, 262144 ctx, correctness,
  determinism, 32.6 tok/s decode, tool-call round-trip, vision round-trip, metrics).
- `logs/supervisor.state`: launch_failures=0, adopt_since=1, last_probe_fail=0.
- Identity unchanged: `qwen3.8-flash-next-spark12` @ :8000, ctx 262144 —
  consumer configs untouched throughout.

## Standing notes

- The maintenance timer fires Sunday 04:00 (no catch-up stamp on first enable).
- ALERT_WEBHOOK is unset: alerts log to logs/alert.log only. Wire ntfy/bridge
  if the supervisor+breaker path should page anyone.
- Supervisor circuit breaker: 3 emergencies in 2h -> OPEN (alert-only until
  logs/supervisor.state is removed). launch_failures cap: 3, same recovery.
