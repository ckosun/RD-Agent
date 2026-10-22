# Expression-factor mode

Phase 0 on this fork is documentation only. Phase 1 implementation is not in this tree.

| Phase | Status on this fork | Notes |
| --- | --- | --- |
| 0. Recon | Written | [00_recon.md](00_recon.md). Lineage pin, non-Docker `rdagent fin_factor` smoke, stock `v1.0.0` call chain, leakage notes, extension points. No feature code. |
| 1+ | Not started here | Grey is doing Phase 1 on a local checkout of tag `v0.8.0`. This fork stays at latest `main` so [00_recon.md](00_recon.md) can be used as the drift note against `v0.8.0`. Do not add expression-factor subclasses, prompts, runners, or loop changes on this branch unless asked. |

Grey Blue first asked for tag `v0.8.0`, then overrode the recon target to the latest default branch. This tree is that latest `main` (release `v1.0.0`, commit `484776c211e4fbbeef03e0ec00d6bbee7362a4f4`). Drift against `v0.8.0` (`274e274d5dbb72cc2ea139d1a7c93d73ce9b1198`) is in [00_recon.md](00_recon.md) section 6.

Do not treat names in the extension-point section as existing APIs. They are proposed subclass locations only.
