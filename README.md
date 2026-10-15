# ugrep

Container images with [ugrep](https://ugrep.com/), a grep-compatible file search tool from [Genivia/ugrep](https://github.com/Genivia/ugrep). Besides regular expressions, ugrep searches with Boolean queries and fuzzy matching, looks inside archives and compressed files, and can index a directory tree so that later searches skip files that cannot match. ugrep is compiled from the release tag. The default image also contains Poppler, so that `ug+` and `ugrep+` search the text of PDF files. The images are rebuilt when ugrep publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by Genivia or the ugrep project. Report problems with the image in this repository and ugrep bugs at [Genivia/ugrep](https://github.com/Genivia/ugrep/issues).

## Quick start

Search the current directory and below for `TODO`, with line numbers:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ugrep -rn 'TODO' .
```

Always name the files or directories to search, `.` for the mounted directory. ugrep searches the working directory by default only when standard input is a terminal, and `docker run` without `-t` gives it none, so without a path it reads standard input and finds nothing. To search standard input, pass `-i` to Docker:

```sh
journalctl -b | docker run --rm -i ghcr.io/randomcontainers/ugrep -c 'error'
```

A few more commands:

```sh
# List the files that contain "invoice", including files inside zip, tar and 7z archives
# and gzip, bzip2, xz, zstd, lz4, brotli and bzip3 files
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ugrep -z -rlw 'invoice' backups

# Lines with both "error" and "timeout", in any order, but not "retry"
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ugrep -rn --bool 'error timeout -retry' logs

# Fuzzy search: matches with up to two characters inserted, deleted or replaced
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ugrep -rn -Z2 'recieve' src

# Matches as JSON, for scripts
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ugrep -rn --json 'TODO' src > todo.json
```

Running the image without arguments prints ugrep's options. `ugrep --help <word>`, for example `--help fuzzy` or `--help regex`, prints the options and notes on one topic, and the [ugrep manual](https://ugrep.com/) has the rest.

## Searching PDF files

`ug+` and `ugrep+` are scripts that run `ug` with a filter for each document tool they find. In the default image that is Poppler's `pdftotext`, so they search the text of PDF files and the contents of all other files in one pass:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint ugrep+ \
  ghcr.io/randomcontainers/ugrep -rn 'net 30' .
```

You can also pass the filter to `ugrep` yourself:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ugrep \
  --filter='pdf:pdftotext % -' -rl 'net 30' .
```

Text that a PDF only holds as an image, such as a scan, has to go through OCR first; the [Tesseract](https://github.com/randomcontainers/tesseract) image does that. The slim image has no `pdftotext`, so there `ug+` and `ugrep+` work like `ug`. They also use antiword, pandoc and ExifTool when present, for Word, OpenDocument, EPUB and RTF files and for image metadata; none of these is in the images (see [Extending the slim image](#extending-the-slim-image)).

## Indexing

Indexing helps on a large tree, or on a slow one such as a network mount. `ugrep-indexer` stores hashes of the character sequences in each file, and `ugrep --index` skips the files whose hashes rule out a match. A search with `--index` still reads every file that changed or was added after indexing, so it finds the same lines as a search without it.

```sh
# Index the mounted directory, leaving out binary files
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint ugrep-indexer \
  ghcr.io/randomcontainers/ugrep -I

# Search with the index
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ugrep --index -I -rn 'TODO' .
```

`ugrep-indexer` writes a hidden `._UG#_Store` file into every directory it indexes, so the mount must be writable by the user the container runs as. Run it again to update the index, and with `-d` to delete the index files. Pass `-z` to both commands to index the contents of archives and compressed files.

## Interactive search

`-Q` opens ugrep's interactive search screen, which searches as you type. It needs a terminal, so pass `-it`:

```sh
docker run --rm -it --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/ugrep -Q
```

`ug` is ugrep for interactive use: it reads a `.ugrep` configuration file from the working directory, or from `HOME`, sorts files by name and, on a terminal, highlights its output. With the mount above, a `.ugrep` file in your directory sets your defaults; `ug --save-config` writes one with the options you pass it. Run it with `--entrypoint ug`.

## What is in the image

| Area | Details |
|---|---|
| Programs | `ugrep`, `ug`, `ugrep-indexer`, and the `ug+` and `ugrep+` scripts, in `/usr/local/bin` |
| Matching | ugrep's own regex engine with SSE2 and AVX2 on `linux/amd64` and NEON on `linux/arm64`, and PCRE2 with JIT compilation for `-P` (Perl syntax) |
| Archives (`-z`) | zip, 7z, tar, pax and cpio |
| Compressed files (`-z`) | gzip, compress (`.Z`), bzip2, xz and lzma, zstd, lz4, brotli and bzip3 |
| Pattern files | The predefined patterns that `-f` finds by name, such as `-f c/includes` or `-f python/defs`, in `/usr/local/share/ugrep/patterns` |
| Default image | Poppler (`pdftotext` and the other Poppler tools), for PDF files |

`ugrep --version` lists the SIMD level in use and the compression formats. On `linux/amd64`, ugrep picks AVX2 at run time on CPUs that have it, and SSE2 otherwise. The AVX-512 code is not built: configure decides whether to build it by running a test program on the build machine, which would make the result depend on the runner. Also not included: the shell completions, the manual pages and the filter tools other than `pdftotext`. The commit, the configure options and the SIMD flags are in `/usr/local/share/randomcontainers/ugrep/buildinfo`.

## Default or slim

Use the default image (`latest`) for general work. It adds Poppler, so `ug+` and `ugrep+` search the text of PDF files (see [Searching PDF files](#searching-pdf-files)). `slim` has ugrep and the libraries it needs, without Poppler. Use it to build your own image, or when you do not search PDF files.

The default image includes Poppler, which is licensed under the GNU General Public License (GPL-2.0-only OR GPL-3.0-only). Use `slim` if your policy excludes GPL software. The default image is also published as `ghcr.io/randomcontainers/ugrep-poppler`, built in the [ugrep-poppler](https://github.com/randomcontainers/ugrep-poppler) repository with the same contents and a different digest.

## Tags

`<version>` is a ugrep release such as `7.8.5`. `<minor>` and `<major>` are its shorter forms, `7.8` and `7`, and follow the newest release in that series.

| Default (with Poppler) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current ugrep version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

The same flag lets ugrep read files that only you can read. ugrep only reads the files it searches, so a read-only mount (`-v "$PWD:/work:ro"`) works for everything except `ugrep-indexer` and `--save-config`, which write index and configuration files.

## Untrusted files

ugrep unpacks archives and compressed files with code compiled into it (the 7z and `.Z` decoders and its zip, tar and cpio readers) and with the distro's compression libraries. `--zmax` sets how many levels of archives and compressed files inside archives it expands; the default is 1. For files from unknown sources, mount them read-only and take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  --network none --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 1g --pids-limit 64 \
  ghcr.io/randomcontainers/ugrep -z -rl 'password' .
```

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no Poppler, so Poppler updates do not rebuild it. The image runs as UID 1000, so switch to `USER root` to install distro packages, then back to `USER 1000:1000`. This one adds ExifTool, so that `ug+` also searches the metadata of JPEG, PNG, GIF and TIFF images and MPEG videos:

```dockerfile
FROM ghcr.io/randomcontainers/ugrep:slim-alpine@sha256:...
USER root
RUN apk add --no-cache exiftool
USER 1000:1000
ENTRYPOINT ["tini", "--", "ug+"]
```

On Ubuntu, start from `slim-ubuntu` and install `libimage-exiftool-perl`. The entrypoint of the image is `["tini", "--", "ugrep"]`. The packages ugrep needs are listed in `/usr/local/share/randomcontainers/ugrep/runtime-deps`. To pick up new ugrep releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/ugrep:latest \
  --repo randomcontainers/ugrep --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/ugrep-poppler` are built in that repository, so verify them with `--repo randomcontainers/ugrep-poppler` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/ugrep:latest --format '{{ json .SBOM }}'
```

ugrep is compiled here, so it is not in the SBOM and image scanners do not check it. The same goes for the code compiled into it: the RE/flex regex library, the 7z decoder from the LZMA SDK and the `.Z` decoder. Their fixes arrive with ugrep releases. The version and commit are in `/usr/local/share/randomcontainers/ugrep/buildinfo`.

ugrep attaches only a Windows build to its releases and does not sign its tags, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [Genivia/ugrep](https://github.com/Genivia/ugrep) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. It also runs ugrep's test suite and stops unless every compression library is compiled in.

## Updates

The project checks the releases of [Genivia/ugrep](https://github.com/Genivia/ugrep/releases) every 15 minutes. A release is picked up once it is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new Poppler image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t ugrep:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

ugrep and the RE/flex regex library compiled into it are licensed under the [BSD 3-Clause license](https://github.com/Genivia/ugrep/blob/master/LICENSE.txt). Three parts keep the terms of the code or data they come from:

- BSD-3-Clause: the `.Z` decoder in `src/zopen.c`, from the University of California.
- Public domain: the 7z decoder in `lzma/C`, a part of Igor Pavlov's LZMA SDK.
- Unicode-3.0: the Unicode tables in the RE/flex library, generated from the Unicode Character Database.

The license texts are in `/usr/local/share/randomcontainers/ugrep/licenses/`: `LICENSE.txt` (ugrep), `LICENSE.zopen`, `README.lzma-sdk` (the LZMA SDK's public domain statement) and `Unicode-3.0` (the Unicode License v3). Links to the source are in `/usr/local/share/randomcontainers/ugrep/source`. The image's license label is `BSD-3-Clause AND LicenseRef-Public-Domain AND Unicode-3.0`.

The default image adds Poppler, licensed under GPL-2.0-only OR GPL-3.0-only, with an MIT-licensed UTF-8 decoder; see [randomcontainers/poppler](https://github.com/randomcontainers/poppler#licenses) for its sources and license files. PCRE2, zlib, the other compression libraries and libstdc++ come from Ubuntu or Alpine and keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
