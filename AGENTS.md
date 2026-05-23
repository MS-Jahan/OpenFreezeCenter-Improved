# AGENTS.md — OpenFreezeCenter Improved

This guide provides developer agents and human contributors with a comprehensive overview of the OpenFreezeCenter (OFC) project, its architecture, details on how it interacts with hardware, and development best practices.

---

## Project Overview

**OpenFreezeCenter** is a Linux-native graphical utility designed for fan speed control, system temperature/RPM monitoring, and battery charge threshold configuration on MSI laptops (specifically tested on MSI GP76 11UG). 

MSI does not provide official software for Linux. This utility bridges the gap by reading and writing to the hardware's Embedded Controller (EC) directly via the Linux kernel's debugfs interface: `/sys/kernel/debug/ec/ec0/io`.

---

## Architecture & Code Structure

The project is structured as a small, lightweight set of scripts.

```
/mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/
├── OFC.py          # Main GTK3 GUI application and fan/battery control logic
├── install.sh      # Setup script (dependencies, venv, Expect runners, desktop copy)
├── file_1.sh       # Expect script to configure /etc/modprobe.d/ec_sys.conf (write support)
├── file_2.sh       # Expect script to configure /etc/modules-load.d/ec_sys.conf (kernel autoload)
├── README.md       # User documentation and instructions
└── LICENSE         # AGPL v3 License file
```

### Runtime Architecture and Lifecycle

```mermaid
graph TD
    User([User]) -->|Launches| install.sh[install.sh]
    install.sh -->|Sets up| Venv[Python venv: ~/Desktop/OFC]
    install.sh -->|Spawns| file_1[file_1.sh: modprobe config]
    install.sh -->|Spawns| file_2[file_2.sh: modules-load config]
    file_1 -->|Configures| sys1[/etc/modprobe.d/ec_sys.conf]
    file_2 -->|Configures| sys2[/etc/modules-load.d/ec_sys.conf]
    install.sh -->|Launches app as root| OFC.py[OFC.py]
    OFC.py -->|Reads & Writes| EC[/sys/kernel/debug/ec/ec0/io]
    OFC.py -->|Wizard generates| config.py[config.py]
```

### Core Components

#### 1. Main Application: [OFC.py](file:///mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/OFC.py)
* **GUI Engine**: Built with PyGObject (GTK 3) and PyCairo.
* **EC Interactions**:
  - `read(BYTE, SIZE, FORMAT)`: Performs binary seek and reads 1 or 2 bytes from `/sys/kernel/debug/ec/ec0/io`. Format options determine whether to return an integer or a hexadecimal string.
  - `write(BYTE, VALUE)`: Accesses `/sys/kernel/debug/ec/ec0/io` and writes a single byte at the specified index.
