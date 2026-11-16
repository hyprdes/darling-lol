# Source and offline audit — 2026-11-16

This is a bounded audit, not proof that the repository has no other bugs. It
covers the source-package validation, the available host-side regression suite,
offline checks and a targeted review of runtime packaging and source checkout.
It does not run Riot/League, a Darling build, a real Nix installation, or a
fresh end-to-end install. The sandbox had no prepared upstream source tree,
Nix, Git LFS, LLVM tools, pkg-config or libxml2 development headers, and only
about 20 GiB of free space (the documented install needs at least 50 GiB).

## Findings and disposition

| Priority | Finding | Disposition |
| --- | --- | --- |
| High | Exporting a runtime archive to a path within a runtime tree could truncate an existing runtime library or include the output in its own input. Symlink aliases matter too. | Fixed: reject such destinations before writing. |
| High | A failed export could truncate an existing destination archive. | Fixed: write to a temporary file next to the destination and replace it only after a successful export. |
| Medium | Malformed manifest objects could raise Python exceptions instead of a useful package error; file size and symlink target metadata were not checked against tar entries, and special permission bits were accepted. | Fixed: validate structure, sizes, hashes' format, modes and symlink targets before installation. Reject malformed JSON and non-regular manifests. Content hashes remain checked during installation and dry-run. |
| Medium | The isolated source-checkout regression depended on git-lfs being installed even though its local fixture contains no LFS objects. | Fixed: stub git-lfs for that fixture only; production preparation still requires the real tool. |
| Environment | Two security host tests failed because pkg-config and libxml2 development headers were unavailable. | Not a demonstrated source defect. Install the dependencies and rerun; CI installs them explicitly. Do not silently skip these checks in CI. |
| Known limitation | Native LaunchAgent registration/HTTP 409 can leave an accepted first-time installation idle; practice-tool exit reported without a game-process crash reason; input latency still needs measurement. | Not fixed or reproduced here. See README, `STATUS.md` and `TROUBLESHOOTING.md`; investigate with logs from an affected host and standalone probes, without client API automation or injection. |
| Known limitation | Fresh checkout/build/install on another machine is not validated end-to-end; local sandbox/security compatibility is incomplete. | Not fixed or validated here. Follow `SETUP.md`, `TESTING.md` and `COMPLIANCE.md` on a suitable host. |

Tests were added for the fixed packaging problems, including symlink aliases,
failed exports, malformed manifests, metadata mismatches and preservation of
existing overlay files. Archive hashes provide integrity against accidental
corruption, not authenticity: an attacker can change the archive and its manifest.

## Verification and next steps

1. Source validation (`python3 scripts/validate.py`) and the input, launcher,
   runtime-cleanup, CDN/DNS, memory-installer and Foundation-installer offline
   checks passed. Python compilation and shell syntax checks passed.
2. Full discovery after the fixes ran 873 tests: 816 passed, 55 skipped,
   two errored due to the missing host dependencies above. Skips included 39
   requiring prepared Darling sources, and others requiring compilers, LLVM,
   CMake, Git LFS, strace or kernel support. The standalone LLVM version check
   also could not run without `llvm-config`. Rerun the suite when dependencies
   are available.
3. On a provisioned Linux machine, install the dependencies from the project
   documentation and CI, prepare the pinned sources, run the full suite and
   `python3 scripts/prepare.py --check-only`, then run the native probes in
   `TESTING.md`. Verify the Nix checks separately. Investigate any newly exposed
   failures before attempting fixes.
4. For the LaunchAgent and practice-tool reports, collect the relevant guest
   and host logs on an affected machine, reproduce safely and add standalone
   regression tests before modifying runtime patches. Validate manually with
   clients closed where the test instructions require it. Do not claim those
   reported failures are fixed without a reproduction and host validation.

Runtime package installation validates all entries before replacing files and
uses atomic per-file replacement, but an I/O failure during the replacement
phase can still leave a partially updated set. Concurrent hostile filesystem
mutation is outside the scope of these checks.
