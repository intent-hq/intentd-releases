# intentd-releases

Intent daemon (`intentd`) update artifacts — beta/stable channel releases.

This repository hosts a public mirror of intentd release artifacts (mirrored
from the `intent-hq/intentd` releases) so the daemon can be installed and
auto-updated without access to the source repository:

- **`vX.Y.Z` releases** — platform archives (`intentd-<target>.tar.xz` /
  `.zip`) and their `.sha256` sidecars.
- **`channel-beta` / `channel-stable` releases** — machine-readable
  `beta.json` / `stable.json` manifests pointing at the latest release per
  channel. Download the manifest asset; do not consume the tags themselves.

No source code lives here. This mirror is temporary until intentd is
open-sourced, at which point artifacts will be served from the intentd
repository directly.
