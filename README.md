# rs-rich-marketplace

The plugin marketplace for [rs-rich](https://github.com/buchochelliq-labs/rs-rich-cli):
one JSON listing per package, and a signed index of them that `rich plugin`
reads. The format, the trust rules and the tiers are rs-rich-cli's
[marketplace design note](https://github.com/buchochelliq-labs/rs-rich-cli/blob/main/docs/design/marketplace.md);
this file says how to use the repository.

```bash
rich plugin search                 # what is listed, and what loads on this host
rich plugin info textcase
rich plugin install textcase       # shows tier, permissions, capabilities; asks
```

## What is here

| Path | What |
|---|---|
| `packages/ID.json` | One listing per package: every version, its artifact's URL, SHA-256 and size, what it needs from the host, its capabilities and permissions. |
| `index.json`, `index.json.minisig` | The index: every listing's SHA-256, a serial and an expiry, signed with the marketplace's main key. Written by the signing workflow only. |
| `keys/keys-N.json` | The public keys, generation by generation. Generation 1 is compiled into `rich`. |
| `artifacts/ID/ID-VERSION.EXT` | Files this repository serves itself (the official WASM plugin and micro packs), from `https://raw.githubusercontent.com/buchochelliq-labs/rs-rich-marketplace/main/artifacts/...`. Never edited once listed. |
| `scripts/` | The index builder, the pull request's artifact check, and `add_crate_version.py`. |

Four kinds are listed: a `wasm-plugin`, a `micro-pack` and a `theme`
install with `rich plugin install`; a `crate` is listed only, and `install`
prints its `cargo add` line instead.

## Submitting a listing

1. **Host the file** where it will stay: a release asset of your
   repository, over `https://`. A listed version never changes, so neither
   may the file.
2. **Write the listing** with `rich plugin publish`, which checks the file
   as an install would (a WASM module is loaded in the sandbox and its
   manifest read; a micro pack is read with micro's limits; a theme must
   parse) and computes the hash:

   ```bash
   rich plugin publish shout.wasm \
     --url https://github.com/me/shout/releases/download/v0.2.0/shout.wasm \
     --source-revision "$(git rev-parse HEAD)" \
     --license MIT --repository https://github.com/me/shout \
     --output packages/shout.json
   ```

   A new version of a package already listed: `--listing packages/ID.json`.
3. **Open a pull request** with `packages/ID.json` only. Its checks:
   - every listing validates against the schema and the rules beyond it;
   - each new version's artifact is downloaded, and its size and SHA-256
     must match;
   - each new WASM plugin, micro pack and theme is installed with `rich`
     against the current host (a WASM module's name, version, ABI and
     capabilities must equal the listing's);
   - a version already listed may only gain `yanked` and `yank_reason`, and
     no listing or version is removed.
4. **Tiers.** A new listing is `community`. `reviewed` (a maintainer
   rebuilt it from `source_revision` and got the same hash) and `official`
   (rs-rich's own packages) are set by a maintainer: the check refuses a
   tier that is set or raised until a maintainer labels the pull request
   `tier-approved`.
5. **After the merge** the signing workflow rebuilds `index.json`, bumps
   its serial, and signs it once the maintainer approves the run. Until
   then the listing is in the repository but not in the index `rich` reads.

**Withdrawing a version:** set `"yanked": true` and a `yank_reason`; never
delete it, so installs that name it can still be verified.

## The official packages

| Id | Kind | Source |
|---|---|---|
| `mermaid` | crate | `rs-rich-mermaid` |
| `dot` | crate | `rs-rich-diagram`, feature `plugin` |
| `lumis` | crate | `rs-rich-lumis` |
| `textcase` | wasm-plugin | rs-rich-cli's `crates/rich-plugin-api/examples/wasm/textcase.wat` |
| `status`, `dev`, `fun` | micro-pack | rs-rich-cli's `crates/rich-micro/builtin/` |

The WASM plugin and the micro packs are built reproducibly in rs-rich-cli,
whose `tests/official_artifacts.rs` pins their hashes and writes them:

```bash
RICH_OFFICIAL_ARTIFACTS=../rs-rich-marketplace/artifacts cargo test -p rs-rich-cli \
    --locked --test official_artifacts -- --ignored write_official_artifacts
```

A crate's new release is one command per crate (the version must be on
crates.io; its `.crate` is downloaded and its hash checked against crates.io's
index):

```bash
python3 scripts/add_crate_version.py packages/mermaid.json 0.0.9
python3 scripts/add_crate_version.py packages/dot.json 0.0.6
python3 scripts/add_crate_version.py packages/lumis.json 0.0.7
```

## For maintainers

- `python3 -m unittest discover -s scripts -p 'test_*.py'` tests the scripts.
- `python3 scripts/check_artifacts.py --all` checks every listed artifact.
- Signing, the protected `marketplace-signing` environment, the deploy key
  that pushes the signed index and key rotation are in rs-rich-cli's
  [`docs/marketplace-repo/README.md`](https://github.com/buchochelliq-labs/rs-rich-cli/blob/main/docs/marketplace-repo/README.md),
  which also keeps a copy of this repository's workflows and scripts.
