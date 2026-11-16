# Raspberry Pi 2B verification: published pi 1.5.2

Date: 2026-11-16. Board: Raspberry Pi 2 Model B, `armv7l`, Raspbian 13 (trixie), Node v22.23.2; no `sudo`.

## Isolated installer acceptance

Ran the updated `scripts/pi2-armv7/install-pi-on-pi2.sh` with a temporary `HOME` under `/tmp` and a pre-fetched official Node tarball. The board's existing global pi 1.5.0 and user files were not modified. Installer result: **11/11 acceptance checks passed**; `scripts/pi2-armv7/pi2-pi-agent.sh verify` independently accepted the captured output.

- npm 10.9.8 installed `@earendil-works/pi-coding-agent@1.5.2` (`added 123 packages in 2m`).
- `pi --version` → `1.5.2`; `pi --help` succeeded; Node's built-in `node:sqlite` loaded.
- No `linux-arm` (ARMv7) native prebuild was present; CLI startup succeeded.
- Measured by the installer: `pi --version` startup 4.60 s; Node baseline RSS 40 MiB; Node installation 183 MB; pi package 154 MB.
- The temporary install tree was removed; board disk free space returned to 1.2 GB.

Local script checks on the PC: `scripts/pi2-armv7/selftest.sh` passed **13/13**, and `bash -n` passed for all three ARMv7 shell scripts.

## Local model requests

In a separate isolated test using the published 1.5.2 package, a loopback llama.cpp router on the Pi served Qwen2.5-0.5B Q4 at 8K context. With `--no-tools --no-session`, `maxTokens: 64`, and a temporary `models.json`, both requests ended with `rc=0` and `stopReason: stop`: a cold request took 52 s and returned `OK.`, and a warm request took 10 s and returned `no reply.`. The configuration used a static OpenAI-compatible local endpoint, not llama.cpp dynamic model discovery. These results verify the local request/response path only—not tool execution, source builds, gateway behavior, or model answer quality. The temporary files and server were removed.

## Scope

This validates the published npm package and the isolated installer path on Pi 2B. It is not a source-build test. No tool call was made. The installed global pi remains v1.5.0.
