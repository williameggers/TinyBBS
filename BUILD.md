# Building TinyBBS (beginner's guide)

This guide explains how to build the **Linux server** version of TinyBBS from
source. You do not need to know C++ or CMake beforehand: the commands below are
the whole build, and each step says what it does and what "working" looks like.

There are two different builds in this repository:

| Build | What it is | Built with |
| --- | --- | --- |
| Server (this guide) | A Linux program that serves the BBS to a browser and Telnet | CMake + a C++ compiler |
| Firmware | The program that runs on the ESP32-S3 USB stick | PlatformIO (see [Firmware](#firmware-build-and-flash-the-esp32-s3) at the end) |

The server build produces two programs:

- `tinybbs-phase1-smoke` — a short check that starts the BBS core and prints the
  settings. It exits straight away.
- `tinybbs-server` — the real long-running server. It keeps going until you
  stop it with Ctrl-C.

## A few words about the tools

- **Compiler** (`g++`): turns the C++ source files into machine code. It comes
  in the `build-essential` package.
- **CMake**: reads the build recipe in `server/CMakeLists.txt` and works out
  the exact compiler commands. You need version 3.20 or newer.
- **Libraries**: ready-made code the server uses. mbedTLS calculates the
  password hashes; CivetWeb provides the HTTP and WebSocket server. The
  "`-dev`" packages contain the header files and linkable libraries needed to
  compile against them.
- **Build folder** (`server/build/`): contains generated files. It is safe to
  delete and recreate at any time.

All commands below are run from the **repository root** — the folder that
contains `README.md`, `server/` and `sd/`. If you are unsure where you are,
run `cd /workspaces/TinyBBS` when using the supplied dev container, or `cd`
into your own checkout before starting.

## Option 1: use the dev container (recommended)

The dev container already installs the compiler, CMake, both libraries and
PlatformIO, so you do not install anything by hand.

1. Install [Docker](https://docs.docker.com/get-docker/) and the
   **Dev Containers** extension in VS Code.
2. Open this repository's folder in VS Code.
3. Press `Ctrl+Shift+P`, choose **Dev Containers: Reopen in Container**, and
   wait for the first build to finish (a few minutes the first time).
4. Open a terminal inside the container (VS Code menu: Terminal → New
   Terminal) and check the tools:

   ```sh
   cmake --version
   g++ --version
   ```

   `cmake version 3.20` or newer is fine. If either command is missing, the
   container image did not build properly — rebuild it with
   **Dev Containers: Rebuild Container**.

Skip to [Build the server](#build-the-server).

## Option 2: install the tools on Debian 13 (trixie)

If you are working directly on a Debian machine instead of in the container,
install the four prerequisites once:

```sh
sudo apt update
sudo apt install -y build-essential cmake libmbedtls-dev libcivetweb-dev
```

Then check them:

```sh
cmake --version
g++ --version
```

On other Linux distributions the package names differ. For example, on Rocky
Linux they are `gcc-c++ cmake mbedtls-devel civetweb-devel` (CivetWeb may need
the EPEL repository). The dev container avoids this entirely.

## Build the server

Run these two commands from the repository root:

```sh
cmake -S server -B server/build
cmake --build server/build
```

What they do:

1. The first command *configures* the build. It reads `server/CMakeLists.txt`,
   checks that the compiler and libraries are present, and creates the
   `server/build/` folder. `-S server` means "the source is in `server`";
   `-B server/build` means "put generated files in `server/build`".
2. The second command *compiles and links* — it turns the source code into the
   two programs. It is safe to run again after you edit a source file; only
   the changed parts are rebuilt.

A successful configure ends with lines like:

```
-- Configuring done
-- Generating done
-- Build files have been written to: .../server/build
```

A successful build ends with:

```
[100%] Built target tinybbs-server
```

The finished programs are:

```
server/build/tinybbs-phase1-smoke
server/build/tinybbs-server
```

### If a build fails

- `cmake: command not found` or `g++: command not found`: the tools are not
  installed. Use the dev container or run the `apt install` command above.
- An error mentioning `mbedtls/sha256.h` or `mbedcrypto`: `libmbedtls-dev` is
  missing.
- An error mentioning `civetweb.h` or `libcivetweb`: `libcivetweb-dev` is
  missing. CivetWeb is needed even for the small smoke check, because both
  programs are built from the same recipe.
- For a completely fresh start, delete the generated folder
  (`rm -rf server/build`) and run the two commands again.

## Check the build on a test copy

The programs need a folder that **contains** a `BBS/` subfolder, with the same
layout as an SD card. Never point them at a live card or a production data
tree: starting the BBS writes runtime files and saves the board clock.

Make a disposable copy from the card content in this repository:

```sh
mkdir -p /tmp/tinybbs-test
cp -a sd/BBS /tmp/tinybbs-test/
```

Now run the smoke check:

```sh
./server/build/tinybbs-phase1-smoke /tmp/tinybbs-test
```

Expected output looks like this (dates and numbers vary with the card):

```
[TINYBBS 00000000] SD_READY :: total=... free=...
SETTINGS: 0 file areas, no message history, clock 1997-03-14 21:00 (browser sync), 10 conferences, 2 bulletins, 8 nodes, 28800 baud, no notebook
```

The `SETTINGS:` line means the core read the card's configuration successfully.
`0 file areas` is normal for a fresh checkout — the file archive is not stored
in the repository. To see the exit status, run `echo $?`; `0` means success.

There is an optional second check that proves a copied account's password
hash matches:

```sh
./server/build/tinybbs-phase1-smoke /tmp/tinybbs-test SYSOP your-password
```

Skip this unless your test copy actually has that account; a fresh card does
not.

## Run the server

Start it with the test copy and tell it where the web pages are:

```sh
./server/build/tinybbs-server \
  --root /tmp/tinybbs-test \
  --web "$PWD/shared/web" \
  --http 127.0.0.1:8080
```

- `--root` is the folder containing `BBS/` (the same one used above).
- `--web` is the repository's `shared/web` folder, which holds `terminal.html`
  and `sysop.html`. You must pass this in a source checkout: the built-in
  default assumes the installed layout `/srv/tinybbs/src/shared/web` and will
  fail here.
- `--http` is the address and port to listen on. `127.0.0.1:8080` means "only
  this machine, port 8080".

When it starts you should see lines like:

```
[TINYBBS 00000000] SERVER_START :: root=/tmp/tinybbs-test platform=server
[TINYBBS 00000000] SETTINGS :: ...
[TINYBBS 00000001] HTTP_READY :: 127.0.0.1:8080
[TINYBBS 00000001] TELNET_OFF :: not configured
```

Open <http://127.0.0.1:8080> in a browser on the same machine and press **DIAL**
to get a node. A fresh card has no users, so create the `SYSOP` account first
as described in [README.md](README.md#getting-started).

Press **Ctrl-C** in the terminal to stop the server cleanly.

If you are running the server inside a dev container but browsing from your
host computer, forward port 8080. In VS Code use **Ports → Forward a Port**, or
add `"forwardPorts": [8080]` to `.devcontainer/devcontainer.json` and rebuild
the container.

### Optional: Telnet

To try the Telnet side as well, add `--telnet 127.0.0.1:2323` to the command
above, then connect from another terminal:

```sh
telnet 127.0.0.1 2323
```

Some systems do not ship a Telnet client; on Debian it is in the `telnet` or
`inetutils-telnet` package. Browser and Telnet callers share the same BBS
sessions.

### Common runtime errors

| Message | Meaning and fix |
| --- | --- |
| `TinyBBS needs a folder containing BBS/: ...` | `--root` must be the folder that *contains* `BBS/`, for example `/tmp/tinybbs-test`, not `/tmp/tinybbs-test/BBS`. |
| `TinyBBS needs a shared/web folder: ...` | Add `--web "$PWD/shared/web"` as shown above. |
| `TinyBBS could not start configured transports` | The port is already in use, or the web folder was wrong. Change `--http` to another port, for example `127.0.0.1:8081`. |
| `Login failed for SYSOP` during the smoke check | That account is not in the copied card, or the password is wrong. It is only an optional check. |

## Firmware: build and flash the ESP32-S3

PlatformIO is installed in the dev container and `pio` is on the PATH, so you
do not need to install it yourself. Check it with:

```sh
pio --version
```

### Build only (no board or USB cable needed)

Building the firmware does not need the board attached. From the repository
root, run:

```sh
pio run
```

The first run downloads the Espressif32 platform, compiler and the libraries
listed in `platformio.ini`, so it can take several minutes and needs network
access. Later runs are much faster.

The `esp32-s3-devkitc1-n16r8` board is a project-local PlatformIO definition in
`boards/`, not a board shipped with PlatformIO. Run `pio run` from the
repository root so PlatformIO can find it. If you get
`Unknown board ID 'esp32-s3-devkitc1-n16r8'`, check that
`boards/esp32-s3-devkitc1-n16r8.json` is present and that you are in this
checkout.

To throw away the build and start again:

```sh
pio run -t clean
pio run
```

The finished files are:

| File | What it is |
| --- | --- |
| `.pio-tinybbs/tinybbs_s3/firmware.bin` | The BBS application itself |
| `.pio-tinybbs/tinybbs_s3/bootloader.bin` | Starts the application |
| `.pio-tinybbs/tinybbs_s3/partitions.bin` | Describes the flash layout |
| `~/.platformio/packages/framework-arduinoespressif32/tools/partitions/boot_app0.bin` | A small boot partition image supplied by the Arduino core |

These four files are all you need to flash the stick. Choose one of the two
flashing routes below.

### Flashing route A: directly from the container (Linux host)

`.devcontainer/devcontainer.json` starts the container in privileged mode so a
host USB serial device is visible inside it. This works when Docker runs on a
Linux host and the board is plugged into that host.

1. Rebuild the container once if you changed that file
   (**Dev Containers: Rebuild Container**).
2. Plug the Pocket-Dongle-S3 into a USB port and list the serial devices:

   ```sh
   pio device list
   ```

   You should see something like `/dev/ttyACM0` or `/dev/ttyUSB0`.
3. Flash:

   ```sh
   pio run -t upload
   ```

   PlatformIO picks the port automatically when only one is connected. To
   choose one explicitly:

   ```sh
   pio run -t upload --upload-port /dev/ttyACM0
   ```

If the device list is empty, the host did not pass the USB device through. If
the device is listed but upload says "permission denied", the container user
cannot open it: the Dockerfile adds `vscode` to the `dialout` group, which
works when the host uses the usual Debian group ID. For a different ID, use a
host udev rule or uncomment `"remoteUser": "root"` in
`.devcontainer/devcontainer.json` and rebuild.

On Docker Desktop for macOS or Windows, USB devices are not shared with Linux
containers by default. Use route B instead.

### Flashing route B: copy the files to a macOS or Windows host

Here the container only builds; the host does the flashing with a small tool
called **esptool**. You do not need PlatformIO on the host.

#### 1. Collect the four files

Inside the container, copy them into one folder:

```sh
cd /workspaces/TinyBBS
mkdir -p firmware-for-host
cp .pio-tinybbs/tinybbs_s3/bootloader.bin \
   .pio-tinybbs/tinybbs_s3/partitions.bin \
   .pio-tinybbs/tinybbs_s3/firmware.bin \
   firmware-for-host/
cp "$HOME/.platformio/packages/framework-arduinoespressif32/tools/partitions/boot_app0.bin" \
   firmware-for-host/
ls -lh firmware-for-host
```

For a local VS Code dev container, the workspace folder is shared with the
host, so `firmware-for-host/` is already in the same folder on your computer —
open it in a host terminal. If the container is remote (for example GitHub
Codespaces), right-click the folder in VS Code's Explorer and choose
**Download...**, or make an archive first and download that:

```sh
tar -czf firmware-for-host.tar.gz firmware-for-host
```

Unzip it on the host before continuing.

#### 2. Install esptool on the host

**macOS** — open Terminal:

```sh
python3 -m venv ~/esptool-venv
~/esptool-venv/bin/pip install esptool
```

**Windows** — open PowerShell (the `py` launcher comes with Python installed
from python.org):

```powershell
py -m venv "$env:USERPROFILE\esptool-venv"
& "$env:USERPROFILE\esptool-venv\Scripts\pip.exe" install esptool
```

If `py` is not found, install Python 3 from <https://www.python.org/downloads/>
and tick **Add Python to PATH** during setup.

#### 3. Find the serial port

Plug the board into the host, not just the container.

- **macOS:** run `ls /dev/cu.*` and look for something like
  `/dev/cu.usbmodem1101` or `/dev/cu.usbserial-1234`.
- **Windows:** open Device Manager → **Ports (COM & LPT)** and note the port,
  for example `COM10`. In PowerShell you can also run
  `[System.IO.Ports.SerialPort]::GetPortNames()`.

#### 4. Flash the four files

Change into the `firmware-for-host` folder on the host and run the command for
your system, replacing the port with the one you found.

**macOS:**

```sh
~/esptool-venv/bin/esptool.py --chip esp32s3 --port /dev/cu.usbmodem1101 --baud 921600 \
  write-flash --flash-mode dio --flash-freq 80m --flash-size 16MB \
  0x0 bootloader.bin \
  0x8000 partitions.bin \
  0xe000 boot_app0.bin \
  0x10000 firmware.bin
```

**Windows (PowerShell):**

```powershell
& "$env:USERPROFILE\esptool-venv\Scripts\esptool.exe" --chip esp32s3 --port COM10 --baud 921600 `
  write-flash --flash-mode dio --flash-freq 80m --flash-size 16MB `
  0x0 bootloader.bin `
  0x8000 partitions.bin `
  0xe000 boot_app0.bin `
  0x10000 firmware.bin
```

The `--flash-mode dio` flag is required for the verified N16R8 board: the
runtime image is QIO/OPI, but the serial write must be DIO. Do not change it to
`qio`. This rewrites only firmware regions; it does not touch the SD card.

Success looks like `Hash of data verified` for each of the four files, followed
by a clean reset without repeated USB connect/disconnect sounds. Confirm that
`THE_BOARD` appears as a 2.4 GHz Wi-Fi SSID. If an upload stalls, see the
"Flash and recovery" section of README.md for more detail.

Either flashing route gives the same result. After a firmware change, also run
the server build earlier in this guide, because both builds share the BBS core
in `src/`.

## More background

- [server/README.md](server/README.md) — what each server phase does and its
  command-line defaults.
- [README.md](README.md) — the board, its features and SD-card setup.
- [docs/APACHE_PORT.md](docs/APACHE_PORT.md) — the production deployment plan.
