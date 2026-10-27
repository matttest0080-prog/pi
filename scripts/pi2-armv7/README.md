# scripts/pi2-armv7

Install and verify the `pi` coding agent on a Raspberry Pi 2 Model B (ARMv7). The full write-up —
the decisive facts, the measured results, the acceptance criteria and the two optional code changes —
is in [`docs/armv7-pi2b.md`](../../docs/armv7-pi2b.md) — also available in
[English](../../docs/armv7-pi2b.en.md), [简体中文](../../docs/armv7-pi2b.zh-CN.md),
[日本語](../../docs/armv7-pi2b.ja.md), [한국어](../../docs/armv7-pi2b.ko.md) and
[Español](../../docs/armv7-pi2b.es.md).

| Script | Runs on | What it does |
| --- | --- | --- |
| `install-pi-on-pi2.sh` | the Pi2 | gates (arch `armv7l`, not root, a downloader present) → installs Node v22 armv7l into `~/opt` → installs the npm package into `~/.local` → verifies → measures. `--dry-run` changes nothing; `--check-only` inspects only. Fails closed. Pins `@earendil-works/pi-coding-agent@1.4.0` by default; override with `PI_PKG=<spec>` (last verified on real hardware: 2026-10-27, 1.4.0). |
| `pi2-pi-agent.sh` | the PC | `probe` (reachability + current state), `install` (fetches the Node tarball on the PC, scp's it over, runs the installer, keeps a log), `verify` (judges a log against the acceptance list), `all`. Set `HOST=user@host` for your board — it is **required**, there is no default address. Set `PI_PKG=@earendil-works/pi-coding-agent@<version>` to verify one specific published version: the driver forwards it to the installer instead of using the installer's pin. |
| `selftest.sh` | the PC | tests the tools themselves, including the paths that must be refused: running the installer on a non-ARMv7 host, `--dry-run` changing nothing, the probe not claiming success when the board is unreachable, and the verifier rejecting a log that shows a non-v22 Node, a stray prebuild, or a root run. |

Notes that matter on this board:

- **No `sudo`.** Everything installs under `$HOME`; the scripts never touch the system.
- **Pin Node to the v22 line.** v22 is the last major with official `linux-armv7l` builds (v23+ have none).
- **Do not assert "no `.node` anywhere".** The npm package bundles x64/arm64/darwin/win32 prebuilds by
  design; on ARMv7 they are simply never loaded. The invariant is "no `prebuilds/linux-arm/` and the CLI
  starts".
- `~/.local` is shared with anything else installed for the user there — remove only the pi pieces when
  rolling back.

## Release tags

Every fork release is an annotated tag on the synced `main` plus a GitHub release carrying the source
tree and `SHA256SUMS`. The last column is the npm package that was actually installed and verified on
the board at that tag — that is what `install-pi-on-pi2.sh` pins by default:

| Fork tag | Fork commit | Upstream tag | Upstream commit | Board-verified package |
| --- | --- | --- | --- | --- |
| `v1.0.4` | `ccfb4aa80` | `v1.0.4` | `7c10bd433` | 1.0.4 (2026-10-06) |
| `v1.3.0` | `b0d4b0f86` | `v1.3.0` | `71023aec1` | 1.3.0 (2026-10-20) |
| `v1.4.0` | `ebe43370c` | `v1.4.0` | `4a080252f` | 1.4.0 (2026-10-27) |

The fork reuses upstream's version numbers, so a local `v1.x.y` here is **the fork's** annotated tag —
its message records the upstream tag's commit. Upstream tags are therefore shadowed by name; fetch
upstream without tags (`git fetch upstream --no-tags`, or `git fetch upstream main`) so a later
`git fetch --tags` cannot overwrite them:

```sh
git fetch upstream main --no-tags
git merge upstream/main
git push origin main
```
