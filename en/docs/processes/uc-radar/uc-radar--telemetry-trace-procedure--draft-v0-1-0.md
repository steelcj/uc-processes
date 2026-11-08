---
dc:title: "UC Radar Telemetry Trace Procedure"
dcterms:version: "0.1.0"
dc:creator: "Christopher Steel"
dc:contributor: "Claude Opus 5.5 (Anthropic)"
dc:subject:
  - "radar"
  - "process"
  - "telemetry"
  - "network monitoring"
  - "evaluation method"
dc:description: "Working draft of the procedure for verifying the telemetry or content exposure gate by tracing a candidate's network behaviour with strace and OpenSnitch. In trial on the radar; to be formalized in uc-processes under uc-radar/telemetry/ once adopted."
dc:publisher: "UniversalCake"
dcterms:created: "2026-10-27"
dcterms:modified: "2026-10-27"
dc:type: "Text"
dc:format: "text/markdown"
dc:language: "en"
sat:language_bcp47: "en"
dc:source: ""
dc:relation: "universal-cake-evaluation-metrics, universal-cake-evaluation--opensnitch--application-firewall, universal-cake-evaluation--opentelemetry--observability-framework"
dc:identifier: "uc-radar--telemetry-trace-procedure"
dcterms:rightsHolder: "Christopher Steel"
dc:rights: >
  Copyright 2026 Christopher Steel / UniversalCake.
  SPDX-License-Identifier: AGPL-3.0-or-later
sat:uuid: ""
sat:version_at_creation: "0.5.1"
sat:migration_status: pre-sat
sat:changelog:
  - version: "0.1.0"
    date: "2026-10-27"
    author: "Claude Opus 5.5 (Anthropic), drafted for Christopher Steel's review"
    notes: >
      VERSION NUMBER PROVISIONAL, pending Christopher's confirmation.
      Working draft written after its first trial run, the OpenTelemetry
      re-score. Includes the fixes that run showed were needed.
---

# UC Radar Telemetry Trace Procedure

Version: 0.1.0
Status: Draft, in trial
Style Guide: style-guide--technical-documentation-for-technologists

## Purpose

The Universal Cake Evaluation Metrics make telemetry or content exposure that cannot be disabled a gate. A Verified rating on that gate means the candidate's outbound behaviour was observed, not read from its documentation or source. This procedure is one way to observe it. It records every network call a candidate makes, attributes each one to a process, and checks whether content the owner would not expect to share leaves in what the candidate does send.

## What it can and cannot see

It sees every outbound connection and datagram a traced process attempts, including DNS lookups, whether or not the connection succeeds. With the canary step, it also sees whether marked content appears in data the candidate exports to a destination the evaluator controls.

It does not see data written to disk and sent later by another process, unless the trace runs long enough to cover that, or that process is traced too. It does not see inside encrypted connections to destinations the evaluator does not control. It covers only the code paths the test exercised, so a feature that dials out only when used must be used during the trace. A clean trace is evidence for the configuration and actions traced, and the evaluation should say exactly which those were.

## Radar status of this procedure

This procedure is itself moving through the radar, and the record below is the worked example of that flow for `uc-processes`.

| Ring | Date | Record |
|---|---|---|
| Assess | 2026-10-27 | OpenSnitch evaluated as the instrument; its daemon traced under `strace`; recommended for Trial as evaluator tooling on a single-user machine |
| Trial, run 1 | 2026-10-27 | OpenTelemetry re-score: Collector core distribution and Java agent traced. Gate Verified; one content-exposure default found. Harness problems found and fixed, see Trial notes |
| Trial, run 2 | Pending | A different kind of candidate, preferably a desktop application with an interface and an update checker, so the procedure is not tuned to server software |
| Adopt | Pending | Human evaluation pass, after a clean second run |
| Formalize | Pending | Versioned process at `uc-processes/uc-radar/telemetry/`; the metrics' telemetry gate points to it |

## Requirements

- A disposable test host: an Incus container or virtual machine, or a cloud sandbox. Other processes on the host make traffic of their own, and a dedicated host keeps that noise small.
- `strace`.
- OpenSnitch 1.8.0 or later from the upstream release, not a distribution archive, which may be years behind. Configure it as an instrument: default action allow, debug logging, the UI socket in a per-user runtime directory or no UI at all, the version-check and VirusTotal plugins off. Use the iptables backend if the kernel lacks `CONFIG_NFT_QUEUE`; `opensnitchd -check-requirements` reports this.
- A local receiving end for whatever the candidate exports, where it has one, so its output can be inspected.
- The candidate's release artifacts, with their published checksums or signatures.

## Procedure

### Record and verify the candidate

Record the exact version of every component traced and the SHA-256 of each artifact. Verify the published signatures or checksums. Record the platform: distribution, kernel, runtime versions, OpenSnitch version and configuration.

### Start the instrument

Start the OpenSnitch daemon, then confirm its interception rule is in place, for example with `iptables -t mangle -S OUTPUT`.

### Run a negative control

Run a process that is known to reach the internet, such as `curl` to an outside site, under the same harness as the candidate. Both `strace` and OpenSnitch must record it. If either does not, stop: the instruments are not working, and a clean trace would mean nothing.

### Trace the candidate

Run each component in a clean environment, `env -i` with only `PATH`, `HOME` and the candidate's own settings. A clean environment matters for two reasons. Inherited proxy variables would redirect connections to the proxy and hide their real destinations. Inherited tokens could be picked up and exported, which would contaminate the content-exposure step.

