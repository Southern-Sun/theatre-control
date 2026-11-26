# Projector Control

A Python 3.11+ CLI for PJLink Class 1 power management over LAN, initially
targeting the Epson Pro Cinema LS12000. Runtime dependencies are standard library
only. No daemon or discovery service is needed.

## Install on CachyOS / Arch Linux (Fish)

Run from this repository. A virtual environment keeps installation separate from
Arch's system Python. Install build tools normally; no global pip installation is needed.

```fish
sudo pacman -S --needed python python-pip python-setuptools
cd /home/john/Documents/Repositories/theatre-control
python -m venv --system-site-packages .venv
.venv/bin/python -m pip install --no-build-isolation --no-deps .
fish_add_path /home/john/Documents/Repositories/theatre-control/.venv/bin
projector --help
```

The system site packages supply setuptools for the offline build step. After
editing source, repeat the pip install command. After an Arch Python major-version
upgrade, recreate the virtual environment and reinstall.

## Configure

```fish
projector init
nano ~/.config/projector-control/config.toml
```

`init` creates `~/.config/projector-control/config.toml` with mode 600 and never
overwrites an existing file. The example is bundled in the installed package.

```toml
[projector]
host = "192.168.88.50"
port = 4352
timeout = 5.0

[authentication]
password_env = "PROJECTOR_PJLINK_PASSWORD"
```

Use the projector's actual IP or hostname, preferably with a DHCP reservation.
The config is required even with host/port overrides. Timeout is in seconds:
connection establishment has this timeout, followed by one shared timeout budget
for the greeting, command send, and response. OS DNS resolution may take longer.

For an interactive Fish session, enter the password without putting it in history:

```fish
read --silent --prompt-str 'PJLink password: ' --export PROJECTOR_PJLINK_PASSWORD
```

The environment variable named by `password_env` takes precedence over an optional
`password = "..."` in the `[authentication]` table. An empty environment variable
also takes precedence; remove it with `set -e PROJECTOR_PJLINK_PASSWORD` to use the
file fallback. Unset credentials are fine when the projector disables authentication.
For unattended KDE use, a private file password is convenient:

```fish
nano ~/.config/projector-control/config.toml
chmod 600 ~/.config/projector-control/config.toml
```

The client accepts up to 32 printable ASCII password characters; follow your
projector's stricter rules. Credentials and raw protocol messages are never printed.
Keep real credentials outside the repository. PJLink's MD5 authentication does not
encrypt traffic; use it on a trusted LAN.

For the LS12000, connect Ethernet and configure wired networking. Set
**Operation > Standby Mode > Communication On** to allow control while off.
Set the **PJLink Password** in the projector's network control settings. The manual's
**PJLink On/Off** setting is described as enabling notifications, which this CLI does
not use. See the [Epson manual](https://files.support.epson.com/docid/cpd6/cpd61119.pdf),
pages 37–40 and 112–113, for these settings and password restrictions.

## Unit tests (no projector required)

```fish
cd /home/john/Documents/Repositories/theatre-control
env PYTHONPATH=src python -m unittest discover -s tests -v
```

Tests mock TCP connections, including fragmented reads, and cover authentication,
all four states, on/off acknowledgements, protocol errors, network failures,
configuration validation, socket cleanup, and CLI behavior. They never contact hardware.

## Safe real-projector test

```fish
projector test
projector status
# Optional connection overrides, before or after the command:
projector test --host 192.168.88.50 --port 4352
```

`test` connects to TCP, handles authentication if requested, and sends only
`%1POWR ?`. A successful reply confirms all three steps. It never sends a power
change. `status` prints exactly `Off`, `On`, `Cooling`, or `Warming` followed by a newline.
Hardware integration has not been verified during development.

## Power commands (run manually)

```fish
projector on
projector status
projector off
```

`on` and `off` report an accepted request, not a completed transition. Query status
later to observe the result. Commands are not retried automatically, and the CLI
does not wait for warming/cooling to finish.

## Shell scripts and KDE shortcuts

Output goes to stdout, errors to stderr. No command prompts for input.

| Exit code | Meaning |
| --- | --- |
| 0 | Success, including an already-existing config for `init` |
| 2 | Invalid CLI arguments or missing/invalid configuration |
| 3 | Network failure or timeout |
| 4 | Missing or rejected authentication credentials |
| 5 | PJLink protocol error, unsupported greeting, or unexpected response |

`status` returns 0 for **all** successfully queried states, including Off.
For a Fish theatre-mode script, check both command success and state as needed:

```fish
set power_state (projector status)
set projector_result $status
if test $projector_result -ne 0
    exit $projector_result
end
if test "$power_state" = On
    echo 'Projector is ready'
end
```

In KDE System Settings, add a custom command shortcut with an absolute path:

```text
/home/john/Documents/Repositories/theatre-control/.venv/bin/projector on
/home/john/Documents/Repositories/theatre-control/.venv/bin/projector off
```

Use separate shortcuts. KDE may not inherit Fish's PATH or session password
variable. Use the private config password or explicitly supply the environment
in your automation. Avoid placing passwords in shortcut command lines.

## Troubleshooting

- Missing config: run `projector init`, then edit the host. Invalid TOML errors
  deliberately omit source content to avoid exposing passwords.
- Connection refused: confirm the IP, port 4352, and PJLink availability in the
  projector settings. Make sure another service is not using a port override.
- Timeout: check Ethernet, LAN routing/VLAN isolation, firewall rules, and standby
  communication. Try a longer `timeout` if the device responds slowly. A successful
  ping alone does not verify PJLink.
- Authentication required/rejected: use the PJLink password, which may differ from
  web control credentials. Check the configured environment variable and its
  precedence over the file password. Rerun `projector test` after correcting it.
- Unsupported authentication or malformed response: verify the endpoint supports
  PJLink Class 1 rather than a web or vendor-specific control service.
- ERR1 means undefined command, ERR2 invalid parameter, ERR3 temporarily unavailable
  (often warming/cooling), and ERR4 device failure. Wait before manually retrying
  ERR3; check projector indicators and the manual for ERR4.

Protocol reference: [PJLink Class 1 specification](https://pjlink.jbmia.or.jp/english/data/5-1_PJLink_eng_20131210.pdf).
Only POWR is implemented, with CR framing and challenge-response authentication.
