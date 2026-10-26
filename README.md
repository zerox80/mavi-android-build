# Mavi Android Docker build

This repository owns the Docker definition used by
[zerox80/mavi](https://github.com/zerox80/mavi) to build Android APKs.

- `Dockerfile`: JDK 21, SDK 37.1, Build Tools 36.0.0 and 37.0.0,
  NDK 28.1.13356709, Rust stable and cargo-ndk.
- `build.py`: copies application sources into the container, refreshes the
  transport forks, runs Gradle and verifies/export artifacts.
- `tests/`: source staging, native-library and failure-path tests.
- Host platform: Linux AMD64, including Docker inside WSL 2 on Windows.
- Android ABIs: ARMv7, ARM64, x86 and x86_64.

## GitHub Actions

The **Build Android APK** workflow in the application repository:

1. Checks out `zerox80/mavi` and `zerox80/mavi-android-build` (`main`).
2. Builds this Dockerfile with Buildx and GitHub Actions caching.
3. Loads the image into the runner and builds the Android app inside it.
4. Uploads the APK as `android-apk` and the fork lockfile as `android-build-lockfile`.

Run it under
[Actions -> Build Android APK](https://github.com/zerox80/mavi/actions/workflows/android.yml).
It runs on pushes, pull requests and manual dispatch.

This repository's **Validate Android build image** workflow runs the unit tests
and Dockerfile checks for pushes and pull requests. Image and APK compilation
run in the application repository's workflow. No registry credentials are needed.

## Local builds via WSL

In the application checkout, from PowerShell:

```powershell
.\Android-docker-build\build.ps1
```

Or from WSL/Linux:

```bash
bash Android-docker-build/build.sh assembleDebug
```

The launcher clones this repository into a temporary Linux directory, builds
its image using Docker, then runs the app build. The temporary checkout is
removed afterwards; Docker build layers and Gradle/Cargo caches remain available.
Rust and Java run exclusively inside Docker.

The container mounts the application sources read-only at `/source`, the
writable lockfile at `/lock/Cargo.lock`, and artifacts at `/output`.
Before every APK build it refreshes h2 (`master`), h3 and Quinn (`main`) from the
`zerox80` forks and verifies their current remote branch heads. The updated
lockfile is returned to the application checkout.

APKs and app bundles are checked for `libmavivpn.so` in all four ABIs before
export. Docker volumes retain Gradle, Cargo and the debug signing key.

## Maintenance

Keep SDK/NDK versions aligned with `android/app/build.gradle.kts` in `mavi`.
The command-line tools checksum comes from
[Android Developers](https://developer.android.com/studio#command-line-tools-only).
SDK licenses are accepted when the Docker image is built.

Unit tests use only Python's standard library and never invoke Cargo:

```bash
python3 -m unittest discover -s tests -v
```

Image builds run via Docker in WSL:

```bash
docker build --pull -t mavi-android-build:local .
```
