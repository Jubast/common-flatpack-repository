# Common Flatpak Repository

A signed Flatpak repository for x86_64, built from the manifests in `apps/`
and published to GitHub Pages.

| App | App ID |
|-----|--------|
| Orca | `com.stablyai.orca` |
| Herdr | `dev.herdr.app` |
| xxHash (CLI example) | `io.github.Cyan4973.xxHash` |

## Quick start

The commands install system-wide. For a per-user install, add `--user` to
every `flatpak` command, e.g. `flatpak remote-add --user ...`.

### Add the remote

The apps use runtimes from Flathub, so add both remotes:

```sh
flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
flatpak remote-add --if-not-exists common-flatpak-repository https://jubast.github.io/common-flatpak-repository/index.flatpakrepo
```

The `.flatpakrepo` file carries the signing key, so every download is
verified; never use `--no-gpg-verify` or the bare `/repo` URL. Key
fingerprint: `38967A136E8DAF9FBC6C19649831A0DBE305B746` (public key in
`keys/repo-public.gpg.asc`).

### List the apps

```sh
flatpak remote-ls --app common-flatpak-repository
```

### Install and run

```sh
flatpak install common-flatpak-repository com.stablyai.orca
flatpak run com.stablyai.orca

flatpak install common-flatpak-repository dev.herdr.app
flatpak run dev.herdr.app

flatpak install common-flatpak-repository io.github.Cyan4973.xxHash
flatpak run io.github.Cyan4973.xxHash --help
```

Orca and Herdr get no access to your files by default. To let one open
projects in your home directory:

```sh
flatpak override --user --filesystem=home com.stablyai.orca
flatpak override --user --filesystem=home dev.herdr.app
```

### Update

```sh
flatpak update com.stablyai.orca   # one app
flatpak update                     # all apps and runtimes
```

### Refresh the remote

Refresh the app list and descriptions shown in software centers:

```sh
flatpak update --appstream common-flatpak-repository
```

If the repository URL changes, point the remote at the new one:

```sh
flatpak remote-modify --url=https://jubast.github.io/common-flatpak-repository/repo common-flatpak-repository
```

### Uninstall

```sh
flatpak uninstall com.stablyai.orca                 # keep its data in ~/.var/app
flatpak uninstall --delete-data com.stablyai.orca   # remove its data too
flatpak uninstall --unused                          # remove runtimes no app needs
flatpak remote-delete common-flatpak-repository
```

## Troubleshooting

**`GPG verification enabled, but no summary found`**: the remote points at an
old URL. `flatpak remotes -d` shows its name and URL. A remote named
`common-flatpack-repository` needs:

```sh
flatpak remote-modify --url=https://jubast.github.io/common-flatpak-repository/repo common-flatpack-repository
```

**Install fails on a missing `org.freedesktop.Platform` runtime**: add the
Flathub remote (see [Add the remote](#add-the-remote)).

## Maintaining the repository

On every push to `main` that touches `apps/` or the workflow,
`.github/workflows/build-and-publish.yml` builds every app, signs the repo
and publishes it to the `gh-pages` branch.

**Add an app**: create `apps/<app-id>/<app-id>.yml`, a flatpak-builder
manifest whose sources are pinned (a git tag or commit, or a sha256-pinned
release artifact), and add a build step for it to the workflow. Modules
shared by several apps live in `apps/shared/`. `io.github.Cyan4973.xxHash` is
a minimal CLI example; `com.stablyai.orca` packages a prebuilt binary via
`extra-data`.

**Setup**: add the Actions secrets `GPG_PRIVATE_KEY` (base64 of the private
key, `secrets.local/gpg-private-key.b64`) and `GPG_KEY_ID`
(the key fingerprint), and serve GitHub Pages from the root of `gh-pages`.

**Rotate the signing key**: replace both secrets and
`keys/repo-public.gpg.asc`; users then remove and re-add the remote.
