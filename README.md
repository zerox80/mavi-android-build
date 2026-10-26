# Mavi Android build image

This repository owns the Docker image used to build Android APKs in
[zerox80/mavi](https://github.com/zerox80/mavi).

- Image: `ghcr.io/zerox80/mavi-android-build:latest`
- Host platform: Linux AMD64, including Docker inside WSL 2 on Windows.
- Android ABIs: ARMv7, ARM64, x86 and x86_64.
- Toolchain: JDK 21, SDK 37.1, Build Tools 36.0.0 and 37.0.0,
  NDK 28.1.13356709, Rust stable and cargo-ndk.

## Publishing

**Actions -> Publish Android build image** runs for pushes to `main` and can
also be started with **Run workflow**. Pull requests build and test the image
without publishing. The workflow publishes `latest` and an immutable source
tag `sha-<commit>` using `GITHUB_TOKEN`; it does not need registry secrets.

After the first publication, set the GHCR package visibility to **Public** so
the main repository, forks and local WSL builds can pull it without credentials.
Package settings: https://github.com/users/zerox80/packages/container/mavi-android-build/settings

The image contains public build tools and `build.py`, without VPN application
source, APKs, signing keys or server configuration. SDK licenses are accepted
while building the image.

## Building APKs

In a checkout of the application repository:

```powershell
.\Android-docker-build\build.ps1
```

Or from WSL/Linux:

```bash
bash Android-docker-build/build.sh assembleDebug
```

The application checkout provides the Compose file and WSL launcher. They
pull this image and mount the sources read-only at `/source`, the writable
lockfile at `/lock/Cargo.lock` and an artifact directory at `/output`.

The container copies source files into its Linux filesystem, refreshes h2
(`master`), h3 and Quinn (`main`) from the `zerox80` forks, and verifies their
remote branch heads before running Gradle. The updated lockfile is returned
to the checkout. No Rust or Java installation is required on Windows.

APKs and app bundles are checked for `libmavivpn.so` in all four ABIs before
export. Gradle, Cargo and the debug signing key use persistent Docker volumes
configured by the application repository.

## Maintaining the image

Keep SDK/NDK versions aligned with `android/app/build.gradle.kts` in `mavi`.
The command-line tools checksum comes from
[Android Developers](https://developer.android.com/studio#command-line-tools-only).
Rebuild without the Buildx cache when refreshing a cached Rust stable toolchain.
The image itself does not pin the h2/h3/Quinn revisions: each APK build refreshes
and checks those forks at runtime.

Unit tests use Python's standard library and never invoke Cargo:

```bash
python3 -m unittest discover -s tests -v
```

Local image builds, if needed, run through Docker in WSL:

```bash
docker build --pull -t ghcr.io/zerox80/mavi-android-build:local .
```
