# Common Flatpak Repository

A self-hosted Flatpak repository, similar in spirit to Flathub: application
manifests live here, their source code lives in each app's own repository,
and a GitHub Actions workflow builds and publishes a signed Flatpak repo to
GitHub Pages on every change.

## Installing apps from this repository

Add the remote (once per machine):

    flatpak remote-add --if-not-exists <repo> https://<owner>.github.io/<repo>/index.flatpakrepo

Then install any app it publishes, for example the bundled `xxhsum` example:

    flatpak install <repo> io.github.Cyan4973.xxHash
    flatpak run io.github.Cyan4973.xxHash --help

The remote is GPG-signed; `flatpak remote-add` picks up the signing key
automatically from the `.flatpakrepo` file, so no `--no-gpg-verify` flag is
needed.

## Adding a new app

1. Create `apps/<app-id>/<app-id>.yml` — a standard flatpak-builder manifest.
   Its `sources` should point at the app's own git repository (pinned to a
   tag or commit), or at a sha256-pinned upstream release artifact when the
   app is redistributable as a prebuilt binary. Don't vendor the app's source
   into this repo.
2. Open a pull request. Once merged to `main`, the next CI run rebuilds and
   republishes the full repository, including the new app.

The example manifest at `apps/io.github.Cyan4973.xxHash/io.github.Cyan4973.xxHash.yml`
is a minimal reference: a CLI tool, no GUI toolkit, built against
`org.freedesktop.Platform`/`org.freedesktop.Sdk` only.

For a more involved case — a proprietary GUI app with no buildable source,
packaged from an upstream prebuilt binary — see
`apps/com.stablyai.orca/com.stablyai.orca.yml`. It fetches a `.deb` release
asset via Flatpak's `extra-data` mechanism, builds on
`org.electronjs.Electron2.BaseApp`, and includes a `--talk-name=org.freedesktop.Flatpak`
grant so the sandboxed app can shell out to host-installed CLIs — a
deliberate, broader permission that's worth reviewing before copying the
pattern elsewhere. The forwarding scripts live in `shared/host-bridge/` and are
reused by `apps/dev.herdr.app/dev.herdr.app.yml`, which installs a pinned
upstream tarball directly. Each app added under `apps/` also needs its own build
step added to `.github/workflows/build-and-publish.yml`, following the
existing per-app steps there.

## One-time repository setup (owner)

These steps are done once by whoever owns this GitHub repository — they
can't be automated from a sandbox without GitHub credentials, which is why
they're manual:

1. **Create the GitHub repository** (if it doesn't already exist) and push
   `main` to it.
2. **Add two repository secrets** under Settings → Secrets and variables →
   Actions:
   - `GPG_PRIVATE_KEY` — the contents of `secrets.local/gpg-private-key.b64`
     (already base64-encoded; paste as-is).
   - `GPG_KEY_ID` — `38967A136E8DAF9FBC6C19649831A0DBE305B746`
3. **Enable GitHub Pages**: Settings → Pages → Source = "Deploy from a
   branch", Branch = `gh-pages`, Folder = `/ (root)`. (The `gh-pages` branch
   doesn't exist yet — it's created by the workflow's first successful run,
   so this setting can be configured either before or after that first run.)
4. **Trigger the first build**: push a change under `apps/` (or anything
   that touches the workflow file), or run the workflow manually via the
   Actions tab ("Run workflow"). Confirm it succeeds, then check that
   `https://<owner>.github.io/<repo>/index.flatpakrepo` and
   `https://<owner>.github.io/<repo>/repo/summary` both load.
5. **Verify the signed install actually works** from a machine with
   `flatpak` installed, using the commands under "Installing apps from this
   repository" above — confirm the app installs and runs without needing
   `--no-gpg-verify`.

## Repository signing key

The repository is signed with a dedicated, passphrase-less GPG key
generated for this purpose. Its public key is committed at
`keys/repo-public.gpg.asc` for transparency; the private key lives only in
the `GPG_PRIVATE_KEY` GitHub Actions secret. To rotate the key: generate a
new keypair, replace both secrets, replace `keys/repo-public.gpg.asc`, and
note that existing users will need to re-run `flatpak remote-add` to pick
up the new key.
