# rs-rich-marketplace

The plugin marketplace for [rs-rich](https://github.com/buchochelliq-labs/rs-rich-cli):
one JSON listing per package, and a signed index of them that `rich plugin`
reads. The format, the trust rules and the tiers are rs-rich-cli's
[marketplace design note](https://github.com/buchochelliq-labs/rs-rich-cli/blob/main/docs/design/marketplace.md);
this file says how to use the repository.

**Browse it on the web:** the docs site's
[Marketplace page](https://buchochelliq-labs.github.io/rs-rich-cli/latest/marketplace/)
searches and filters this index in the browser (by kind, tier and
recommended; official packages are marked), and links each package to its
install command and its reviews. It ships with rs-rich-cli 0.0.23; until
that release is out, main's docs have it at
[`/dev/marketplace/`](https://buchochelliq-labs.github.io/rs-rich-cli/dev/marketplace/).
The rest of the documentation is at
[buchochelliq-labs.github.io/rs-rich-cli](https://buchochelliq-labs.github.io/rs-rich-cli/).

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
| `artifacts/ID/ID-VERSION.EXT` | Files this repository hosts itself (the official WASM plugins, micro packs and themes). Each version since the themes is served as a release asset, `https://github.com/buchochelliq-labs/rs-rich-marketplace/releases/download/ID-vVERSION/ID-VERSION.EXT`, one release per version; the first ones keep their `https://raw.githubusercontent.com/buchochelliq-labs/rs-rich-marketplace/main/artifacts/...` URLs, since a listed version never changes. Never edited once listed. |
| `scripts/` | The index builder, the pull request's artifact check, `add_crate_version.py`, and the release, stats and discussion scripts the workflows run. |
| `schemas/stats.schema.json` | The schema of `stats.json` on the `stats` branch. |

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
   Users see the tiers as **Official**, **Approved** (`reviewed` in the
   listing) and **Community**.
5. **Recommended.** A listing may carry `"recommended": {"since":
   "YYYY-MM-DD", "note": "..."}`, shown as ★ Recommended. Adding,
   changing or removing it is refused until a maintainer labels the pull
   request `recommended-approved`, a separate label from `tier-approved`.
   (No listing carries it yet: the marketplace's validator,
   `RS_RICH_CLI_REF`, knows the field from rs-rich-cli 0.0.23.)
6. **Names.** Users can always tell rs-rich's own packages: only they
   are `official`, and a listing that is not may not pass for one. The
   check refuses, in a listing that is new or renamed:
   - a reserved id: `rs-rich…`, `rich-…`, `richcli…`, `official…`, `rich`
     or `rs` (separators ignored);
   - an id that looks like an official one: the same once `-`, `_` and `.`
     are dropped, or one letter added, removed or changed (`textcase2`,
     `text-case`, `wordp1ay`), for official ids of four characters or
     more;
   - an id that is not ASCII (`textcаse`, with a Cyrillic а), which the
     schema refuses too;
   - "official" or "rs-rich" in its name, or "official" in its
     description, in any case and with any separators.

   A genuine case ("an unofficial port of …", an id that only happens to
   be close) gets a maintainer's `name-approved` label on the pull
   request, which lifts this check for it.
7. **After the merge** the signing workflow rebuilds `index.json`, bumps
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
| `textcase` | wasm-plugin | rs-rich-cli's `crates/rich-plugin-api/examples/wasm/textcase.wat` (0.1.0) and `crates/rich-cli/official/wasm/textcase-VERSION.wat` (0.2.0 on) |
| `wordplay` | wasm-plugin | rs-rich-cli's `crates/rich-cli/official/wasm/wordplay-VERSION.wat` |
| `status`, `dev`, `fun` | micro-pack | rs-rich-cli's `crates/rich-micro/builtin/` |
| `dusk`, `paper`, `ocean`, `high-contrast`, `mono` | theme | rs-rich-cli's `crates/rich-cli/official/themes/ID.ini` |

The WASM plugins, the micro packs and the themes are built reproducibly in
rs-rich-cli, whose `tests/official_artifacts.rs` pins their hashes and
writes them:

```bash
RICH_OFFICIAL_ARTIFACTS=../rs-rich-marketplace/artifacts cargo test -p rs-rich-cli \
    --locked --test official_artifacts -- --ignored write_official_artifacts
```

A new version of an official WASM plugin, pack or theme is hosted here as
a release asset: write its listing with that URL, and commit the file as
`artifacts/ID/ID-VERSION.EXT` in the same pull request.

```bash
rich plugin publish artifacts/wordplay/wordplay-0.2.0.wasm \
  --url https://github.com/buchochelliq-labs/rs-rich-marketplace/releases/download/wordplay-v0.2.0/wordplay-0.2.0.wasm \
  --source-revision COMMIT --listing packages/wordplay.json
```

The pull request's check reads the file from the checkout (the release
does not exist yet), and requires the tag `ID-vVERSION` and the name
`ID-VERSION.EXT` to match the version. After the merge,
`publish-artifacts.yml` makes the release at the merge commit, uploads
the file and reads it back to check its size and SHA-256. It never
replaces an asset that is there. The signing workflow does not wait for
it: `rich` checks every download's SHA-256 against the signed listing, so
an install in the minutes before the asset exists fails cleanly ("not
found") and works once it is there.

A crate's new release is one command per crate (the version must be on
crates.io; its `.crate` is downloaded and its hash checked against crates.io's
index):

```bash
python3 scripts/add_crate_version.py packages/mermaid.json 0.0.9
python3 scripts/add_crate_version.py packages/dot.json 0.0.6
python3 scripts/add_crate_version.py packages/lumis.json 0.0.7
```

## Downloads, ratings and reviews

Every night (and by hand), `stats.yml` writes `stats.json` to the `stats`
branch:
`https://raw.githubusercontent.com/buchochelliq-labs/rs-rich-marketplace/stats/stats.json`.
Per package id it holds:

- **downloads**, per listed version and in total: a release asset's
  download count for a version hosted as a release, crates.io's count for
  a crate, and `null` (not counted) for a version on a raw URL;
- **reviews**: the package's discussion's URL, its 👍 and 👎 count and its
  comment count, or `null`.

`rich plugin info` shows them, and `rich plugin search --sort downloads`
sorts by them; so does the docs site. The counts are the ones GitHub and
crates.io keep anyway: `rich` sends nothing about who installs what, and
never phones home. The file is not signed and not in the index, so
nothing is installed or trusted because of it.

### Rating and reviewing a package

Each package has a discussion in this repository's
[Reviews](https://github.com/buchochelliq-labs/rs-rich-marketplace/discussions/categories/reviews)
category, titled with its id, opened by `discussions.yml` when the
listing is merged. To rate a package, react 👍 or 👎 to the discussion's
first post; to review it, comment. Only those two reactions on the first
post are counted, once a night.

## For maintainers

### One-time setup for stats and reviews

1. **Enable Discussions:** Settings > General > Features > Discussions.
2. **Create the "Reviews" category:** Discussions > Categories > New
   category, named exactly `Reviews`, with the **Announcement** format, so
   only maintainers and the workflow open discussions while anyone can
   react and comment. Neither step has an API.
3. **Run "Review discussions"** once by hand (Actions > Review discussions
   > Run workflow) to open a discussion for every package already listed.
   Until steps 1 and 2 are done, it fails and says what is missing.
4. **Run "Stats"** once by hand: it makes the `stats` branch. No
   protection is needed on it (main's ruleset covers only the default
   branch), and none of this needs the signing environment.
5. **Run "Publish artifacts"** once by hand if release-hosted versions were
   merged before this workflow existed.
6. **Create the `name-approved` label** (Issues > Labels), beside
   `tier-approved`: it lets a pull request through the name check above.
7. **Create the `recommended-approved` label** too: it lets a pull request
   add, change or remove a listing's `recommended`.

GitHub pauses scheduled workflows in a repository with no activity for 60
days; re-enable "Stats" under Actions if that happens.

### Day to day

- `python3 -m unittest discover -s scripts -p 'test_*.py'` tests the scripts.
- `python3 scripts/check_artifacts.py --all` checks every listed artifact.
- `python3 scripts/publish_releases.py --dry-run` (with a `GITHUB_TOKEN`)
  says which releases are missing.
- Signing, the protected `marketplace-signing` environment, the deploy key
  that pushes the signed index and key rotation are in rs-rich-cli's
  [`docs/marketplace-repo/README.md`](https://github.com/buchochelliq-labs/rs-rich-cli/blob/main/docs/marketplace-repo/README.md),
  which also keeps a copy of this repository's workflows and scripts.
