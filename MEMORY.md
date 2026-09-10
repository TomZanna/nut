# MEMORY - NUT `tripplitesu` driver work (handoff)

Handoff notes to continue development on the Tripp Lite SmartOnline driver
`drivers/tripplitesu.c` inside this devcontainer.

Last updated: after the devcontainer validation session.
Environment: Linux devcontainer (Debian bookworm) with gcc 12.2.0,
autoconf 2.71, automake 1.16.5, libtool 2.4.7.
Project version string in this tree: `2.8.5.1169-1169+gcc98cd759`.

## 0. TL;DR - current status

The feature is IMPLEMENTED, BUILD-VERIFIED and RUNTIME-VERIFIED (against a
simulated UPS; no real hardware). Nothing from the previous TODO list is open.

- Precedence implemented:

      cmdparam (upscmd value)  >  offdelay (ups.conf)  >  built-in default

- Stateless by design: no RAM staging via `upsrw`, so no persistence problem.
- Real SmartOnline hardware E2E is still NOT done (no hardware available);
  a pty-based simulator reproduces the serial protocol instead (section 8).

## 1. Goal

Per-invocation shutdown delay for the shutdown-related instant commands of
`drivers/tripplitesu.c`, chosen at `upscmd` time without restarting the driver
or editing `ups.conf`.

## 2. Scope decision (still valid)

The `instcmd()` code quoted in `Configurazione-Timeout-UPS-Tripp-Lite-in-NUT.md`
(with `TSU_SHUTDOWN_ACTION`, `ups.outlet_banks`, `do_command(SET, RELAY_OFF...)`)
is `drivers/tripplitesu.c`, NOT `drivers/tripplite.c`.

Other drivers (deliberately untouched by this work):
- `drivers/tripplite.c`: already reads `offdelay`/`startdelay`/`rebootdelay`
  from ups.conf and exposes RW `ups.delay.*`; only a CLI cmdparam override is
  missing there.
- `drivers/tripplite_usb.c`: already has `offdelay` + `ups.delay.shutdown`;
  `soft_shutdown()` uses `offdelay`; `startdelay`/`rebootdelay` are under `#if 0`.

User-approved decision: patch ONLY `drivers/tripplitesu.c`.

## 3. Git state - READ FIRST before making more edits

- Branch: `master`. HEAD is the commit `cc98cd759 "tmp"`, which ALREADY
  CONTAINS all the code/doc work described here. `git status` normally shows
  only untracked build artifacts; after this handoff edit, the only modified
  tracked file is this `MEMORY.md`.
- Tracked files changed by the "tmp" commit:

      drivers/tripplitesu.c        (+107/-7)   <- the feature
      docs/man/tripplitesu.txt     (+36)       <- new offdelay + INSTANT COMMANDS
      docs/man/upscmd.txt          (+10/-1)    <- SYNOPSIS ['value'] + OPTIONS
      NEWS.adoc                    (+7)        <- 2.8.6 bullet

- ALSO in the "tmp" commit but NOT for upstream (working notes only):

      MEMORY.md                                         (this file)
      Configurazione-Timeout-UPS-Tripp-Lite-in-NUT.md   (input notes, Italian)

  => When preparing the upstream PR, split/drop these two so that only the 4
     real files above go upstream.

- Untracked at the time of writing (build artifacts, do NOT commit):
  `.devcontainer/`, `tests/cgilib.c`, `tests/test_cgilib`.

- Do NOT create a commit / Signed-off-by on the contributor's behalf
  (section 11). Rewriting the "tmp" history is the human's decision.

### Build / test commands (devcontainer)

    ./autogen.sh
    ./configure --with-serial=yes
    make -j"$(nproc)"                          # full tree: 0 warnings, 0 errors
    make -C docs/man tripplitesu.8 upscmd.8    # needs asciidoc/a2x
    make distcheck-light                       # required before upstreaming

Docs rendering needs `asciidoc`/`a2x` plus `docbook-xsl`, `xsltproc`, `dblatex`;
on Debian:

    sudo apt-get install -y asciidoc docbook-xsl xsltproc libxml2-utils dblatex

