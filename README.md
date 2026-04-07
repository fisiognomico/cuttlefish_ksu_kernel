# KernelSU Cuttlefish

Builds a KernelSU-Next + SUSFS patched GKI kernel for Android 13 Cuttlefish (x86_64). The kernel is built via GitHub Actions (or locally with `act`) using the WildKernels pipeline.

## Prerequisites

### GitHub Actions
Just push to the repo — no local setup needed.

### Local builds with `act`
- Docker
- [`act`](https://github.com/nektos/act) binary (e.g. `~/.local/bin/act`)
- ~50 GB free disk space

## Building with GitHub Actions

A push to `main` or `master` triggers the `build-cuttlefish.yml` workflow automatically. You can also trigger it manually from the Actions tab via **workflow_dispatch**.

To customize build parameters, trigger `build.yml` directly from the Actions UI and fill in the inputs (see [Build Inputs](#build-inputs)).

## Building with `act`

### Configure act

Create `~/.config/act/actrc` (if it doesn't exist) with:

```
-P ubuntu-latest=catthehacker/ubuntu:act-latest
```

### Run the build

```bash
~/.local/bin/act workflow_dispatch \
  -j build-gki \
  -W .github/workflows/build.yml \
  --input kernel_version=5.15.X-android13-lts \
  --input feature_set=WKSU+SUSFS \
  --input arch=x86_64
```

### Retrieve artifacts

`act` doesn't upload artifacts to GitHub. Copy them out of the Docker container:

```bash
# Find the container (it may still be running or recently exited)
docker ps -a | grep act

# Copy artifacts
docker cp <container_id>:/github/workspace/kernel-out/bzImage .
docker cp <container_id>:/github/workspace/kernel-out/initramfs.img .
```

## Build Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `kernel_version` | yes | `5.15.X-android13-lts` | Kernel version string (format: `MAJOR.MINOR.SUB-androidVER-PATCH`) |
| `feature_set` | yes | `WKSU+SUSFS` | Features to enable (`WKSU`, `SUSFS`, or `WKSU+SUSFS`) |
| `arch` | no | `x86_64` | Target architecture (`x86_64` or `aarch64`) |
| `variant` | no | | Build variant (e.g. `Bypass`, `Hakan` — aarch64 only) |
| `ksu_commit` | no | latest tag | KernelSU-Next commit or tag to use |
| `patches_commit` | no | latest | WildKernels kernel_patches commit |
| `susfs_commit` | no | latest | SUSFS commit |

## Output Artifacts

The build produces two files in `kernel-out/`:

| File | Description |
|---|---|
| `bzImage` | Compressed kernel image |
| `initramfs.img` | Initial ramdisk with kernel modules |

On GitHub Actions these are uploaded as a downloadable artifact named `<version>-kernel`.

## Running on Cuttlefish

### 1. Download system images

Get the Cuttlefish host package and system images from [Android CI](https://ci.android.com/) using the `aosp-android13-gsi` branch, target `aosp_cf_x86_64_phone-userdebug`:

- `cvd-host_package.tar.gz`
- `aosp_cf_x86_64_phone-img-*.zip`

### 2. Extract and launch

```bash
# Extract host package
tar -xzf cvd-host_package.tar.gz

# Unzip system images into the same directory
unzip aosp_cf_x86_64_phone-img-*.zip

# Launch with custom kernel
cvd create \
  -kernel_path=bzImage \
  -initramfs_path=initramfs.img
```

## Project Structure

| Path | Purpose |
|---|---|
| `.github/workflows/build-cuttlefish.yml` | Top-level workflow — triggers `build.yml` with Cuttlefish defaults |
| `.github/workflows/build.yml` | Main build pipeline (kernel download, patching, compilation) |
| `.github/actions/download-kernel/` | Clones GKI kernel source via `repo` |
| `.github/actions/wksu/` | Applies KernelSU-Next (Wild KSU) patches |
| `.github/actions/susfs/` | Integrates SUSFS source into the kernel tree |
| `.github/actions/susfs-patches/` | Applies SUSFS kernel patches |
| `.github/actions/configure-kernel/` | Sets kernel config options (defconfig fragment) |
| `.github/actions/clean-kernel-flags/` | Strips debug/tracing flags for a clean build |
