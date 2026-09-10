# MEMORY - Tripp Lite `tripplitesu` shutdown delay / cmdparam work

Handoff notes to continue development inside a devcontainer.

## Goal

Implement per-invocation shutdown delay for the NUT Tripp Lite SmartOnline
driver `drivers/tripplitesu.c`, with strict precedence:

    cmdparam (upscmd value)  >  offdelay (ups.conf)  >  built-in default

Stateless by design: no RAM staging via `upsrw`, so no persistence problem.

## Key finding (scope)

The `instcmd()` code quoted in `Configurazione-Timeout-UPS-Tripp-Lite-in-NUT.md`
(with `TSU_SHUTDOWN_ACTION`, `ups.outlet_banks`, `do_command(SET, RELAY_OFF...)`)
is `drivers/tripplitesu.c`, NOT `drivers/tripplite.c`.

Status of the other two drivers (untouched by this work):
- `drivers/tripplite.c`: already reads `offdelay`/`startdelay`/`rebootdelay`
  from ups.conf and exposes RW `ups.delay.*`; only a CLI cmdparam override is
  missing there.
- `drivers/tripplite_usb.c`: already has `offdelay` + `ups.delay.shutdown`;
  `soft_shutdown()` uses `offdelay`; `startdelay`/`rebootdelay` are under `#if 0`.

Decision (user-approved): patch ONLY `drivers/tripplitesu.c`.

## Pipeline already supports cmdparam (no core changes needed)

- `clients/upscmd.c` `do_cmd()`: sends `INSTCMD <ups> <cmd> <value>` when a
  value is given; `-h` already documents `[<value>]`.
- `server/netinstcmd.c` `net_instcmd()`: `numarg == 3` -> `cmdparam`, forwarded
  to the driver socket.
- `drivers/dstate.c` (~lines 1130-1180): parses
  `INSTCMD <cmdname> [<cmdparam>] [TRACKING <id>]`, calls `main_instcmd()`
  first, then `upsh.instcmd(cmdname, cmdparam)`.
- `main_instcmd()` (`drivers/main.c:1060`) ignores `extra` and only handles
  `shutdown.default` / `driver.*`, so it does not swallow driver commands.
  NOTE: `shutdown.default` (used by upsmon / `-k`) cannot receive a cmdparam,
  so that path must rely on ups.conf.

## Changes already done (working tree, uncommitted)

Modified files: `drivers/tripplitesu.c`, `docs/man/tripplitesu.txt`,
`docs/man/upscmd.txt`, `NEWS.adoc`.

`drivers/tripplitesu.c`:
- header comment: added `offdelay` to the ups.conf parameters list; added a note
  about the optional delay argument of the shutdown commands.
- new defines after `MAX_RESPONSE_LENGTH`: `DEFAULT_OFFDELAY 10U`,
  `MIN_OFFDELAY 1U`, `MAX_OFFDELAY 3600U` (0 excluded: SDA "0" cancels a
  scheduled shutdown).
- new statics: `offdelay = DEFAULT_OFFDELAY`, `offdelay_from_conf = 0`.
- new helpers before `instcmd()`:
  - `parse_delay()` using `str_to_uint_strict()` (rejects empty, `+`/`-`,
    whitespace, trailing junk) plus range check;
  - `get_shutdown_delay(cmdname, extra, fallback, &delay)` implementing the
    precedence and logging `LOG_ERR` on invalid input.
- `instcmd()`: removed `NUT_UNUSED_VARIABLE(extra)`; `shutdown.reboot` and
  `shutdown.return` use `offdelay` (or cmdparam); `shutdown.reboot.graceful`
  keeps its own 60 s default unless a cmdparam is given; invalid cmdparam ->
  `STAT_INSTCMD_FAILED` BEFORE any serial traffic. `load.off`/`load.on`,
  `shutdown.stop`, `test.battery.*` unchanged.
- `upsdrv_shutdown()`: SDA delay = `offdelay` only if `offdelay_from_conf`
  (i.e. set in ups.conf), otherwise the historical 5 s -> no behavior change
  for existing configurations.
- `upsdrv_makevartable()`: added the `offdelay` var with help text.
- `upsdrv_initups()`: reads/validates `offdelay` with `str_to_uint_strict` +
  range, `fatalx()` if invalid (same style as the existing `command_delay`).

Docs:
- `docs/man/tripplitesu.txt`: new `offdelay` entry in EXTRA ARGUMENTS + new
  "INSTANT COMMANDS" section (optional seconds value, defaults, range 1..3600).
