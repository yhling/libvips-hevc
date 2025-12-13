# Building Windows Binaries with libvips (HEVC Support)

## Overview

The `libvips/build-win64-mxe` repository provides a Docker-based build system for cross-compiling libvips and its dependencies for Windows. This document explains how the build process works and how to add `--with-hevc` support.

## Build Process Analysis

### Current Workflow Files

The repository currently has **one GitHub Actions workflow**:
- **`.github/workflows/oci-publish.yml`**: Publishes the base Docker image to GitHub Container Registry (`ghcr.io/libvips/build-win64-mxe:latest`)

### Build Script (`build.sh`)

The main build script (`build.sh`) handles the build process:

1. **Argument Parsing**: Accepts `--with-hevc` flag
2. **Plugin System**: Uses a plugin directory system to include dependencies
3. **Docker Container**: Builds and runs a Docker container with MXE (M cross environment)

### How `--with-hevc` Works

When you use `--with-hevc`:

1. **Flag Processing**:
   ```bash
   --with-hevc) with_hevc=true ;;
   ```

2. **Plugin Addition**:
   ```bash
   if [ "$with_hevc" = true ]; then
     plugin_dirs+=" /data/plugins/hevc-deps"
   fi
   ```

3. **GPL Library Detection**:
   ```bash
   [ "$with_hevc" = true ] && contains_gpl_libs=true
   ```
   - This means HEVC builds will **only create shared libraries** (not static) by default
   - Static builds with GPL libraries would violate the GPL license

4. **Environment Variable**:
   ```bash
   -e HEVC="$with_hevc" \
   ```
   Passed to the Docker container during the packaging step

### Build Targets

When `--with-hevc` is used, the default targets are:
- `x86_64-w64-mingw32.shared`
- `i686-w64-mingw32.shared`
- `aarch64-w64-mingw32.shared`

(Static builds are omitted due to GPL licensing)

## Creating a GitHub Actions Workflow

To automate building Windows binaries with HEVC support via GitHub Actions, you can create a workflow like this:

### Example Workflow

Create `.github/workflows/build-windows.yml`:

```yaml
name: Build Windows Binaries with HEVC

on:
  workflow_dispatch:
    inputs:
      variant:
        description: 'Build variant'
        required: true
        default: 'vips-web'
        type: choice
        options:
          - vips-web
          - vips-all
      with_hevc:
        description: 'Build with HEVC support'
        required: false
        default: false
        type: boolean

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Windows binaries
        run: |
          chmod +x build.sh
          ./build.sh \
            --with-hevc=${{ inputs.with_hevc }} \
            ${{ inputs.variant }}

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: windows-binaries
          path: packaging/*.zip
          retention-days: 30
```

### Automated Workflow (on Release)

For automatic builds on releases:

```yaml
name: Build Windows Binaries

on:
  release:
    types: [created]
  workflow_dispatch:
    inputs:
      with_hevc:
        description: 'Build with HEVC support'
        required: false
        default: false
        type: boolean

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        variant: [vips-web, vips-all]
        with_hevc: [false, true]
        exclude:
          # Skip HEVC builds for now (can be enabled later)
          - with_hevc: true
    permissions:
      contents: read
      packages: read

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Windows binaries
        run: |
          chmod +x build.sh
          if [ "${{ matrix.with_hevc }}" = "true" ]; then
            ./build.sh --with-hevc ${{ matrix.variant }}
          else
            ./build.sh ${{ matrix.variant }}
          fi

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: windows-binaries-${{ matrix.variant }}-${{ matrix.with_hevc && 'hevc' || 'no-hevc' }}
          path: packaging/*.zip
          retention-days: 90
```

## Local Build Instructions

To build locally with HEVC support:

```bash
# Clone the repository
git clone https://github.com/libvips/build-win64-mxe.git
cd build-win64-mxe

# Build with HEVC support
./build.sh --with-hevc vips-web

# Or for vips-all variant
./build.sh --with-hevc vips-all
```

The built binaries will be in the `packaging/` directory as zip files.

## Important Notes

1. **GPL Licensing**: HEVC support includes GPL-licensed libraries (x265), so:
   - Only shared libraries are built by default
   - Static builds are disabled to comply with GPL
   - You must comply with GPL if distributing binaries with HEVC support

2. **Patent Issues**: HEVC is patent-encumbered. Ensure you have appropriate licenses before distributing binaries with HEVC support.

3. **Build Time**: Building with HEVC support takes longer due to additional dependencies (libde265, x265).

4. **Dependencies**: The `--with-hevc` flag adds:
   - `libde265` (HEVC decoder)
   - `x265` (HEVC encoder)

## References

- Repository: https://github.com/libvips/build-win64-mxe
- Build Script: `build.sh`
- Base Dockerfile: `container/base.Dockerfile`
- Build Dockerfile: `container/Dockerfile`
- HEVC Plugin: `plugins/hevc-deps/`