* **Fan Profile Engine**: Translates high-level choices into specific EC register writes. Handles Auto, Basic, Advanced, and Cooler Booster modes.
* **Auto-Configuration Wizard**: Prompts the user via GTK dialogs on first startup to generate the machine-specific [config.py](file:///mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/config.py) module.

#### 2. Configuration Module (Generated at Runtime): `config.py`
This file is generated dynamically in `OFC.py`'s entry point if it is not found. It stores:
* **Selected Fan Profile**: ID representing the active mode (1-4).
* **Fan Speeds**: Speed curves (CPU & GPU) for Auto and Advanced modes.
* **CPU Gen Flag**: `1` for Intel 10th Gen and above, `0` for 10th Gen and below.
* **EC Address Map**: Target offsets for fan speed sliders, temperature readers, RPM outputs, and Cooler Booster toggles.
* **Battery Charge Limit**: Target charging threshold percentage (50-100).

> [!WARNING]
> Do NOT commit `config.py` to source control. It contains machine-specific offsets generated on the user's laptop and can disrupt installations on other models.

#### 3. Installer and Configuration Scripts: [install.sh](file:///mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/install.sh), [file_1.sh](file:///mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/file_1.sh), and [file_2.sh](file:///mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/file_2.sh)
* **Dependencies**: Updates apt packages and installs `python3-pip`, `libgirepository1.0-dev`, `libcairo2-dev`, `expect`, and `uv` (installed via pip if not present on the system path).
* **Venv Setup**: Uses the `uv` package manager to create the virtual environment and install `PyGObject` and `pycairo` at `~/Desktop/OFC` via `uv pip install`.
* **Kernel Module Configuration**:
  - Requires `ec_sys` module configuration. To write to registers safely, the `ec_sys` kernel module must be loaded with `write_support=1`.
  - The scripts `file_1.sh` and `file_2.sh` are **Expect** scripts that automate starting `nano` under `sudo` to append lines to `/etc/modprobe.d/ec_sys.conf` and `/etc/modules-load.d/ec_sys.conf`.

---

## Embedded Controller (EC) Details

The program operates by reading and writing to standard offsets in the EC space. Below is the breakdown of known registers and their equations.

### Register Mapping Matrix

| Register / Offset | CPU Gen >= 10th (`CPU = 1`) | CPU Gen < 10th (`CPU = 0`) | Purpose | Read/Write | Valid Range |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Cooler Booster Control** | `0x98` | `0x98` | Toggles maximum fan speeds (Cooler Booster) | R/W | `2` (Off), `130` (On) for CPU=1; `0` (Off), `128` (On) for CPU=0 |
| **Profile Speed Select** | `0xd4` | `0xf4` | Sets active profile curve type (Auto vs Advanced/Basic) | R/W | `13` (Auto), `141` (Advanced) for CPU=1; `12` (Auto), `140` (Advanced) for CPU=0 |
| **CPU Temp Address** | `0x68` | `0x68` | Reads current CPU temperature in degrees C | Read | `0` to `100` |
| **GPU Temp Address** | `0x80` | `0x80` | Reads current GPU temperature in degrees C | Read | `0` to `100` |
| **CPU Fan RPM** | `0xc8` | `0xc8` | Reads raw CPU fan tachometer value | Read | Word (2 Bytes) |
| **GPU Fan RPM** | `0xca` | `0xca` | Reads raw GPU fan tachometer value | Read | Word (2 Bytes) |
| **Battery Threshold** | `0xe4` | `0xe4` | Configures max charge threshold | R/W | `178` (50% charge limit) to `228` (100% charge limit) (Calculated as `Limit + 128`) |
| **CPU Fan Speeds** | `0x72` to `0x78` | `0x72` to `0x78` | 7-point fan curve offsets for CPU | R/W | `0` to `150` |
| **GPU Fan Speeds** | `0x8a` to `0x90` | `0x8a` to `0x90` | 7-point fan curve offsets for GPU | R/W | `0` to `150` |

### Key Formulas

#### Fan RPM Calculation
Fan speeds are read from the EC as tachometer counts. Convert the counts to rotations-per-minute (RPM) using:
$$\text{RPM} = \frac{478000}{\text{Tachometer Value}}$$

If the tachometer returns `0`, the RPM is defined as `0` to avoid division by zero.

#### Battery Charge Threshold
The battery threshold register accepts values between `178` (50% charge) and `228` (100% charge).
$$\text{EC Byte Value} = \text{Desired Percentage} + 128$$

---

## Development Best Practices

### 1. Embedded Controller Safety (Critical)
> [!CAUTION]
> The Embedded Controller directly controls electrical hardware and cooling. Incorrect writes can trigger physical thermal shutdowns or prevent fans from spinning, potentially damaging hardware.
* Always clamp fan curve speeds to safe ranges (`0` to `150`).
* Validate battery thresholds strictly between `50` and `100` before writing to `0xe4`.
* When switching to profiles 1-3 (Auto, Basic, Advanced), **always disable Cooler Booster first** before writing the target curve. The Cooler Booster bit override takes absolute precedence.

### 2. Single-File Codebase Constraint
* [OFC.py](file:///mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/OFC.py) is a single-file application. Keep GUI code, EC helpers, and configuration handlers within this file. Avoid splitting logic into custom sub-modules unless introducing extremely large independent features (e.g., a completely different dashboard).

### 3. Non-Blocking GTK UI Update Loops
* Never use `time.sleep()` inside GTK callback event handlers or monitoring loops. Doing so freezes the user interface.
* Use `GLib.timeout_add(500, update_label)` to schedule background sensor polling. Ensure this function returns `True` to continuously fire every 500ms.

### 4. Configuration Driven Addresses
* Never hardcode EC addresses inside UI layouts or profile functions. Always query `config.py` for target addresses (e.g. `config.CPU_GPU_TEMP_ADDRESS`).

### 5. Automation & Expect Scripts
* The scripts [file_1.sh](file:///mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/file_1.sh) and [file_2.sh](file:///mnt/DABCEB02BCEAD851/Projects/OpenFreezeCenter-Improved/file_2.sh) simulate inputs to `nano`. If modifying these files, exercise extreme care with escape sequences (`\x1B` for Escape, `\x13` for Ctrl+S, etc.) and expect patterns. Ensure `install.sh` remains idempotent so it can be re-run safely if an installation fails or is updated.

---

## Testing Workflows

### Mocking EC Interface (Development on Non-MSI Machine)
If developing on a standard machine or virtual machine without the `/sys/kernel/debug/ec/ec0/io` file, python will throw a `FileNotFoundError` immediately during configuration setup or updates.

For testing UI transitions, stub the `read()` and `write()` methods:

```python
# Insert for local testing/mocking
def write(BYTE, VALUE):
    print(f"[MOCK EC WRITE] Register {hex(BYTE)} -> {VALUE}")

def read(BYTE, SIZE, FORMAT):
    # Return placeholder temperature values or dummy RPM counts
    if BYTE in [0x68, 0x80]: # Temp
        return 45
    if BYTE in [0xc8, 0xca]: # RPM Tachometer
        return 200 # yields ~2390 RPM
    return 0 if FORMAT == 0 else "00"
```

### Verification Checklist
When modifying profile settings, verify the following:
1. **Auto Mode**: Select "Auto" profile and confirm that registers `0xd4`/`0xf4` update to `13`/`12`, and the fan speed curves match the configured defaults.
2. **Battery Threshold**: Select different battery limit percentages (e.g., 60%, 80%) and confirm that the value written to `0xe4` is `188`/`208`.
3. **Cooler Booster**: Verify that selecting Cooler Booster correctly toggles the `0x98` register to `130`/`128` depending on CPU gen, and immediately spins up the fans.
