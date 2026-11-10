# MEMORY - NUT `tripplitesu` driver work (handoff)

Handoff notes to continue development on the Tripp Lite SmartOnline driver
`drivers/tripplitesu.c` inside this devcontainer.

Last updated: session 9 (watchdog rework, `shutdown.default` auto-reboot).
Environment: Linux devcontainer (Debian bookworm) with gcc 12.2.0,
autoconf 2.71, automake 1.16.5, libtool 2.4.7.
Project version string in this tree: `2.8.5.1171-1171+gcd666a928`.

## 0. TL;DR - current status

The feature is IMPLEMENTED, BUILD-VERIFIED and RUNTIME-VERIFIED (against a
simulated UPS; no real hardware). Nothing from the previous TODO list is open.

- Precedence implemented:

      cmdparam (upscmd value)  >  offdelay (ups.conf)  >  built-in default

- Stateless by design: no RAM staging via `upsrw`, so no persistence problem.
- Real SmartOnline hardware E2E is still NOT done (no hardware available);
  a pty-based simulator reproduces the serial protocol instead (section 8).

### 0.1 Session 2 (current) - outlet commands, relay status, return delay

IMPLEMENTED, BUILD/`distcheck-light`-VERIFIED and RUNTIME-VERIFIED with the
extended pty simulator (`/tmp/nut_ups_sim2.py`, see section 12). Uncommitted at
the time of writing:

      drivers/tripplitesu.c    docs/man/tripplitesu.txt
      data/cmdvartab           NEWS.adoc

- Per-outlet commands: `outlet.n.load.off|on|cycle` (no value) -> `ROF|RON|RNF`
  with `"<n>;1"`.
- `load.off` / `load.on`: back to the upstream behaviour (only `ROF`/`RON` on
  every bank); any value is ignored.  `SDA`/`SDR` are used only by
  `shutdown.*` and `upsdrv_shutdown()`.
- `shutdown.reboot` / `shutdown.reboot.graceful`: value = delay before the
  off, and the restart delay is `delay + 1 second` (SDR is in whole minutes,
  so `set_start_delay()` rounds it up; see the Session 3 section: the device
  does NOT come back when the restart delay is 0).  `shutdown.return` uses
  `startdelay` for the return delay.
  `shutdown.stayoff` is now implemented
  (`ARB 2` + `SDA <delay>`) instead of being disabled in `#if 0`.
- New `startdelay` ups.conf parameter (minutes, default 1) and RW
  `ups.delay.start` (seconds), used by `shutdown.return` and by
  `upsdrv_shutdown()`.
- Buzzer: enabling/disabling is the RW variable `ups.beeper.status`
  (`enabled`/`disabled` -> `EDB1`/`EDB2`); `beeper.mute` remains an instant
  command (`BUZ2`).