then re-run `./configure` so it detects them (look for
`checking for asciidoc... /usr/bin/asciidoc`).

## 4. Exact code recap - `drivers/tripplitesu.c`

Line numbers refer to the current file (HEAD `cc98cd759`).

- Header comment (~lines 67-110): added `offdelay` to the ups.conf parameter
  list and a note that the shutdown commands accept an optional seconds value
  which takes precedence over `offdelay`, which in turn precedes the default.

- Defines after `#define MAX_RESPONSE_LENGTH 256` (lines 149-155):

      #define DEFAULT_OFFDELAY 10U
      #define MIN_OFFDELAY 1U
      #define MAX_OFFDELAY 3600U

  `0` is deliberately excluded: SDA "0" cancels a scheduled shutdown.

- Statics (lines 182-183):

      static unsigned int offdelay = DEFAULT_OFFDELAY;
      static int offdelay_from_conf = 0;

- `parse_delay(const char *val, unsigned int *delay)` (lines 497-512): rejects
  NULL/empty, uses `str_to_uint_strict(val, &tmp, 10)` (rejects leading
  whitespace, a leading `+`/`-`, trailing junk and overflow), then range-checks
  against MIN/MAX_OFFDELAY. Returns 1/0.

- `get_shutdown_delay(const char *cmdname, const char *extra,
  unsigned int fallback, unsigned int *delay)` (lines 518-533): starts from
  `fallback`; if `extra` is non-empty parses it via `parse_delay`, and on
  failure logs `LOG_ERR` `"instcmd(%s): invalid delay value '%s'
  (expected %u..%u seconds)"` and returns 0.

- `instcmd(const char *cmdname, const char *extra)` (from line 535):
  `NUT_UNUSED_VARIABLE(extra)` removed.
    * `shutdown.reboot` and `shutdown.return`: `get_shutdown_delay(cmdname,
      extra, offdelay, &delay)`, then `SDR;1` and `snprintf(parm,"%u",delay)`
      -> `SDA;<delay>`.
    * `shutdown.reboot.graceful`: same but fallback is `60U` (its own default,
      NOT affected by `offdelay`).
    * invalid `extra` -> `STAT_INSTCMD_FAILED` BEFORE any serial traffic.
    * `load.off`/`load.on`, `shutdown.stop`, `test.battery.*` unchanged
      (they ignore `extra`).

- `upsdrv_shutdown(void)` (lines 945-964): SDA delay is
  `offdelay_from_conf ? offdelay : 5U`, i.e. `offdelay` is honored only when
  explicitly set in ups.conf; otherwise the historical 5 s is kept (no behavior
  change for existing configurations).

- `upsdrv_makevartable(void)` (lines 976-988): adds the `offdelay` VAR_VALUE
  with help text built by `snprintf`:
  `"Set shutdown delay, in seconds (default=%u, range %u..%u)."`

- `upsdrv_initups(void)` (lines 990-1031): after the `command_delay` handling,
  reads `getval("offdelay")`; if non-empty validates with `str_to_uint_strict`
  + range and `fatalx(EXIT_FAILURE, "Invalid offdelay parameter: %s
  (expected %u..%u seconds)", ...)` on failure; otherwise sets
  `offdelay_from_conf = 1` and debugs at level 2.
  NOTE: `upsdrv_initups()` calls `ser_open()` FIRST, so this validation runs
  only after the serial port opens (same as the pre-existing `command_delay`) -
  see section 8 for the pty workaround.

## 5. Docs / NEWS recap

- `docs/man/tripplitesu.txt`:
  * new `*offdelay*='num'::` entry in EXTRA ARGUMENTS (line 45), with the
    `upscmd ... shutdown.return 45` example in an indented literal block;
  * new "INSTANT COMMANDS" section (line 57) documenting the optional seconds
    value, defaults and the accepted range 1..3600.
- `docs/man/upscmd.txt`:
  * SYNOPSIS (line 16) now shows `... 'ups' 'command' ['value']`;
  * new `'command'::` / `'value'::` entries in OPTIONS (`'value'` at line 74).
- `NEWS.adoc`: new bullet ``- `tripplitesu` driver updates:`` at line 351,
  inside the 2.8.6 section (after the riello bullet, before usbhid-ups).
