# creusot-snap

A [Snap](https://snapcraft.io/) package for [Creusot](https://github.com/creusot-rs/creusot), a deductive verification tool for Rust. The snap bundles Creusot together with all its solver dependencies (Z3, cvc5, CVC4, Alt-Ergo) and the OCaml/Why3 toolchain so that users can install everything with a single command.

## What's Inside

The snap packages these components:

| Component | Version | Purpose |
|---|---|---|
| **Creusot** (`cargo-creusot`, `creusot-rustc`) | pinned commit `7010798` | Rust verification front-end |
| **Why3** (via opam) | OCaml 5.3.0 switch | Intermediate verification framework |
| **Z3** | 4.15.3 | SMT solver |
| **cvc5** | 1.3.1 | SMT solver |
| **CVC4** | 1.8 | SMT solver (legacy, still used by some Why3 strategies) |
| **Alt-Ergo** | 2.6.2 | SMT solver |

## Installing the Snap

```bash
# From a local .snap file
sudo snap install --classic --dangerous creusot_0.1_amd64.snap
```

> **Note:** The snap uses `classic` confinement because Creusot needs access to the user's Rust project files and toolchain.

### Setting Up the Alias

The snap is not yet published to the Snap Store, so the `cargo-creusot` alias is **not** created automatically. You must set it up manually after installing:

```bash
sudo snap alias creusot.cargo-creusot cargo-creusot
```

Without this alias, you would need to invoke the command as `creusot.cargo-creusot` instead of `cargo-creusot`.

## Usage

After installation and alias setup, `cargo-creusot` is available as a cargo subcommand:

```bash
# In a Creusot-enabled Rust project
cargo creusot prove
```

The snap sets up the required environment variables automatically:

| Variable | Value |
|---|---|
| `PATH` | `$SNAP/share/creusot/bin:$SNAP/bin:$PATH` |
| `XDG_DATA_HOME` | `$SNAP/share` |
| `CREUSOT_DATA_HOME` | `$SNAP/share/creusot` |
| `WHY3LIB` | `$SNAP/opam/5.3.0/lib/why3` |
| `WHY3DATA` | `$SNAP/opam/5.3.0/share/why3` |
| `WHY3CONFIG` | `$SNAP/share/creusot/why3.conf` |

## Building the Snap

### Prerequisites

- Ubuntu 24.04 (or compatible)
- [Snapcraft](https://snapcraft.io/docs/snapcraft-overview) installed
- Internet access (build downloads solver binaries and builds OCaml/Creusot from source)

### Build

```bash
cd creusot-snap
snapcraft
```

> **Warning:** The build takes a long time (~30–60 minutes) because it compiles the full OCaml toolchain, Why3, and Creusot from source.

The output is a `.snap` file in the project root (e.g., `creusot_0.1_amd64.snap`).

### Test locally

```bash
sudo snap install --classic --dangerous creusot_0.1_amd64.snap
sudo snap alias creusot.cargo-creusot cargo-creusot
cargo creusot prove   # in a Creusot project
sudo snap remove creusot
```

## CI / Release Workflow

The GitHub Actions workflow (`.github/workflows/release.yml`) automates building and releasing:

| Trigger | Action |
|---|---|
| Push to `master` | Build snap → upload artifact → upload to `latest` GitHub release |
| Push to `stable` | Build snap → upload artifact |
| GitHub Release published | Build snap → attach to the release |
| Manual `workflow_dispatch` | Build snap → upload artifact |

The workflow uses `snapcore/action-build@v1` to build and `softprops/action-gh-release@v1` to attach artifacts to releases.

## Updating Creusot Version

To update the version of Creusot packaged in the snap:

1. **Find the new commit hash** from the [Creusot repository](https://github.com/creusot-rs/creusot).

2. **Update `snap/snapcraft.yaml`:**
   ```yaml
   creusot:
     source-commit: <new-commit-hash>
   ```

3. **Check solver compatibility.** If the new Creusot version requires different solver versions, update the download URLs in the `z3`, `cvc5`, `cvc4`, and `altergo` parts.

4. **Check OCaml/opam compatibility.** If Creusot's `creusot-deps.opam` requires a newer OCaml version, update the `opam switch create` line.

5. **Build and test locally** before pushing.

6. **Push to `master`** to trigger the CI build, or create a GitHub Release for a tagged version.

## Project Structure

```
creusot-snap/
├── snap/
│   └── snapcraft.yaml       # Snap build recipe (all build logic lives here)
├── .github/
│   └── workflows/
│       └── release.yml       # CI: build → artifact → release
├── .gitignore
└── README.md
```

## Architecture Notes

The `snapcraft.yaml` defines six parts that are built in order:

1. **z3** — Downloads pre-built Z3 binary
2. **cvc5** — Downloads pre-built cvc5 binary
3. **cvc4** — Downloads pre-built CVC4 binary
4. **altergo** — Downloads pre-built Alt-Ergo binary
5. **opam** — Installs the opam package manager
6. **creusot** (after opam) — Initializes opam, installs OCaml 5.3.0, installs Why3 dependencies, builds Creusot from source, generates the prelude, and copies everything into the snap

The snap uses a `nil` plugin for solver downloads (just curl + copy) and the `rust` plugin for the Creusot build itself.

A notable build detail: the `sed` command in the creusot part patches a hardcoded `CARGO_MANIFEST_DIR` path to point to the snap's install location (`/snap/creusot/current/creusot/cargo-creusot`), which is required for `cargo creusot new` to find template files.

## Troubleshooting

- **Build fails downloading solvers:** Check that the download URLs in `snapcraft.yaml` are still valid. Solver projects occasionally reorganize their release assets.
- **Creusot build fails:** The pinned commit may have dependencies that have changed upstream. Check the Creusot repo for build instructions at that commit.
- **`cargo creusot` not found after install:** Make sure you ran `sudo snap alias creusot.cargo-creusot cargo-creusot`. Also verify the snap is installed with `--classic` confinement via `snap list creusot`.
- **Why3 errors at runtime:** The Why3 configuration is baked into the snap. If Creusot upstream changes its Why3 setup, the environment variables and config file paths in `snapcraft.yaml` may need updating.
