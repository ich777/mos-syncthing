# MOS Syncthing

mos-syncthing provides a **MOS plugin** that integrates
[Syncthing](https://github.com/syncthing/syncthing)
into the MOS ecosystem.

---

## Overview

This repository contains the **MOS plugin implementation**, optional helper
functions, configuration files (such as `settings.json`).

The plugin allows MOS to make use of Syncthing for continuous file
synchronization between multiple devices.

### Binary Source

- Syncthing: [https://github.com/syncthing/syncthing](https://github.com/syncthing/syncthing)

---

## Build & Automation

This repository includes a **GitHub Actions workflow** used to build and package
the plugin and its associated components for MOS.

The build process is fully automated and produces artifacts that can be
installed through the MOS Hub.

---

## Licensing

The contents of this repository (plugin code, build scripts, configuration,
and automation) are licensed under **GPL-3.0**.

`Syncthing` itself is licensed under its respective upstream license (MPL-2.0).

---

## Third-Party Software

This repository builds and packages third-party open-source software.
Packaged components remain licensed under their original upstream licenses.

Refer to `THIRD_PARTY.md` for details.
