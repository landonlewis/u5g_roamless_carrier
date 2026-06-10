# U5G Backup Carrier Switcher

A shell script for switching the cellular carrier on a UniFi U5G-US backup device running a Roamless eSIM. Run from any Mac or Linux machine on the same network.

## Background

The UniFi U5G-US uses an eSIM with Roamless, which by default selects the carrier automatically — often landing on AT&T. This script lets you manually force a specific carrier (T-Mobile, Verizon, or AT&T) or return to automatic selection.

Carrier switching is done via AT commands sent to the modem over SSH using the `atcli` utility built into the U5G firmware.

---

## Requirements

- Mac or Linux with `ssh` available (standard on both)
- Network access to the U5G device
- SSH enabled on the U5G (see Setup below)

---

## Setup

### 1. Enable SSH on the U5G

In the UniFi Network console:

1. Go to **Devices**
2. Click the gear icon in the bottom-left corner — **Device Updates and Settings**
3. Expand **Device SSH Settings**
4. Set a username and password
5. Save

### 2. Clone or download this repo

```sh
git clone <repo-url>
cd <repo-dir>
```

### 3. Create the `.env` file

Copy the example and fill in your device details:

```sh
cp .env.example .env
```

Edit `.env`:

```sh
U5G_HOST="192.168.1.100"   # IP address of your U5G device
U5G_USER="your_username"    # SSH username set in UniFi console
```

> **Note:** `.env` is listed in `.gitignore` and will not be committed to version control.

### 4. Make the script executable

```sh
chmod +x carrier
```

### 5. (Optional) Set up SSH key authentication

By default the U5G prompts for your SSH password on every command. Setting up
key-based auth lets `carrier` run without a password prompt.

1. **Generate a key** (skip if you already have one at `~/.ssh/id_ed25519`):

   ```sh
   ssh-keygen -t ed25519 -C "u5g-carrier"
   ```

   Press Enter to accept the default location. You can set a passphrase or leave
   it blank.

2. **Copy the key to the U5G:**

   ```sh
   ssh-copy-id <U5G_USER>@<U5G_HOST>
   ```

   You'll be prompted for your SSH password one last time. If `ssh-copy-id`
   isn't available, copy the key manually:

   ```sh
   cat ~/.ssh/id_ed25519.pub | ssh <U5G_USER>@<U5G_HOST> 'cat >> ~/.ssh/authorized_keys'
   ```

3. **Verify it works without a password:**

   ```sh
   ssh <U5G_USER>@<U5G_HOST> 'echo connected'
   ```

   If it prints `connected` without asking for a password, you're set.

> **Note:** Some UniFi firmware resets `authorized_keys` on reboot or firmware
> update. If you start getting password prompts again, re-run step 2.

### 6. (Optional) Add to PATH

To run `carrier` from anywhere without `./`:

```sh
echo "export PATH=\"\$PATH:$(pwd)\"" >> ~/.zshrc
source ~/.zshrc
```

---

## Usage

```sh
./carrier <command> [host]
```

| Command | Description |
|---------|-------------|
| `status` | Show the currently connected carrier |
| `signal` | Show current signal strength |
| `info` | Show carrier status and signal together |
| `auto` | Let Roamless automatically select the best carrier |
| `tmobile` | Force T-Mobile (MNC: 310260) |
| `att` | Force AT&T (MNC: 310410) |
| `verizon` | Force Verizon (MNC: 311480) |

### Examples

```sh
./carrier status          # Check current carrier
./carrier signal          # Check signal strength
./carrier info            # Carrier + signal together
./carrier tmobile         # Switch to T-Mobile
./carrier verizon         # Switch to Verizon
./carrier att             # Switch to AT&T
./carrier auto            # Let Roamless decide

# Override host for a one-off command:
./carrier tmobile 192.168.1.200
```

---

## Files

| File | Description |
|------|-------------|
| `carrier` | Main shell script |
| `.env` | Your local config (not committed) |
| `.env.example` | Example env file to copy from |
| `.gitignore` | Excludes `.env` from version control |

---

## How It Works

The script SSHes into the U5G and sends an AT command to the modem via `atcli`:

| Carrier | AT Command |
|---------|------------|
| Auto | `AT+COPS=0` |
| T-Mobile | `AT+COPS=1,2,"310260",7` |
| AT&T | `AT+COPS=1,2,"310410",7` |
| Verizon | `AT+COPS=1,2,"311480",7` |

After switching, it waits 3 seconds and reports the active network to confirm the change took effect. The script maps the modem's numeric MNC to a friendly carrier name, so status output looks like `Carrier: AT&T (310410)`.

### Status, signal, and info

- `status` reads the active carrier via `atcli -o` and prints the labeled name.
- `signal` queries the modem directly with `qmicli -d qrtr://3 --nas-get-signal-info` to report signal strength (RSSI, RSRP, SNR, etc.).
- `info` runs both, showing carrier and signal in one view. This is also what's shown automatically after a carrier switch.

> **Note:** The `signal` command uses the QMI device path `qrtr://3`, which is specific to the U5G's modem. If `signal` stops working after a firmware update, that path may have changed — check it on the device with `qmicli -L` or `ls /dev/qcqmi*`.

---

## Notes

- **Carrier changes take effect immediately** — no reboot required, though expect a few seconds of connectivity loss during the switch
- **Settings do not survive a reboot** — if the U5G reboots it will return to automatic carrier selection. Re-run the script to force your preferred carrier again
- **Roamless controls the eSIM** — if a carrier isn't available through your Roamless plan, the command will fail. Contact Roamless support to confirm which US carriers are supported on your eSIM profile
- The numeric codes used (`310260`, `310410`, `311480`) are standard US carrier MCC/MNC identifiers

---

## Troubleshooting

**`ERROR: .env file not found`**
Create the `.env` file as described in Setup step 3.

**Password prompted every time**
Set up SSH key auth as described in Setup step 5.

**Command returns `ERROR` or `Timeout`**
- Verify the U5G is online and reachable: `ping <U5G_HOST>`
- Verify SSH works: `ssh <U5G_USER>@<U5G_HOST> 'echo connected'`
- The requested carrier may not be available through your Roamless eSIM profile — try `auto` to reset, then contact Roamless support

**Carrier shows a numeric code with no name (e.g. `310260`)**
The script labels known carriers automatically (e.g. `Carrier: T-Mobile (310260)`). If you see a raw code with `Unknown carrier`, the modem returned an MNC the script doesn't recognize — the raw `atcli` output above it still shows the full network details.
