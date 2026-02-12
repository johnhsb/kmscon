# Claude Code Project Instructions

## Project
kmscon - KMS/DRM based Linux terminal emulator with Hangul input support.
Fork of kmscon with libhangul integration for Korean input method.

## Build
```sh
meson setup build
ninja -C build
```
Debian package:
```sh
dpkg-buildpackage -us -uc -b
# Output: ../kmscon-hangul_<version>_arm64.deb
```

## Install
```sh
sudo dpkg -i ../kmscon-hangul_<version>_arm64.deb
sudo systemctl restart kmsconvt@tty1.service
```
Config: `/etc/kmscon/kmscon.conf`

## Key Source Files
- `src/kmscon_terminal.c` — terminal logic, hangul input processing
- `src/kmscon_conf.c` / `src/kmscon_conf.h` — config options
- `src/meson.build` — binary and module build definitions
- `meson.build` / `meson.options` — top-level build config, feature flags
- `debian/` — kmscon-hangul package build files

## Architecture
- Input path: evdev → XKB → keysym → `input_event()` → [hangul] → VTE → PTY
- Feature flags: `BUILD_ENABLE_*` in config.h (auto-generated)
- Hangul code guarded by `#ifdef BUILD_ENABLE_HANGUL`
- Modules: mod-pango, mod-unifont, mod-gltex, mod-drm3d (shared_module)
- libtsm: uses subproject fallback (system libtsm < 4.4.0)

## Conventions
- Language: C (gnu99), commit messages in English
- All new features must be guarded by `#ifdef BUILD_ENABLE_*`
- Debian package name: `kmscon-hangul` (Provides/Conflicts/Replaces kmscon)
- debian/rules: exclude libtsm files from package (system libtsm4 conflict)
- Build note: `-Wno-error=date-time` needed for dpkg-buildpackage

## Environment
- Platform: Raspberry Pi (aarch64), DietPi, Debian trixie
- Display: HDMI-A, drm2d backend, bbulk text renderer
- XKB: kr layout, pc105 model, kr104 variant