- `offdelay` is already present in `docs/nut.dict` (line 2912); no change
  needed there.

## 6. Pipeline facts (core, unchanged - do not modify)

- `clients/upscmd.c` `do_cmd()`: sends `INSTCMD <ups> <cmd> <value>` when a
  value is given; `-h` already documents `[<value>]`.
- `server/netinstcmd.c` `net_instcmd()`: `numarg == 3` -> `cmdparam`, forwarded
  to the driver socket.
- `drivers/dstate.c` `sock_arg()` (~lines 1130-1230): parses
  `INSTCMD <cmdname> [<cmdparam>] [TRACKING <id>]`; `numarg == 3` -> cmdparam;
  `numarg == 5` with `TRACKING` -> cmdparam + id. Calls `main_instcmd()` first,
  then `upsh.instcmd(cmdname, cmdparam)`. If TRACKING was requested it replies
  `TRACKING <id> <status>\n` (see `send_tracking`, dstate.c:930).
- `main_instcmd()` (`drivers/main.c`) ignores `extra` and only handles
  `shutdown.default` / `driver.*`. `shutdown.default` calls
  `upsdrv_callbacks.upsdrv_shutdown()`, so it CANNOT receive a cmdparam and must
  rely on ups.conf (that is why `upsdrv_shutdown` keys off
  `offdelay_from_conf`).
- `STAT_INSTCMD_*` (`drivers/upshandler.h`): HANDLED=0, UNKNOWN=1, INVALID=2,
  FAILED=3, CONVERSION_FAILED=4.

## 7. Validation done in the devcontainer (all green)

Build:
- `make -j$(nproc)` full tree -> EXIT=0, 0 warnings, 0 errors.
- `drivers/tripplitesu` built clean; new strings present in the binary; `-h`
  lists `offdelay=<value>  Set shutdown delay, in seconds (default=10, range 1..3600).`
- `make distcheck-light` -> EXIT=0; inner testsuite TOTAL 6 / PASS 6 / FAIL 0;
  tarball `nut-2.8.5.1169.tar.gz` produced.
- `docs/man/tripplitesu.8` and `docs/man/upscmd.8` generated (EXIT=0) and pass
  `groff -man -Tutf8 -z` with no diagnostics.
- No non-ASCII bytes in any changed file (the 6 non-ASCII NEWS lines are
  pre-existing author names; the added bullet is pure ASCII).

`offdelay` from config, via pty (no hardware needed):

    PTY_TIMEOUT=6 python3 /tmp/nut_pty_test.py ./drivers/tripplitesu -x offdelay=abc
        -> "Invalid offdelay parameter: abc (expected 1..3600 seconds)", exit 1
    ... -x offdelay=0     -> "Invalid offdelay parameter: 0 (expected 1..3600 seconds)", exit 1
    ... -x offdelay=45    -> no offdelay fatal; keeps running (harmless)

`instcmd` cmdparam, via the simulated UPS (`/tmp/nut_ups_sim.py`). Column shows
the `SDA;<value>` frame observed on the serial link, for the default (no
offdelay = 10) vs `-x offdelay=45`:

    command                            default   offdelay=45   reply (TRACKING 1 <n>)
    shutdown.return 45                 SDA;45    SDA;45        0 HANDLED
    shutdown.return                    SDA;10    SDA;45        0 HANDLED
    shutdown.return 1                  SDA;1     SDA;1         0 HANDLED
    shutdown.return 3600               SDA;3600  SDA;3600      0 HANDLED
    shutdown.return 0045               SDA;45    SDA;45        0 HANDLED
    shutdown.return 0                  (none)    (none)        3 FAILED
    shutdown.return 3601               (none)    (none)        3 FAILED
    shutdown.return abc                (none)    (none)        3 FAILED
    shutdown.return -5                 (none)    (none)        3 FAILED
    shutdown.return 45x                (none)    (none)        3 FAILED
    shutdown.return +45                (none)    (none)        3 FAILED
    shutdown.reboot                    SDA;10    SDA;45        0 HANDLED
    shutdown.reboot 120                SDA;120   SDA;120       0 HANDLED
    shutdown.reboot.graceful           SDA;60    SDA;60        0 HANDLED
    shutdown.reboot.graceful 120       SDA;120   SDA;120       0 HANDLED
    test.battery.start (value ignored) (none)    (none)        0 HANDLED
    shutdown.default (upsdrv_shutdown) SDA;5     SDA;45        0 HANDLED