- `docs/man/upscmd.txt`: SYNOPSIS now shows `['value']`; added `'command'` and
  `'value'` entries in OPTIONS.
- `NEWS.adoc`: new 2.8.6 bullet `- tripplitesu driver updates:` inserted after
  the riello bullet and before the usbhid-ups bullet.

## Validation already done (macOS, build run by the user)

- `./autogen.sh && ./configure && make -j8` completed by the user with no errors.
- Binary contains the new strings (proof the patch is compiled in):
  `Set shutdown delay, in seconds (default=%u, range %u..%u).`
  `Invalid offdelay parameter: %s (expected %u..%u seconds)`
  `instcmd(%s): invalid delay value '%s' (expected %u..%u seconds)`
- `./drivers/tripplitesu -h` lists:
  `offdelay=<value>  Set shutdown delay, in seconds (default=10, range 1..3600).`

## TODO in the devcontainer (not done yet)

1. Linux build + warnings check for the modified file:
   `./autogen.sh && ./configure && make -j$(nproc) -C common && make -j$(nproc) -C drivers tripplitesu`
2. Runtime validation of parsing, no hardware needed (`-s` avoids ups.conf):
   `mkdir -p /tmp/nut-state /tmp/nut-altpid`
   `NUT_STATEPATH=/tmp/nut-state NUT_ALTPIDPATH=/tmp/nut-altpid ./drivers/tripplitesu -s testups -x port=/dev/null -x offdelay=abc`
     -> expect fatal `Invalid offdelay parameter: abc (expected 1..3600 seconds)`
   same with `-x offdelay=0`    -> expect fatal (range starts at 1)
   same with `-x offdelay=45`   -> expect NO offdelay fatal; fails later on
                                   serial init (acceptable)
3. End-to-end on real SmartOnline hardware (serial):
   driver `-DDDDD`, then `upsd`, then with an upsd.users admin user:
     `upscmd -u admin -p pw <ups> shutdown.return 45`
        -> serial frame `SDA;45` (plus SDR;1 / ARB), log shows `with value 45`
     `upscmd -u admin -p pw <ups> shutdown.return`
        -> `SDA;<offdelay>` (or `SDA;10` with no offdelay in ups.conf)
     `upscmd -u admin -p pw <ups> shutdown.return abc`
        -> `ERR INSTCMD-FAILED`, no serial frame sent
     add `-w` to get the tracked actual result.
4. `make distcheck-light` before proposing upstream (required by AGENTS.md).
5. Docs render NOT validated locally: `asciidoc`/`a2x` are not installed on the
   mac. Changed man sources: `docs/man/tripplitesu.txt`, `docs/man/upscmd.txt`.
   `offdelay` is already in `docs/nut.dict`; the `myups` example sits in an
   indented literal block, following the precedent of `docs/man/upsc.txt`.

## Decisions / open points

- Range 1..3600 s; `0` deliberately rejected (SDA "0" = cancel).
- `shutdown.reboot.graceful` keeps its own 60 s default, not affected by `offdelay`.
- `upsdrv_shutdown()` keeps 5 s unless `offdelay` is explicitly set in ups.conf.
- No new command names (`load.off.delay` / `load.on.delay`) in this change.

## Process constraints (AGENTS.md)

- Human review + DCO; do NOT add a `Signed-off-by` for the contributor.
- Disclose AI assistance as required by the PR template; do not claim hardware
  acceptance that was not actually tested.
- Plain ASCII in sources/docs; keep the change focused.

## Git working tree notes

- Modified (uncommitted): `drivers/tripplitesu.c`, `docs/man/tripplitesu.txt`,
  `docs/man/upscmd.txt`, `NEWS.adoc`.
- Untracked: `Configurazione-Timeout-UPS-Tripp-Lite-in-NUT.md` (input notes),
  this `MEMORY.md`, plus `tests/cgilib.c`, `tests/snmp-ups*.c`,
  `tests/snmp-ups-setvar-test`, `tests/test_cgilib` which appear to be build
  artifacts from the local `make` and are NOT part of this change.

## Docker attempt (aborted on request)

- Docker 29.7.2 (Docker Desktop; socket `/Users/tomzanna/.docker/run/docker.sock`).
- Clean copy of tracked sources made at `/tmp/nut-linux` (20M) for a Linux build;
  `debian:stable-slim` image pulled, but the container run was killed on request
  before building. It will be replaced by the devcontainer approach.


