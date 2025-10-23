# BUILDING NATIVE PACKAGES FOR DISLOCKER

This document provides instructions for building native system packages (e.g., `.deb` for Debian/Ubuntu, `.rpm` for Fedora/RHEL) from the `dislocker` source code.

For general compilation and installation, please see `README.md` and `INSTALL.md`.

## Building for Debian / Ubuntu 22.04 LTS

Ubuntu 22.04 LTS ships with **mbedTLS 2.28**, but the latest `dislocker` source code requires **mbedTLS 3.x**. The package for the new version has to be built before building dislocker.

This section provides the instructions based on a fresh install of [Ubuntu 22.04.5 LTS (Jammy Jellyfish)](https://releases.ubuntu.com/jammy/ubuntu-22.04.5-desktop-amd64.iso) 64-bit PC (AMD64) desktop image.

The commands can be executed in the default terminal emulator (GNOME Terminal).

### Building mbedTLS 3.x package

#### Update system and enable source repositories

```bash
sudo apt update
sudo apt -y full-upgrade
sudo add-apt-repository -y universe multiverse restricted
sudo sed -i -E 's/^# deb-src ([^#]+jammy[^#]*)/deb-src \1/' /etc/apt/sources.list
sudo apt update
```

#### Install all build tools and dependencies

```bash
sudo apt-get install -y \
  build-essential ca-certificates cmake curl \
  debhelper devscripts doxygen equivs \
  faketime fakeroot git graphviz \
  lintian pkg-config quilt ubuntu-dev-tools wget
```

#### Configure the Build Environment

Set the maintainer identity to prevent warnings and define a fixed timestamp to ensure a reproducible build.

```bash
export DEBFULLNAME="Local Build"
export DEBEMAIL="local@build.org"
export DAK_TIMESTAMP="2025-10-15 00:00:00"
```

#### Download and Prepare Source Files

Fetch the Debian packaging files and the newer upstream source code. It creates a clean workspace, gets Debian's 3.6.4 source package (for its `debian` directory), and prepares the upstream 3.6.5 source tarball for use with `uupdate`.

```bash
mkdir -p ~/build/mbedtls && cd ~/build/mbedtls
dget -ux https://deb.debian.org/debian/pool/main/m/mbedtls/mbedtls_3.6.4-2.dsc
wget https://github.com/Mbed-TLS/mbedtls/releases/download/mbedtls-3.6.5/mbedtls-3.6.5.tar.bz2
mv mbedtls-3.6.5.tar.bz2 ../mbedtls_3.6.5.orig.tar.bz2
```

#### Merge New Source with Debian Packaging

Use `uupdate` to merge the new upstream source with the older Debian packaging files, creating the final source tree for building.

```bash
cd mbedtls-3.6.4
uupdate -v 3.6.5 ../mbedtls_3.6.5.orig.tar.bz2
cd ../mbedtls-3.6.5
```

#### Patch debian/control for Jammy Compatibility

This safely removes dependencies that are too new for Ubuntu 22.04, a crucial step to prevent build failures.

```bash
sed -i '/dpkg-build-api/d' debian/control
sed -i -E 's/dpkg-dev \(>= 1\.22\.5\)/dpkg-dev/' debian/control
```

#### Finalize the Changelog

Create the changelog entry for the backport. Setting `TZ=UTC` ensures the timestamp is in a standardized timezone, which is a packaging best practice.

```bash
TZ=UTC dch -v 3.6.5-0ubuntu1 -D jammy "Backport Mbed TLS 3.6.5 for Jammy."
```

#### Build the Packages (Two-Stage Process)

This two-stage process is necessary to correctly update the library symbols files and create a clean build without warnings.

##### Stage 1: Generate and Update Symbols

First, perform an initial build. This build is expected to produce warnings about mismatched symbols, but it will generate the correct symbol files that we need for the new version.
Then, copy the newly generated symbols files into the debian source directory.
At last, clean up the new symbols files to conform to Debian policy. This removes the Debian revision from symbol versions and deletes '#MISSING' tags.

```bash
faketime -f "$DAK_TIMESTAMP" debuild -b -us -uc || true
cp debian/libmbedcrypto16/DEBIAN/symbols debian/libmbedcrypto16.symbols
cp debian/libmbedtls21/DEBIAN/symbols debian/libmbedtls21.symbols
sed -i -e 's/3\.6\.5-0ubuntu1/3.6.5/g' \
       -e '/#MISSING/d' \
       debian/libmbedcrypto16.symbols debian/libmbedtls21.symbols
```

##### Stage 2: Final Clean Build

Clean the build tree and run the build process again. With the corrected symbols files and a fixed timestamp, this final build will be clean and will not produce any symbols-related or clock skew warnings from `lintian`.

```bash
fakeroot debian/rules clean
faketime -f "$DAK_TIMESTAMP" debuild -b -us -uc
```

The list of 11 files that belong to the new final build are those named `*3.6.5-0ubuntu1*`:

1.  **The Installable Packages (.deb files).** These are the most important files.
    *   `libmbedcrypto16_3.6.5-0ubuntu1_amd64.deb`
    *   `libmbedtls21_3.6.5-0ubuntu1_amd64.deb`
    *   `libmbedx509-7_3.6.5-0ubuntu1_amd64.deb`
    *   `libmbedtls-dev_3.6.5-0ubuntu1_amd64.deb`
    *   `libmbedtls-doc_3.6.5-0ubuntu1_all.deb`

2.  **The Debug Packages (.ddeb files).** These are for debugging purposes only.
    *   `libmbedcrypto16-dbgsym_3.6.5-0ubuntu1_amd64.ddeb`
    *   `libmbedtls21-dbgsym_3.6.5-0ubuntu1_amd64.ddeb`
    *   `libmbedx509-7-dbgsym_3.6.5-0ubuntu1_amd64.ddeb`

3.  **Build Information and Logs.** These files document how the packages were built. They are just text files, can't be installed.
    *   `mbedtls_3.6.5-0ubuntu1_amd64.buildinfo`
    *   `mbedtls_3.6.5-0ubuntu1_amd64.changes`
    *   `mbedtls_3.6.5-0ubuntu1.build`

#### Install the mbedTLS 3.x Packages

The `-dev` package is required to build `dislocker` later, but not to just use it. The `-doc` package is optional documentation.

```bash
cd ~/build/mbedtls
sudo apt install \
  ./libmbedcrypto16_3.6.5-0ubuntu1_amd64.deb \
  ./libmbedtls21_3.6.5-0ubuntu1_amd64.deb \
  ./libmbedtls-dev_3.6.5-0ubuntu1_amd64.deb \
  ./libmbedtls-doc_3.6.5-0ubuntu1_all.deb \
  ./libmbedx509-7_3.6.5-0ubuntu1_amd64.deb
```

All these packages can coexist with 2.x versions, except the `-dev` package, which will replace the older `libmbedtls-dev` if it is already installed.

A warning may appear when installing local `.deb` files that are not located in a system directory, but the installation is successful:

```
N: Download is performed unsandboxed as root as file '/home/<username>/build/mbedtls/libmbedcrypto16_3.6.5-0ubuntu1_amd64.deb' couldn't be accessed by user '_apt'. - pkgAcquire::Run (13: Permission denied)
```

#### Optional: Install Debug Symbol Packages

The build process also creates debug symbol packages (`.ddeb` files). These are **not required** for normal use or for building other software.

They are used by developers with debugging tools like `gdb` to analyze program crashes. The symbols provide a map from the compiled machine code back to the original source code, making it possible to see exactly which function in the code caused a problem.

If you are a developer and need to debug dislocker's interaction with these libraries, install them manually using `dpkg`.

```bash
cd ~/build/mbedtls
sudo dpkg -i \
  ./libmbedcrypto16-dbgsym_3.6.5-0ubuntu1_amd64.ddeb \
  ./libmbedtls21-dbgsym_3.6.5-0ubuntu1_amd64.ddeb \
  ./libmbedx509-7-dbgsym_3.6.5-0ubuntu1_amd64.ddeb
```