Run each component under `strace -f -tt -e trace=connect,sendto,sendmsg,bind,listen -s 128 -o <file>`. Bound each run with `timeout`, and stop the candidate itself, not the `strace` process. Cover startup, normal use including every feature that might dial out, a quiet period, and shutdown. Configure the candidate as shipped, changing only what is needed to run it, and record any change.

### Test for content exposure

Where the candidate sends or exports data, run it once more with distinct markers placed where secrets usually live: a positional argument, a property or option named like a password, and an environment variable. Search the exported or transmitted data for each marker. If a marker appears, find the setting that removes it, rerun, and confirm it is gone. The gate turns on whether exposure can be disabled, so the rerun is what makes it Verified.

### Attribute everything

From the `strace` files, list every distinct destination of `connect`, `sendto` and `sendmsg`. From the OpenSnitch log, list every connection and the process ID it was attributed to. Identify every process that made an outside connection. A connection from a process that is not part of the candidate is noise, not evidence, and should be named as such in the evaluation.

### Stop and confirm cleanup

Stop the candidate, then OpenSnitch, and confirm OpenSnitch removed its rule.

### Record the result

In the evaluation's gate row: the result, the evidence tag Verified with the date, and what was traced, for how long, in what configuration. In the scope section: what was not traced. In the scorecard metadata: this procedure and its version as the assessment method.

## Interpreting the result

The gate passes when every outside connection the candidate made was one the owner configured, or can be disabled, and the disabling was verified. It fails when a connection or an exposure cannot be disabled. Local listeners are not outbound telemetry, but record them, since a listener on a public interface is a security concern. A connection the evaluator cannot explain is not a pass: record it and investigate before rating.

## Trial notes

### Run 1, OpenTelemetry, 2026-10-27

What worked: the negative control showed up in both instruments; OpenSnitch attributed every connection by process ID, using its `/proc` monitor; the canary test found a real default, command-line arguments in exported telemetry, and the rerun verified that it can be disabled.

What needed fixing, now reflected in the procedure above:

- Killing the `strace` process left the traced Collector running and the harness waiting. Bound runs with `timeout` and signal the candidate directly.
- `ss` was not installed on the test host. Read listeners from `/proc/net/tcp` or install `iproute2` as a requirement.
- `pkill -f` matched the harness's own shell. Track process IDs instead.
- The test host's own management processes made outside connections during the run. Without attribution they would have looked like the candidate's. This is why attribution is a step of its own.
- Inherited proxy variables would have redirected the candidate's connections. The clean environment is now a stated requirement.
- The Collector's cosign signatures and the Java agent's PGP signature were downloaded but not verified, because the tooling and keys were not set up. Verification is now the first step.

## Reference harness

The shell script below is the harness from run 1, with the fixes above applied. Adjust paths and the candidate commands per evaluation.

```bash
#!/bin/bash
# uc-radar telemetry trace harness, draft 0.1.0
set -u
T=${T:-$PWD}; L=$T/logs; mkdir -p "$L"
CLEAN=(env -i PATH=/usr/bin:/bin HOME=/root LANG=C.UTF-8)
ST=(strace -f -tt -qq -e trace=connect,sendto,sendmsg,bind,listen -s 128)

# record what is being traced
sha256sum "$T"/dl/* > "$L/artifacts.sha256"
uname -r > "$L/platform.txt"; cat /etc/os-release >> "$L/platform.txt"

# instrument
opensnitchd -config-file "$T/osn-etc/config.json" -rules-path "$T/osn-etc/rules" -debug \
  > "$L/opensnitchd.log" 2>&1 &
OSN=$!; sleep 4
iptables -t mangle -S OUTPUT > "$L/iptables-during.txt"

# negative control
"${CLEAN[@]}" "${ST[@]}" -o "$L/control.strace" curl -s -o /dev/null -m 10 https://example.com

# candidate: a long-running service, bounded by timeout
"${CLEAN[@]}" timeout -s TERM 420 "${ST[@]}" -o "$L/service.strace" \
  "$T/bin/otelcol" --config="$T/app/collector.yaml" > "$L/service.out" 2>&1 &
SVC=$!; sleep 5
cat /proc/net/tcp /proc/net/tcp6 > "$L/listeners.txt"

# candidate: an instrumented application, defaults except a service name
"${CLEAN[@]}" OTEL_SERVICE_NAME=uc-trial timeout -s TERM 180 "${ST[@]}" -o "$L/app.strace" \
  java -javaagent:"$T/dl/javaagent.jar" -cp "$T/app" Demo 90 > "$L/app.out" 2>&1

wait "$SVC"
kill -TERM "$OSN"; wait "$OSN"
iptables -t mangle -S OUTPUT > "$L/iptables-after.txt"

# summary: distinct destinations per traced component
for f in "$L"/*.strace; do
  echo "== $f"
  grep -oE '(connect|bind|sendto|sendmsg)\([0-9]+, \{sa_family=AF_[A-Z0-9]+, (sun_path="[^"]*"|sin6?_port=htons\([0-9]+\), sin6?_addr=[^)]*\))' "$f" \
    | sed -E 's/\([0-9]+, /(/' | sort | uniq -c
done
```

## License

This document, *UC Radar Telemetry Trace Procedure*, by **Christopher Steel**, with AI assistance from **Claude Opus 5.5 (Anthropic)**, is licensed under the [GNU Affero General Public License v3.0 or later](https://www.gnu.org/licenses/agpl-3.0.html).

## Changelog

| Version | Status | Notes |
|---------|--------|-------|
| 0.1.0 | Draft, version provisional | Working draft after trial run 1, the OpenTelemetry re-score; includes the harness fixes that run showed were needed and the radar status record for the procedure itself |
