# libheif

Container images with the command-line tools of [libheif](https://github.com/strukturag/libheif), the library for HEIF images: `heif-dec` converts HEIC and AVIF files to JPEG, PNG, TIFF, WebP or Y4M, `heif-enc` encodes images as HEIC or AVIF, `heif-info` lists what a HEIF file contains, and `heif-thumbnailer` writes PNG thumbnails. `latest` also includes ExifTool, for editing or removing the metadata of the converted files. libheif and the HEVC decoder libde265 are compiled from their upstream releases on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when libheif publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the libheif project. Report problems with the image in this repository and libheif bugs [upstream](https://github.com/strukturag/libheif/issues).

## Quick start

Convert a HEIC photo to JPEG:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/libheif photo.heic photo.jpg
```

The entrypoint runs `heif-dec` under `tini` in `/work`, and without arguments it prints the help. `heif-dec` picks the output format from the file name: `.jpg` or `.jpeg`, `.png`, `.tif` or `.tiff`, `.webp` and `.y4m`. Without an output name it writes a JPEG next to the input. `-q` sets the JPEG or WebP quality (the defaults are 90 and 75). `heif-convert`, the name `heif-dec` had before libheif 1.18, is a link to it.

Convert every HEIC file in the current directory:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh ghcr.io/randomcontainers/libheif \
  -c 'for f in *.[hH][eE][iI][cC]; do heif-dec --quiet "$f" "${f%.*}.jpg"; done'
```

The other tools run with `--entrypoint`:

```sh
# Encode a PNG as AVIF; an output name ending in .heic gives HEIC
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint heif-enc \
  ghcr.io/randomcontainers/libheif -q 60 input.png -o output.avif

# List the images, their sizes and metadata in a HEIF file
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint heif-info \
  ghcr.io/randomcontainers/libheif photo.heic

# Write a PNG thumbnail at most 256 pixels wide and high
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint heif-thumbnailer \
  ghcr.io/randomcontainers/libheif -s 256 photo.heic thumbnail.png
```

`heif-enc` reads JPEG, PNG, TIFF, WebP, Y4M, HEIC and AVIF files, so it also turns HEIC into AVIF and AVIF into HEIC. It reads raw pixel data from `.raw` and `.img` files, or from any file with `--raw`. `--help` on each tool lists its options, and `heif-enc --list-encoders` and `heif-dec --list-decoders` show the codecs.

### Metadata and GPS positions

`heif-dec` turns the image upright, applying the rotation stored in the HEIF file, and copies the photo's EXIF and XMP metadata into JPEG, PNG and WebP output, GPS position included. TIFF output gets no metadata. The default image has ExifTool to remove or change it:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh ghcr.io/randomcontainers/libheif \
  -c 'heif-dec --quiet photo.heic photo.jpg && exiftool -overwrite_original "-gps*=" photo.jpg'
```

A GPS position can be stored in both the EXIF and the XMP metadata. `-gps:all=` only clears the EXIF GPS tags; `-gps*=` removes every tag whose name starts with GPS from both.

## What is in the image

| | slim | default |
|---|---|---|
| `heif-dec` (also as `heif-convert`), `heif-enc`, `heif-info`, `heif-thumbnailer` | yes | yes |
| ExifTool | no | yes |

ExifTool is the [randomcontainers/exiftool](https://github.com/randomcontainers/exiftool) build.

| Format | Decoder | Encoder |
|---|---|---|
| HEIC (HEVC) | libde265, compiled here | x265 |
| AVIF (AV1) | dav1d | aom |

The codecs are built into libheif, which loads no plugins. x265, dav1d and aom are the Ubuntu or Alpine packages. The Ubuntu and Alpine libde265 packages lack upstream security fixes, so libde265 is compiled here from its upstream release, like libheif. Images are written and read with libjpeg-turbo, libpng, libtiff and libwebp.

Not included: `heif-view`, the GdkPixbuf loader, the C headers, H.264 (AVC) and H.266 (VVC) images, JPEG and JPEG 2000 coded images inside HEIF files, and the ISO/IEC 23001-17 uncompressed format. The CMake options and the codec versions are in `/usr/local/share/randomcontainers/libheif/buildinfo`.

## Default or slim

The default image (`latest`) has the libheif tools and ExifTool and is recommended for general use: photos from phones carry EXIF and XMP metadata, often with a GPS position, which you may want to change or remove after converting them. Use `slim`, which has the libheif tools without ExifTool, when you only convert images, or as the base when you build your own image.

The default image is also published as `ghcr.io/randomcontainers/libheif-exiftool`, built in the [libheif-exiftool](https://github.com/randomcontainers/libheif-exiftool) repository with the same contents and a different digest.

## Tags

`<version>` is a libheif release such as `1.23.6`. `<minor>` and `<major>` are its shorter forms, `1.23` and `1`, and follow the newest release in that series.

| Default (with ExifTool) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current libheif version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

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

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new libheif release and are rebuilt when the base image changes. Switch to root to install more, for example GNU parallel to convert many files at once, then back:

```dockerfile
FROM ghcr.io/randomcontainers/libheif:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends parallel \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache parallel`. The entrypoint is `["tini", "--", "heif-dec"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new libheif releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/libheif/` holds the version, the source URLs, the build options, the license files and `runtime-deps`, the list of distro packages the tools need at run time. The [ImageMagick image](https://github.com/randomcontainers/imagemagick) has its own build of libheif and libde265 at the same paths, so build on one of the two images instead of copying one into the other.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/libheif:latest \
  --repo randomcontainers/libheif --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/libheif-exiftool` are built in that repository, so verify them with `--repo randomcontainers/libheif-exiftool` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/libheif:latest --format '{{ json .SBOM }}'
```

libheif and libde265 are compiled here, so they are not in the SBOM and image scanners do not check them. Their versions are in `/usr/local/share/randomcontainers/libheif/buildinfo`.

libheif and libde265 publish no signatures or checksum files. The SHA-256 values in `package.yml` are the digests GitHub records for the `libheif-<version>.tar.gz` and `libde265-<version>.tar.gz` release assets, and the build checks each archive against them before compiling.

## Updates

The project checks the releases of [strukturag/libheif](https://github.com/strukturag/libheif/releases) every 15 minutes. A release is picked up once it is 24 hours old. Its source tarball is checked against the digest GitHub records for the release asset, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new ExifTool image is published, and all of them at least every 7 days, so distro security fixes, including those for x265, dav1d and aom, reach the current tags.

libde265 is pinned to a version in `package.yml` and does not follow its upstream releases automatically. Updating it is a commit to `package.yml`, which rebuilds the images of the current libheif version.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  --build-arg LIBDE265_VERSION=<version> \
  --build-arg LIBDE265_SHA256=<sha256> \
  -t libheif:local .
```

The `LIBDE265_*` values are the `version` and `sha256` of the `libde265` entry under `extra-artifacts` in `package.yml`. Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci) and layers ExifTool onto this image.

## Licenses

libheif and libde265 are licensed under the GNU Lesser General Public License, version 3 or later (LGPL-3.0-or-later), and the `heif-*` tools under the MIT license. libheif is linked with x265, which is licensed under the GNU General Public License, version 2 or later, so the image's license label is `LGPL-3.0-or-later AND MIT AND GPL-2.0-or-later`. The license files are in `/usr/local/share/randomcontainers/libheif/licenses/`: `COPYING.libheif` and `COPYING.libde265`, and `COPYING.examples`, the MIT license with the copyright lines of the tools' source files. The other libraries from Ubuntu or Alpine keep their own licenses, for example BSD-2-Clause for aom and dav1d. The SBOM lists them.

The corresponding source for each image:

- libheif and libde265: every libheif version has a GitHub release in this repository, named `v<version>`, with the exact `libheif-<version>.tar.gz` and `libde265-<version>.tar.gz` that were compiled. The build applies no patches. `/usr/local/share/randomcontainers/libheif/source` lists that release and the download URLs of both archives. When libde265 is updated, the release also keeps the archive of the earlier version.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every CMake option.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

`latest` also contains ExifTool, which is mainly licensed under the same terms as Perl. Its license files and corresponding source are described in the [exiftool repository](https://github.com/randomcontainers/exiftool#licenses).

HEVC, the codec of HEIC files, and AV1 may be covered by patents in some countries. HEIC files are encoded with x265 and decoded with libde265, AVIF files encoded with aom and decoded with dav1d. Check what applies where you use them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