- Auto-reboot (`ups.start.auto` / `ARB`): the shutdown commands change it as
  the naming scheme requires; the driver logs every change ("auto_reboot:
  ups.start.auto changed from ... to ...") and the man page points to `upsrw`
  for restoring a preferred value.  **No** ups.conf parameter for it (the
  user chose the minimal approach).
- New variables: `outlet.count`, `outlet.n.status` (from `SOL<n>`),
  `outlet.n.switchable`, `battery.date` (from `BRD` field 0).
- Power-on (boot) delay: RW variable `outlet.delay.start` <-> `UBD`
  (0..999 s, get/set only: NOT an instant command).  `ups.delay.start` stays
  bound to SDR (the return delay), following the meaning used by nutdrv_qx,
  huawei-ups2000, phoenixcontact_modbus and bicker_ser.
- Auto-reboot setting: RW variable `ups.start.auto` <-> `ARB` ("yes" = 1,
  "no" = 2; get/set only: NOT an instant command); see the bullet above for
  how the persistence of the device setting is handled.
- Watchdog: RW variable `ups.watchdog.status` <-> `WDG` ("enabled" = non-zero
  timeout, "disabled" = 0 seconds; get/set only).  The timeout (255 s by
  default) and the restart-alarm flag are preserved device settings.
- ECO mode: RW variable `input.eco.switchable` <-> `ECO` ("ECO" = 1,
  "normal" = 2; get/set only).  The vendor software only writes it, so the
  read is best-effort (see section 12).
- TXV: `input.transfer.low`/`input.transfer.high` were already RW; they are
  now validated against the device range and the ranges are published as
  `input.transfer.low.min`/`.max` and `input.transfer.high.min`/`.max`.
- BPA: new instant command `clear.fault.record` (action: sends `BPA0`).
- NOT verified on real hardware: the `RNF` outlet-cycle command (absent from
  the PowerAlert capture) and the `SOL` polarity ("0" = on, inferred from the
  capture transitions).

### 0.2 Session 3 (current) - real hardware round + reboot/return fixes

The user ran the driver against the real SUINT1500RTXL2Ua (the UPS is on the
host `codbox`, reached with
`socat -dddd pty,link=/tmp/ttyUPS,raw,echo=0,mode=666 TCP:codbox:5000`) and
reported ONE functional problem: `shutdown.reboot` did not perform a complete
reboot - it either only shut the UPS down (when it was running) or only
started it (when it was off).  Cause and fix (uncommitted, verified with
`/tmp/nut_ups_sim2.py`: "all checks passed"):

- The device only restarts the load when the restart delay (SDR, minutes) is
  LONGER than the shutdown delay (SDA, seconds).  With `SDR 0` an online
  device switches its outputs off and leaves them off, while an offline one
  is only started up - exactly the symptom above (firmware 16).
- `shutdown.reboot` / `shutdown.reboot.graceful` now send
  `reboot_start_delay(delay)` = `delay + 1` second (0 stays 0, as requested),
  which `set_start_delay()` rounds up to whole minutes: the smallest restart
  delay that lands behind the shutdown (SDR 1 for `shutdown.reboot 20`, SDR 2
  for `shutdown.reboot 60`).  A shutdown delay of 0 cannot reach the reboot
  commands (`offdelay` and the command value must be >= 1 s, because 0 means
  "cancel the scheduled shutdown").
- `shutdown.return` and `upsdrv_shutdown()` keep sending `startdelay`
  unchanged; the man page (and NEWS) now tell the user to choose a
  `startdelay` longer than the `offdelay` used with those.
- Also found and reverted a regression introduced by an earlier edit of this
  work: the `shutdown.return` branch had been renamed to `shutdown`, so the
  command - still advertised through `dstate_addcmd("shutdown.return")` -
  answered `STAT_INSTCMD_UNKNOWN` and sent nothing to the device.  Caught by
  the simulator, which expects the `SDR+SDA` frames.
- Still to confirm on the hardware (the devcontainer lost the `codbox` bridge
  during this session, so the last live round could not be repeated): that
  `shutdown.reboot 20` switches the loads off at ~20 s and brings them back at
  the end of the rounded-up restart delay (`SDR 1`, `SDA 20`), and what the
  device does with a non-zero restart delay when it is already off (it used to
  be started by `SDR 0`; `upsrw -s ups.delay.start=0` still writes that, with
  no shutdown delay).

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

## 12. Session 2 details - outlet control, relay status, return delay

Requested by the user: implement only the on/off/reboot commands for both the
whole UPS and the individual outlets, add the relay status / more readable
information, and add the off and return delays. Decisions taken:

- Outlet commands use the standard NUT names and take NO value
  (`outlet.n.load.off|on|cycle`).
- Only `load.off` / `load.on` accept a value; `shutdown.reboot*` untouched.
  (SUPERSEDED in session 3: `load.off`/`load.on` are relay-only again/values
  ignored, and the delay values belong to the `shutdown.*` commands.)
- Return delay = SDR (minutes, per the user's own notes), made configurable.
- `ups.delay.shutdown` is deliberately NOT exposed RW: SDA is an action, not a
  setting (writing 45 would shut the UPS down after 45 s). Only the return
  delay (SDR) is a real setting, so only it is exposed RW.

Code map (`drivers/tripplitesu.c`):
- `DEFAULT_STARTDELAY 60U` (seconds), `MIN_STARTDELAY 0U`,
  `MAX_STARTDELAY 3600U`, `MAX_STARTDELAY_MINUTES` + `static startdelay`.
- `set_start_delay(seconds)`: sends `SDR`, rounded up to whole minutes.
- `get_outlet_status()`: polls `SOL<n>` (0 = on, 1 = off) per bank and sets
  `outlet.n.status`; failures are debug-only, never `dstate_datastale()`.
- `get_battery_date()`: `BRD` field 0 -> `battery.date`; field 1 only debug.
- `parse_return_delay()`: 0..3600 s (0 is valid here, unlike `parse_delay`).
- `instcmd()`: outlet command table + `load.off`/`load.on` value handling.
  (SUPERSEDED in session 3: `load.off`/`load.on` no longer take values.)
- Buzzer: `instcmd()` sends `EDB1` / `EDB2` / `BUZ2`; the three commands are
  registered in `upsdrv_initinfo()`.  They take no value and, following the
  other drivers (belkin, bcmxcp), they do not use the POWERSTATE log macros.
  Evidence: `serial.txt` ("ebd2 disable buzz, ebd1 enable buzz", "buz2 mute")
  and `full-capture.txt` (`~00S004EDB2`, `~00S004EDB1`, `~00S004BUZ2`, twice
  each).  Note that the capture files are UTF-16, so plain `grep` finds
  nothing: use `iconv -f UTF-16LE -t UTF-8` first.
- Power-on delay (`UBD`, exposed as RW `outlet.delay.start`, 0..999 s):
  - It is a device *setting*, so it has no instant command: only `setvar()`
    (via `upsrw`) and a poll in `initinfo` (via `get_boot_delay()`), plus a
    read-back after writing so the variable shows what the device accepted.
  - Written zero padded (`%03u`, e.g. `UBD007`), like the vendor software.
  - Name rationale: `ups.delay.start` already means "return/restart delay
    after a shutdown" in NUT (SDR here), so the device power-on delay is
    published in the outlet collection as the main-outlet form
    (`outlet.delay.start` = all outlets); `outlet.<something>` unindexed is
    already used by the SNMP drivers (e.g. `outlet.current`).
  - Evidence: `serial.txt` ("ubd007 per seconds power on delay") and the
    capture (`~00P003UBD` polled 6 times, `~00S006UBD007`, `~00S006UBD000`).
  - If upstream prefers the reverse mapping (`ups.delay.start` = UBD, SDR
    config-only), it is a small change: swap the two exposures and keep the
    `ups.delay.start` name on whichever value the reviewers pick.
- Auto-reboot setting (`ARB`, exposed as RW `ups.start.auto`, "yes"/"no"):
  - Standard name: `nut-names.txt` defines `ups.start.auto` as "UPS starts
    when mains is (re)applied" and explicitly notes that shutdown commands
    must set it (shutdown.stayoff -> "no", shutdown.return -> "yes"); the
    driver's existing `auto_reboot(1)` calls already implement that rule, so
    the variable was added on top of them.
  - Mapping: ARB 1 -> "yes", ARB 2 -> "no" (the capture shows only these two
    values: `~00P003ARB` polls, `~00S004ARB1`/`~00S004ARB2`).
  - Implementation: `get_auto_reboot()` publishes the polled value;
    `auto_reboot()` now also refreshes the published value after a
    successful write (or when nothing had to be written), so `upsc` always
    agrees with the device; `setvar()` accepts "yes"/"no" case-insensitively
    and returns STAT_SET_FAILED otherwise (huawei-ups2000 does the same,
    with STAT_SET_INVALID).
  - Poll-before-write is kept from the original code, so an already correct
    setting produces no device write (verified in the harness).
- Watchdog (`WDG` -> RW `ups.watchdog.status`, "enabled"/"disabled"):
  - Device format: `<timeout in seconds>;<restart alarm flag>`; the capture
    shows only `WDG0;2`, `WDG255;2` and `WDG255;1` (`~00P003WDG` is polled,
    so the get works).  `serial.txt`: "wdg per whatchdog con 255 seconds",
    "2 restart alarm disable, 1 enable".
  - `nut-names.txt` only defines the status for watchdogs
    (`ups.watchdog.status`, enabled/disabled; `tripplite-hid` publishes a
    seconds countdown there), so the timeout and the flag are NOT exposed as
    separate variables: they are preserved from the device and the timeout
    defaults to 255 s (the value the vendor software used) when the watchdog
    is armed and no timeout was ever reported.
  - UNVERIFIED on hardware: the mapping "enabled = non-zero timeout" (the
    alternative reading would be the restart-alarm flag).  Confirm with
    `upsc` while arming/disarming from the vendor software.
- ECO mode (`ECO` -> RW `input.eco.switchable`, "ECO"/"normal"):
  - Name taken from nut-names ("Input High Efficiency (aka ECO) mode switch
    (opaque string)", example `normal`), which is also what `mge-hid` fills
    with "normal"/"ECO".
  - The vendor software only ever WRITES ECO (never polls it in the
    capture), so the get is best-effort: the driver publishes "normal" as an
    assumption and then updates the value on every successful read (startup
    and after a set).  On a device that cannot report it, the value reflects
    what the driver wrote.
- TXV: get/set already existed (`input.transfer.low`/`high`); this session
  added validation (numeric, inside the range reported by the device, with a
  sanity range of 1..999 V when the device reports none), published the
  ranges as `input.transfer.low.min`/`.max` and
  `input.transfer.high.min`/`.max`, and fixed a copy-paste bug that used
  `max_low_transfer` as the gate for the *high* range enums.
- BPA -> instant command `clear.fault.record` ("Clear fault/error record",
  the closest recorded name; it lives in the vendor-dependent part of
  nut-names) which sends `BPA0`, matching the capture (`~00S004BPA0` =
  "bpa per reset bypass reason" in `serial.txt`).  The reason value itself is
  deliberately not published (not requested, and its values 01/00/99/200 are
  undocumented).
- `setvar()`: `ups.delay.start` (RW) -> `startdelay` + `SDR`.
- `upsdrv_makevartable()`/`upsdrv_initups()`: `startdelay` is in MINUTES.
- `upsdrv_initinfo()`: `outlet.count`, `outlet.n.switchable`, one command set
  per bank, `ups.delay.start` (RW, ST_FLAG_STRING, aux 4), `battery.date`.
- `upsdrv_shutdown()`: SDR now via `set_start_delay(startdelay)` (was "1").
- `load.off` / `load.on` are now registered even with 0 outlet banks (they also
  act on the whole UPS with a value); without a value they return FAILED when
  no bank is known.  (SUPERSEDED in session 3: back to the upstream
  conditional registration, i.e. only when outlet banks are reported.)

Protocol notes (from `full-capture.txt`): `ROF/RON<n>;1` switch one bank,
`SOL<n>` reports its relay, `SDA<sec>` schedules a shutdown and `SDR<min>`
schedules the return. `RNF` (relay cycle) is defined in the driver but never
appears in the capture: it is used for `outlet.n.load.cycle` on the original
author's word, and is NOT hardware-verified.

Tests run (all green, no hardware): `/tmp/nut_ups_sim2.py <driver>` (extended
simulator + socket-driven checks: command list via `DUMPALL`, variables via
`DUMPVALUE`, `SET` for the RW variables (`ups.delay.start`,
`outlet.delay.start`, `ups.start.auto`, `ups.watchdog.status`,
`input.eco.switchable`, `input.transfer.low/high`), per-bank frames, buzzer and
`clear.fault.record` frames, invalid values, no-op writes
(poll-before-write), relay status transitions, device settings that cannot be
reported back (`eco_readable = False`), SOL silence without datastale). Also
`SIM_STARTDELAY_MIN=3 ... -x startdelay=3` and pty parameter checks
(`startdelay=abc|61` -> fatal, `0|2` -> keeps running). Note for the harness:
the driver broadcasts `SETINFO` notifications on the socket, so replies must be
read up to the `TRACKING` line, not just to the first newline.

Known open points: `SDR 0` for the reboot commands needs hardware verification
(the capture is ambiguous: `SDR00000` at 16:15 brought the UPS back on after a
front-panel off, but a `SDR00000` at 15:59 did not revive it immediately);
`shutdown.stayoff` was disabled upstream with "doesn't seem to work" and now
sends `ARB 2` + `SDA <delay>`, also to be verified on hardware;
`outlet_banks` is still unbounded if the device reports garbage.

CLOSED in session 4 (live read-only test against the real UPS): the earlier
worry that the STB field layout differs on the user's SUINT1500RTXL2Ua was a
capture artifact - the PowerAlert capture displays only 32 of the 34 declared
bytes of the STB reply, and the real reply has one more leading field, which
is why the visible text looked shifted.  The driver's mapping is correct:
`battery.charge` 100, `battery.voltage` 54.8 V and `battery.temperature`
34 C were read from the device (see the session 4 notes below).

### Session 3 refinements (from user review)

- `load.off`/`load.on` reverted to the upstream relay-only behaviour (any
  value is ignored); the `SDA`/`SDR` handling added to them in session 2 was
  removed, and they are registered again only when the device reports outlet
  banks (upstream shape).
- Buzzer: the `beeper.enable`/`beeper.disable` instant commands were replaced
  by the RW `ups.beeper.status` variable (enabled/disabled), because it is a
  device setting; `beeper.mute` stays a command (`BUZ2`), because it is a
  one-shot action.  Precedent: `belkinunv.c` publishes `ups.beeper.status`
  RW with the same enum values.
- Reboot commands: the value is the delay before the off, and the return is
  `SDR 0` (upstream used a hardcoded `SDR 1`); `shutdown.return` keeps
  `startdelay`.
- `shutdown.stayoff` implemented: upstream had BOTH its handler and its
  `dstate_addcmd()` inside `#if 0 /* doesn't seem to work */`, so the command
  was neither advertised nor handled; now it sets `ARB 2` and `SDA <delay>`.
- ARB persistence: no new ups.conf parameter (user decision).  `auto_reboot()`
  logs every change with the previous value, and the man page documents
  restoring a preferred value with `upsrw -s ups.start.auto=...`.
- No "start only" command was added (user decision: it is handled with the
  reboot commands on their side) and nothing about it was added to any
  documentation.

### Session 4 - live read-only validation (real UPS via socat pty)
The user provided `/tmp/ttyUPS` (symlink to a socat pty to the real
SUINT1500RTXL2Ua) and asked for the absolute minimum: **poll/get only, no set
and no command**.  The driver was started with a dedicated state path
(`NUT_STATEPATH=/tmp/nut-live`, upsname `livesu`) and queried with
`/tmp/nut_ro_query.py`, which only sends `DUMPALL`/`DUMPVALUE`.  Audit and
evidence:

- 0 write frames for the whole session (`grep -c 'bytes sent [~00S'` = 0,
  including the clean SIGTERM exit): initinfo/updateinfo only poll, confirmed
  both by code inspection and by the log.
- Real device values read: `ups.status`=OFF, `ups.model`=SUINT1500RTXL2Ua,
  `ups.firmware`=16, `ups.load`=0, `input.voltage`=233.9,
  `input.frequency`=50.0, `input.transfer.low`=130, `input.transfer.high`=275
  with the ranges 130..138 / 267..275 (so the new TXV validation bounds are
  real), `battery.charge`=100, `battery.voltage`=54.8, `battery.temperature`=34,
  `battery.date`=20160912, `outlet.count`=2, `outlet.1/2.status`=on,
  `ups.start.auto`=yes, `ups.watchdog.status`=disabled,
  `outlet.delay.start`=0, `ups.test.result`="No test performed".
- Raw polls that worked: `LET`->2, `BRD`->20160912;20210912,
  `ARB`->1, `UBD`->000, `WDG`->000;2, `TXV`->130;275, `SOL1/2`->0/0,
  `STO`->7;000;... (mode 7 = OFF), `STB`->0;0;0;00000;1049;000;0548;;034;100.
- Polls the device **rejects with `~00R`** (not supported for reading):
  `ECO`, `EDB`, `VSN` (input.sensitivity) and `UID`.  `do_command()` returns
  -1 for anything that is neither `~00D...` nor `~00A`, so the new
  `input.eco.switchable` and `ups.beeper.status` keep their documented
  assumed/last-written values and nothing is marked stale: this validates the
  "best-effort read" design and must be stated in the PR (those two values are
  NOT device readings on this model).
- Nuance for `outlet.n.status`: with the whole UPS in the OFF state both banks
  report `SOL=0`, which we publish as `on` - the variable describes the bank
  relay state, not whether the output actually carries voltage.


### Session 5 - RNF live test and failure reporting

The user asked to try an outlet cycle with `RNF` on the live UPS (still
read-only otherwise).  One single frame was written for the whole session:

    ~00S006RNF1;1   ->  ~00R   (protocol rejection, not the ~00A ACK)

- `RNF` is NOT supported by the SUINT1500RTXL2Ua (fw 16) - which explains why
  the PowerAlert capture never used it (it cycles banks with `ROF` + `RON`).
  Nothing was switched: `SOL1` stayed 0 (`outlet.1.status` = on),
  `STA` (alarms), `STO` (mode 7 = OFF) and `ups.load` were unchanged.
- The driver answered `TRACKING 7 0` (HANDLED) for that command: `instcmd()`
  ignored the return value of `do_command()`, which is negative for both a
  rejection (`~00R`) and a timeout.  Fixed: `send_set_command()`/
  `send_set_or_fail()` now turn such frames into `STAT_INSTCMD_FAILED`, in
  every branch (load.*, outlet.*, beeper.mute, clear.fault.record,
  shutdown.*, including the auxiliary `auto_reboot()` write, whose failure
  now fails the command; an *unreadable* `ARB` still leaves the setting
  alone and does not fail the command, to avoid false failures on models that
  do not report it).  Failures are logged as
  "device rejected or did not answer [<frame>]".
- The `outlet.n.load.cycle` command is kept registered (other models may
  support `RNF`) and the man page now says that it fails if the device
  rejects it; the man page also documents that a failed command had at least
  one of its operations refused or unanswered (a partial effect is possible).
- Harness: the simulator can now answer `~00R` (`reject_frames`) or stay
  silent (`silent_frames`) for chosen SET parameters; new tests cover a
  rejected cycle, a rejected bank inside `load.off`, an unanswered
  `clear.fault.record` (3 s timeout) and the same command succeeding again.
- Known gap (not changed, user's scope): `setvar()` still reports the
  variable writes without checking the frames (it re-reads the value
  afterwards, which reveals failures only when the device can report it).

## Session 6 (act, devcontainer) - cleanup round and full validation

Decisions taken by the user and implemented:

- `offdelay` and the other defaults keep their historical values; the driver
  shutdown handling still sends the historical 5 seconds unless `offdelay`
  is set in ups.conf (now documented in the man page).
- `shutdown.return` does NOT ask for a timed return any more: it sets ARB=1,
  sends `SDR 0` (clearing any return delay left over by an earlier command)
  and then `SDA <delay>`.  `shutdown.stayoff` does the same with ARB=2.
  Only the two reboot commands set a restart delay (`delay+1` second,
  rounded up to whole minutes), and `upsdrv_shutdown()` keeps `startdelay`.
- `startdelay` stays in minutes (the device SDR unit); the man page now says
  that out-of-range values are rejected and that writes through
  `ups.delay.start` (seconds) are rounded up.  The PR notes should mention
  the seconds variant as a possible follow-up.
- `outlet.n.load.cycle` was REMOVED again (`RNF` stays a define only, as in
  upstream), because the tested SUINT1500RTXL2Ua rejects it.
- The working notes (`MEMORY.md`, `Configurazione-...md`, `full-capture.txt`,
  `serial.txt`, `.devcontainer/`) are untracked as of this session
  (`git rm --cached`) and only live in the working tree.

Other fixes in this round:

- `upsdrv_initinfo()`: the outlet bank count from `LET` is validated against
  the new `MAX_OUTLET_BANKS` (16); a negative or huge value no longer starts
  billions of iterations or an endless `SOL<n>` polling storm.
- `setvar("ups.delay.start")` only publishes the value once the device has
  accepted the `SDR` write, and reports `STAT_SET_FAILED` otherwise.
- All the remaining SETs report failures (shutdown.stop, test.battery.*,
  boot delay, watchdog, ECO, beeper, TXV, VSN, UID) instead of pretending.
- The four `shutdown.*` commands share one table-driven branch, the delay
  parsers were folded into `parse_uint_range()`, and the dead `WDG_AVAILABLE`
  define was dropped.

Validation of this tree (all green; no hardware):

    gcc -fsyntax-only -Wall -Wextra -Wformat=2 ... drivers/tripplitesu.c   # clean
    make -C drivers tripplitesu                                            # clean
    make -j"$(nproc)"                                        # exit 0, no new warnings
    make -C docs/man tripplitesu.8 upscmd.8 && groff -man -Tutf8 -z ...     # exit 0
    make -C docs/man tripplitesu.txt-spellchecked                           # exit 0
    make distcheck-light                                                    # completed OK
    python3 /tmp/tsu_test.py ./drivers/tripplitesu                          # 44 passed
    python3 /tmp/tsu_test.py ./drivers/tripplitesu --let N                  # 6 passed

Two gotchas found while validating (both fixed/avoided): the NUT docs check
refuses an AsciiDoc admonition whose first sentence starts with a lower-case
letter ("WARNING: enabling ..." -> "WARNING: Enabling ..."), and the first
`make distcheck-light` failed only because of that.

`/tmp/tsu_test.py` is this session's pty+simulator harness: it uses a
per-pid state directory and log file (concurrent runs used to clobber each
other), checks the exact serial frames of every instant command and the
TRACKING statuses, the rejection paths (`~00R`), the `LET` clamp and the
published variables/commands.  `/tmp/tsu_driver.<pid>.log` has the driver
log of the last run.

Still to confirm on real hardware: the reboot timing after `SDR delay+1`,
`SDR 0` on `shutdown.return`/`shutdown.stayoff` (does the load really stay
off?), the watchdog "enabled = timeout != 0" mapping and the persistence of
`ups.start.auto`.

## Session 7 - `command_delay` redesign (auto mode + minimum interval)

Requested by the user after re-reading PR #3273: keep `command_delay` in
ups.conf with three meanings, and decide the default with the maintainer.

### Semantics implemented (ups.conf `command_delay`, milliseconds)

- `0` - no pacing at all (also the compiled-in default: `DEFAULT_COMMAND_DELAY`,
  so upstream behaviour does not change until the maintainer picks a value);
- `-1` - automatic: start with no pacing, then raise the interval to
  `DEFAULT_AUTO_COMMAND_DELAY` (1000 ms) once, on the first read timeout of a
  POLL command, and log a warning.  An explicit value (0 or positive) is never
  changed; SET timeouts are ignored (an unanswered SET may have been carried
  out all the same);
- `x > 0` - keep at least x ms between two commands.  This is a *minimum
  interval measured from the previous command* (`pace_before_command()` +
  `note_command_sent()`, `gettimeofday`), not a blind sleep before every
  command: an idle link or a slow answer does not pay for it, and a user-set
  value is never overridden.  `ser_flush_io()` now happens after the wait.
- Parsing: `str_to_long_strict()` with range `-1..MAX_COMMAND_DELAY` (60000),
  `fatalx` message in the `offdelay`/`startdelay` style; this also removes the
  old `atol()` (trailing junk accepted) and the `(useconds_t)` wrap risk.

### Measured evidence (why 1000 ms and not 100)

From `full-capture.txt` (PowerAlert serial capture, 571 exchanges; the file is
UTF-16, needs `iconv -f UTF-16LE`): the answer to a command arrives in 9-16 ms,
but the vendor's own software never sends two commands closer than 0.760 s,
typically 2.2-3.0 s apart.  So the device has never been driven at the pace
our driver uses, and the "1 second" is an approximation of the vendor pace.

Measured with `/tmp/tsu_pace_test.py` (new, see below), 25 s per run:

    command_delay    commands   gaps min/median      notes
    0                122        107 / 121 ms         no escalation
    100              106        171 / 224 ms         no escalation
    250              74         230 / 375 ms         no escalation
    -1 (+silent STI) 42         123 / 143 ms then    log warning + 1133 ms
                                3148 (timeout)       after the escalation

Note the driver already has a "natural" gap of ~110-120 ms: `ser_send_pace()`
writes at 10 ms per byte, so a 10-byte frame takes ~100 ms to reach the
device.  The measured gap is therefore `command_delay + ~110 ms`: with
`command_delay=1000` the device actually sees ~1.11 s between commands, which
is above the fastest pace the vendor ever used (0.76 s) but below its typical
2.2-3.0 s; `command_delay=1500` (~1.6 s) is what it takes to stay inside the
vendor's observed range at all times.

### Validation done (this session)

- `make -C drivers tripplitesu`: clean build, no warnings; `-h` shows the new
  help text.
- `nut_pty_test.py`: `abc`, `-2`, `60001` -> fatal with the expected message;
  `0`, `-1`, `250` keep running.  With `-1` the warning is logged on the first
  timeout (pty with no device answering, at ~3 s).
- `/tmp/tsu_test.py` (session 6 harness): 44 passed, 0 failed - with the
  default and with `-x command_delay=100`.
- A/B check with the outdated `/tmp/nut_ups_sim2.py`: it reports the same 9
  failures with and without this change (`outlet.n.load.cycle` and the
  `shutdown.return`/`shutdown.stayoff` expectations predate the session 6
  cleanup), so they are harness drift, not a regression.
- `docs/man/tripplitesu.8` regenerated and `groff -man -Tutf8 -z` clean;
  `scripts/augeas/nutupsconf.aug.in` regenerated with the generator (it still
  lists `command_delay`; note the file is gitignored and is NOT built by a make
  rule, run `cd scripts/augeas && python3 gen-nutupsconf-aug.py` after
  `maintainer-clean`).
- `make distcheck-light`: see the tail of `/tmp/distcheck_light.log`.

### Harness additions (in /tmp, not in the repo)

- `/tmp/tsu_pace_test.py <driver> <command_delay> [seconds] [silent_poll]`:
  focused pacing measurement; prints gap statistics and the escalation lines.
- `/tmp/nut_ups_sim2.py` grew `SIM_SILENT_POLL=<cmd>` and
  `SIM_SILENT_POLL_COUNT=<n>` (leave one kind of POLL unanswered n times) plus
  a timestamped frame list and a pacing report at the end.

### Open point for the maintainer

Which default: `0` (no behaviour change, current choice), `-1` (self-fixing,
no cost on healthy devices) or a fixed value (1000, or 1500 to stay inside the
vendor cadence).  Flipping it is one line (`DEFAULT_COMMAND_DELAY`) plus the
man page sentence.

## Session 8 (act, devcontainer) - disarm the watchdog on graceful exit

Goal (user): the watchdog must end up disabled when the driver makes a
graceful exit (e.g. for a package update / driver restart), instead of being
left armed and power cycling the load as soon as the device stops hearing
from us.

Implementation (`drivers/tripplitesu.c` only):

- `set_watchdog()` now records the setting the device accepted: on a
  successful `WDG` write it updates `watchdog_seconds` and sets
  `watchdog_known`, so the exit path knows the watchdog is armed even when
  the read-back fails.  The static comment for `watchdog_seconds` changed
  from "last reported by the device" to "currently set on the device".
- New `disable_watchdog_on_exit()`: when `watchdog_seconds` is non-zero it
  logs `disable_watchdog_on_exit: disabling the watchdog before exiting`
  (LOG_INFO) and calls `set_watchdog(0)`, i.e. the frame `WDG0;<flag>`: the
  device settings are preserved (timeout 0, restart-alarm flag last seen).
- `upsdrv_cleanup()` calls `disable_watchdog_on_exit()` before
  `ser_close()`.  That hook is reached through `atexit()` in
  `drivers/main.c` on every graceful exit: SIGTERM/SIGINT/SIGQUIT (handled
  signal -> `exit_flag` -> `exit()` -> atexit), `driver.exit`, and the end
  of a `-d` data dump.  A crash or a SIGKILL never reaches it, so the
  watchdog keeps covering a driver that dies without unwinding.

Nothing is sent when the watchdog is already off, or when the device never
reported it (`watchdog_seconds == 0`): that keeps the Session 4 property
"0 write frames on SIGTERM" on the real UPS, whose `ups.watchdog.status` is
disabled.

Validation (devcontainer, no hardware):

- `make -C drivers tripplitesu` clean, no new warnings.
- New harness `/tmp/tsu_wdg_exit_test.py <driver>` (pty UPS with `WDG` in the
  device dict): A `WDG=255;2` at startup -> `('WDG','0;2')` on SIGTERM;
  B `WDG=000;2` -> no frame on SIGTERM; C arm with `SET ups.watchdog.status
  enabled` (`WDG255;2`) then SIGTERM -> `WDG0;2`.  4 passed, 0 failed.
- Existing `/tmp/tsu_test.py ./drivers/tripplitesu` still 44 passed, 0 failed
  (its simulated WDG is `000;2`, so no extra frame is sent); `--let N` PASS.
- `make -C docs/man tripplitesu.8` OK.

Docs: `docs/man/tripplitesu.txt` (`ups.watchdog.status` section) and
`NEWS.adoc` (tripplitesu bullet) describe the automatic disarm and its
limits.

Unverified on hardware: that `WDG 0;<flag>` really disarms a device (the
"enabled = non-zero timeout" mapping itself is still unconfirmed, see the
live-validation notes above).

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

## Session 6 - device settings are read from the UPS (no assumed values)

Session 6 reworked how the driver reports device settings, after the
session-4 observation that the real SUINT1500RTXL2Ua rejects the `ECO`,
`EDB`, `VSN` and `UID` polls:

- New read-only `input.transfer.reason`, published from the `BPA` poll (the
  raw two-digit code; its meaning is still undocumented) on every update
  cycle.  The existing `clear.fault.record` instant command still clears the
  recorded reason on the device (`BPA0`).  `netxml-ups` uses the same
  variable name for the same idea.
- The six "device setting" getters (`UBD`/`outlet.delay.start`,
  `ARB`/`ups.start.auto`, `WDG`/`ups.watchdog.status`,
  `ECO`/`input.eco.switchable`, `EDB`/`ups.beeper.status`, `BPA`) now log
  `LOG_WARNING` (was `upsdebugx(2)`) when the device does not answer or
  answers nonsense, and leave the previously published value in place.
  Nothing is marked stale: `dstate_datastale()` is not called for them,
  because they are auxiliary information (the staleness flag is global,
  not per-variable).
- `upsdrv_initinfo()` no longer publishes any assumed value: the variable
  and its RW flags/aux are only set after a successful read, so a device
  that cannot report a setting (e.g. ECO/EDB on the SUINT1500RTXL2Ua)
  simply does not expose it and `upsrw` cannot offer it.  The `bootdelay`
  static and the `DEFAULT_BOOTDELAY` macro became dead with this and were
  removed.
- `auto_reboot()` (also used by the shutdown handler) re-applies the RW
  flags after its own `dstate_setinfo()`, so `ups.start.auto` becomes
  settable even when the setting had not been read at startup.
- `set_watchdog()` warns when it arms the watchdog without ever having read
  the timeout from the device (defensive: `ups.watchdog.status` is normally
  only published after a successful `WDG` read).
- Documented in `docs/man/tripplitesu.txt` (shared note about device
  settings, new `input.transfer.reason` entry, reworded
  `clear.fault.record`) and in `NEWS.adoc`.
- Validation: the pty harness `/tmp/tsu_sim.py` + `/tmp/tsu_test.sh` runs
  three scenarios (device answers everything / answers once then rejects /
  never answers) against a real `upsd` + `upsc`/`upsrw`/`upscmd` stack:
  34 checks, all passing.  Note the pty delivers driver output a byte at a
  time, so the harness waits for `ups.status` instead of sleeping.
- Build-tree note: `docs/man/Makefile.in` and `include/Makefile.in` in this
  checkout were stale (`generic_gpio.txt` vs `generic_gpio_libgpiod.txt`,
  and `nut-pragmas-*.h` missing from the dist tarball), which made
  `make distcheck-light` fail before this work as well.  Repaired by
  re-running `automake`/`autoreconf` (generated files, not tracked) - the
  own diff stays limited to the driver, its man page and `NEWS.adoc`.


## Session 9 - watchdog rework and `shutdown.default` auto-reboot

Two related changes in `drivers/tripplitesu.c` (plus its man page, `NEWS.adoc`,
`data/cmdvartab` and `docs/nut-names.txt`), all still uncommitted:

- Watchdog is driven through variables only, with a cached setting:
  `ups.watchdog.timeout` (0..255 s, 0 disables) and `ups.watchdog.alarm`
  (1 = restart alarm enabled, 2 = disabled) are the two fields the device
  stores (the `WDG` payload is the combined `"<timeout>;<flag>"`),
  `ups.watchdog.status` is their derived enabled/disabled view, and
  `reset.watchdog` is now parameterless: it re-sends the setting the driver
  has, and fails when the device never reported one, so a kick can never arm
  a watchdog with assumed values.  `upsdrv_cleanup()` disarms an armed
  watchdog on an intentional exit, so a driver restart (package upgrade) does
  not leave a watchdog that would power cycle the load.
- `load.off`/`load.on` stop at the first frame the device rejects (multi-frame
  commands address outlets 1-based), as the shutdown commands already did.
- **`shutdown.default` always turns auto-reboot on** (`ARB 1`), so the load is
  powered again - all outlets back on - once mains is (re)applied.  The
  `shutdown_autoreboot` ups.conf parameter from the earlier draft of this work
  was REMOVED at the user's request (a shutdown that leaves the load off is
  not wanted); `-x shutdown_autoreboot=...` is now refused as an unknown
  variable (main.c prints "is not a valid variable name", with its usual
  `exit(EXIT_SUCCESS)`), and `scripts/augeas/nutupsconf.aug.in` (gitignored,
  regenerated by `make dist`) no longer lists it.
- What stays configurable is `shutdown_mode=continue|abort` (default
  `continue`): best effort keeps going when ARB cannot be confirmed, `abort`
  stops before the load is switched off (fail safe).  The operations are ARB
  and SDA only.
- The handler no longer sends `startdelay` (SDR): a return delay stored on the
  device is left alone (documented, with "write 0 to `ups.delay.start`" for
  the stay-off case); `startdelay` now only seeds `ups.delay.start`.
- Note: `main_instcmd()` ignores the `value` of `shutdown.default`, so the
  delay of that command (and the choice of `shutdown_mode`) can only come from
  `ups.conf` - hence the parameter.
- Validation: `/tmp/tsu_wdg_rw_test.py` against the pty simulator - 93 passed,
  0 failed (watchdog read/write/kick/refused values, multi-frame fail-fast,
  exit-disarm, best effort vs fail safe, removed parameter refused).  Driver
  build clean, `make -C drivers check` clean, man page and generated
  `tripplitesu.8` regenerated (no `shutdown_autoreboot` anywhere),
  `make spellcheck` (whole tree, including `NEWS.adoc`) clean, and
  `make distcheck-light` logged to `/tmp/tsu_dc2.out`.
- `docs/nut.dict` is back to its committed content: the only text that needed
  the `autoreboot` entry was the removed parameter name.