=> precedence `cmdparam > offdelay > default` confirmed; graceful keeps 60;
   invalid values fail with no serial frame; `upsdrv_shutdown()` keeps 5 s
   unless offdelay is in ups.conf.
The driver log confirms e.g.
`instcmd(shutdown.return): invalid delay value '0' (expected 1..3600 seconds)`.

## 8. Test harness (reusable) - no hardware needed

Why a pty: `upsdrv_initups()` calls `ser_open()` before reading the config, so
`-x port=/dev/null` fails at `tcgetattr` before reaching the offdelay check.
With a pseudo-terminal the driver opens it fine and then validates the config.

- `/tmp/nut_pty_test.py` (Appendix A): minimal pty runner for a driver; useful
  for config-validation tests. Prints child output + exit code (or TIMEOUT).
- `/tmp/nut_ups_sim.py` (Appendix B): full SmartOnline simulator + socket
  driver. It answers the driver's serial polls, then sends `INSTCMD ...` on the
  driver's Unix socket and reports the tracked status and the `SDA` values.

  Usage: `python3 /tmp/nut_ups_sim.py ./drivers/tripplitesu [-x offdelay=NN]`
  Driver output is also saved to `/tmp/nut_ups_sim_driver.log`.

If `/tmp` was wiped, recreate both files from Appendix A/B before rerunning.

Protocol quick reference (from the driver source):
- Request frame: `~00<type><3-digit-len><command><params>` where type is
  `P` = poll, `S` = set, e.g. POLL AVL -> `~00P003AVL`.
- Response: `~00D<3-digit-count><data>` (data is space-padded; the driver
  right-trims spaces), or `~00A` for a plain ACK (used for SET commands).
- Commands used at startup: `AVL`, `MNU`, `MOD`, `VER`, `RAT`, `LET`, `UID`,
  `TXV`, `VSN`, then updateinfo `STO`, `STB`, `STA`, `STI`, `TSR`, `ENV`.
- Driver socket path: `$NUT_STATEPATH/<driver>-<upsname>` =
  `/tmp/nut-state/tripplitesu-testups`; lines terminated by `\n`.
- `MNU` must answer exactly `Tripp Lite` or `init_comm()` fails.
- `LET` = number of outlet banks (use "2" to get `load.off`/`load.on`).
- SDA = shutdown delay (SET), SDR = shutdown restart (SET).

## 9. Decisions / open points (unchanged)

- Range 1..3600 s; `0` deliberately rejected (SDA "0" = cancel).
- `shutdown.reboot.graceful` keeps its own 60 s default, not affected by
  `offdelay`.
- `upsdrv_shutdown()` keeps 5 s unless `offdelay` is explicitly set in ups.conf.
- No new command names (`load.off.delay` / `load.on.delay`) in this change.
- `offdelay` is a VAR_VALUE (not RW), so it is not exposed/edited via `upsrw`.

## 10. Possible future work on `tripplitesu` (ideas, not started)

- Real SmartOnline hardware E2E (the old TODO #3; replaced here by the simulator).
- Optional seconds value for `load.off` / `load.on` (or new
  `load.off.delay` / `load.on.delay` command names).
- Optional seconds value for `test.battery.*` if the device supports it.
- Consider a `setvar`/`upsrw` path for the delay (would need persistence
  handling - the current design is deliberately stateless).
- Port the same cmdparam override idea to `drivers/tripplite.c` and
  `drivers/tripplite_usb.c` (as separate changes).

## 11. Process constraints (AGENTS.md)

- Human review + DCO; do NOT add a `Signed-off-by` on the contributor's behalf.
- Disclose AI assistance as required by the PR template; do not claim hardware
  acceptance that was not actually tested.
- Plain ASCII in sources/docs; keep each change focused; update Makefile.am /
  NEWS / dict only where required.
- Run `make distcheck-light` before upstreaming.

## Appendix A - `/tmp/nut_pty_test.py`

```python
#!/usr/bin/env python3
"""Run the tripplitesu driver against a pseudo-terminal (no hardware needed).

Usage: nut_pty_test.py <driver> [extra driver args...]
Prints the child's stdout/stderr, any bytes read from the serial master
side, and the child exit code (or 'TIMEOUT' if it keeps running).
"""
import os
import pty
import select
import subprocess
import sys
import threading
import time

TIMEOUT = float(os.environ.get("PTY_TIMEOUT", "8"))


def main():
    driver = sys.argv[1]
    driver_args = sys.argv[2:]

    master, slave = pty.openpty()
    slave_name = os.ttyname(slave)

    env = dict(os.environ)
    env.setdefault("NUT_STATEPATH", "/tmp/nut-state")
    env.setdefault("NUT_ALTPIDPATH", "/tmp/nut-altpid")

    cmd = [driver, "-s", "testups", "-D", "-x", "port=" + slave_name] + driver_args
    print("RUN: " + " ".join(cmd), file=sys.stderr)
    print("PTY slave: " + slave_name, file=sys.stderr)

    proc = subprocess.Popen(
        cmd, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, env=env)

    def drain():
        while True:
            if proc.poll() is not None:
                try:
                    r, _, _ = select.select([master], [], [], 0.2)
                    if not r:
                        break
                except OSError:
                    break
            try:
                r, _, _ = select.select([master], [], [], 0.2)
            except OSError:
                break
            if r:
                try:
                    data = os.read(master, 4096)
                except OSError:
                    break
                if not data:
                    break
                print("SERIAL> " + repr(data), file=sys.stderr)

    thread = threading.Thread(target=drain, daemon=True)
    thread.start()

    try:
        out, _ = proc.communicate(timeout=TIMEOUT)
        status = "CHILD_EXIT=%d" % proc.returncode
    except subprocess.TimeoutExpired:
        proc.kill()
        out, _ = proc.communicate()
        status = "TIMEOUT (still running after %.0fs, killed)" % TIMEOUT

    sys.stdout.write(out.decode("utf-8", "replace"))
    # give the drain thread a moment to flush late serial reads
    time.sleep(0.3)
    print("=== " + status, file=sys.stderr)


main()
```

## Appendix B - `/tmp/nut_ups_sim.py`

```python
#!/usr/bin/env python3
"""Simulate a Tripp Lite SmartOnline UPS over a pty and drive the driver's
instant commands through its Unix socket. No real hardware needed.

Usage: nut_ups_sim.py <driver> [extra driver -x args...]
"""
import os
import pty
import select
import socket
import subprocess
import sys
import threading
import time

DRIVER = sys.argv[1]
EXTRA = sys.argv[2:]

STATE = "/tmp/nut-state"
os.makedirs(STATE, exist_ok=True)

master, slave = pty.openpty()
slave_name = os.ttyname(slave)

env = dict(os.environ)
env["NUT_STATEPATH"] = STATE
env["NUT_ALTPIDPATH"] = "/tmp/nut-altpid"

cmd = [DRIVER, "-s", "testups", "-D", "-x", "port=" + slave_name] + EXTRA
print("RUN: " + " ".join(cmd), flush=True)

logf = open("/tmp/nut_ups_sim_driver.log", "w")
proc = subprocess.Popen(cmd, stdout=logf, stderr=subprocess.STDOUT, env=env)

frames = []
sda_values = []
lock = threading.Lock()

POLL_RESP = {
    "AVL": "1111111111",
    "MNU": "Tripp Lite",
    "MOD": "SmartOnline SIM",
    "VER": "1.2.3",
    "RAT": "230;0;230;0;0;0;0;0;0;200;180;250;240;0;48;0",
    "LET": "2",
    "UID": "SIM-0001",
    "TXV": "180;260",
    "VSN": "0",
    "STO": "0;500;0;2300;10;0;50",
    "STB": "0;0;0;0;0;0;270;0;25;100",
    "STA": "0;0;0;0",
    "STI": "0;500;2300",
    "TSR": "0",
    "ENV": "25;50;0;0;0;0",
}


def enc_data(s):
    b = s.encode()
    return b"~00D" + ("%03d" % len(b)).encode() + b


def handle(typ, payload):
    cmd3 = payload[:3].decode("ascii", "replace")
    with lock:
        frames.append((typ, payload.decode("ascii", "replace")))
    if typ == "P":
        resp = POLL_RESP.get(cmd3)
        os.write(master, enc_data(resp) if resp is not None else b"~00A")
    else:
        if cmd3 == "SDA":
            with lock:
                sda_values.append(payload[3:].decode("ascii", "replace"))
        os.write(master, b"~00A")


def reader():
    buf = b""
    while True:
        try:
            r, _, _ = select.select([master], [], [], 0.2)
        except (OSError, ValueError):
            break
        if r:
            try:
                d = os.read(master, 4096)
            except OSError:
                break
            if not d:
                break
            buf += d
        while len(buf) >= 7:
            if buf[:3] != b"~00":
                idx = buf.find(b"~00")
                if idx == -1:
                    buf = buf[-2:]
                    break
                buf = buf[idx:]
                if len(buf) < 7:
                    break
            ln = int(buf[4:7])
            if len(buf) < 7 + ln:
                break
            typ = chr(buf[3])
            payload = buf[7:7 + ln]
            buf = buf[7 + ln:]
            handle(typ, payload)


threading.Thread(target=reader, daemon=True).start()

SOCK = os.path.join(STATE, "tripplitesu-testups")
for _ in range(150):
    if os.path.exists(SOCK):
        break
    time.sleep(0.1)
time.sleep(2.0)  # let upsdrv_initinfo / updateinfo complete


def send_instcmd(args, tracking="1"):
    s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    s.connect(SOCK)
    line = "INSTCMD " + args
    if tracking:
        line += " TRACKING " + tracking
    s.sendall((line + "\n").encode())
    s.settimeout(4)
    data = b""
    try:
        while True:
            chunk = s.recv(4096)
            if not chunk:
                break
            data += chunk
            if b"\n" in data:
                break
    except socket.timeout:
        pass
    s.close()
    return data.decode("ascii", "replace").strip()


def run(label, args):
    with lock:
        start = len(sda_values)
    reply = send_instcmd(args)
    time.sleep(0.4)
    with lock:
        newsda = list(sda_values[start:])
    print("%-42s reply=%-22s SDA=%s" % (label, reply, newsda), flush=True)
    return newsda, reply


print("=== first frames seen ===", flush=True)
with lock:
    for f in frames[:12]:
        print("   ", f, flush=True)

print("=== INSTCMD tests ===", flush=True)
run("shutdown.return 45", "shutdown.return 45")
run("shutdown.return (no value)", "shutdown.return")
run("shutdown.return 1", "shutdown.return 1")
run("shutdown.return 3600", "shutdown.return 3600")
run("shutdown.return 0 (invalid)", "shutdown.return 0")
run("shutdown.return 3601 (invalid)", "shutdown.return 3601")
run("shutdown.return abc (invalid)", "shutdown.return abc")
run("shutdown.return -5 (invalid)", "shutdown.return -5")
run("shutdown.reboot", "shutdown.reboot")
run("shutdown.reboot 120", "shutdown.reboot 120")
run("shutdown.reboot.graceful", "shutdown.reboot.graceful")
run("shutdown.reboot.graceful 120", "shutdown.reboot.graceful 120")
run("test.battery.start (ignores value)", "test.battery.start")
run("shutdown.return 45x (trailing junk)", "shutdown.return 45x")
run("shutdown.return +45 (plus sign)", "shutdown.return +45")
run("shutdown.return 0045 (leading zeros)", "shutdown.return 0045")
run("shutdown.default (upsdrv_shutdown)", "shutdown.default")

print("=== shutting down driver ===", flush=True)
proc.terminate()
try:
    proc.wait(timeout=5)
except subprocess.TimeoutExpired:
    proc.kill()
logf.close()
print("driver exit code: %s" % proc.returncode, flush=True)
```
