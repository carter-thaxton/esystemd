# esystemd: Keep the Interface, Skip the Empire

## The pitch

systemd won the interface war and lost the Unix-philosophy argument. esystemd keeps the first and fixes the second. It's a small, auditable, from-scratch implementation of the systemd *interface* that boots stock Debian, packages and all, with nothing for package maintainers to do.

Unit files are now how Linux describes services. Every other alternative init asks maintainers to write a second set of definitions, which is why they stay niche. esystemd takes the opposite bet: treat the interface as a standard and give it a second implementation. POSIX has many implementations, so why should "the thing that starts sshd" have exactly one? systemd's own stability promise says these interfaces are public, and we're taking it at its word. Prior art has been forks or narrow shims, never a full independent implementation.

## What "the systemd interface" actually is

It's a lot more than INI files. Hyrum's law applies: the real interface is everything clients can observe.

- **Unit files:** drop-ins, `@` templates, specifiers, masking, generators, and the implicit dependencies that `DefaultDependencies=` quietly adds.
- **Transaction semantics:** Requires, Wants, BindsTo, Conflicts, After and Before, job merging, and exactly which jobs are dropped when systemd detects a dependency cycle.
- **D-Bus:** `org.freedesktop.systemd1` on `/run/systemd/private` and the system bus, including `JobRemoved` and `PropertiesChanged`, which every blocking `systemctl` call waits on.
- **cgroup path names:** libsystemd finds your login session by parsing `/proc/PID/cgroup` for strings like `session-3.scope`. Yes, really.
- **Wire protocols:** `sd_notify`, `LISTEN_FDS` socket activation, the fd store, and the journal stdout stream.
- **Filesystem tells:** if `/run/systemd/system/` exists, you were "booted with systemd". Many programs branch on that.

Writing down the unwritten parts is a deliverable in its own right, whether or not anyone ever runs esystemd.

## Design: small enough to actually audit

Keep the scary parts tiny, and lean on modern kernel primitives instead of decades of fallbacks.

&#91;embedded content: esystemd architecture · PID 1, manager, exec helper, D-Bus endpoint\]

PID 1 runs to a few hundred lines. The sandboxing lives in a stateless exec helper, and D-Bus stays out of the manager.

## How we get there

The test is simple: if systemd's own tools can't tell the difference, the interface holds.

1. **Boot under stock clients.** Debian runs with `init=/usr/lib/esystemd/init` and unmodified upstream `systemctl`, logind and udevd. We port upstream's integration tests. Those tests never mention \`JobRemoved\`, so we also record D-Bus traffic from real clients and diff it.
2. **Test the whole archive.** Load every unit file in Debian stable under both implementations, diff the results, and diff whole boots.
3. **Desktop.** Add user managers and logind integration, with GNOME and KDE working. Then harden and audit.
4. **Shed systemd's binaries and propose to Debian.** Upload as an `init` alternative, then propose narrow virtual packages. An embedded downstream like Raspberry Pi would be a first candidate target.

## Non-goals

- **Not a fork.** We share no code with systemd, and upstream's libsystemd0 stays.
- **Not the rest of the empire.** networkd, resolved, homed and systemd-boot can stay as they are.
- **Not a new config format.** That's the whole point.
- **Not relitigating 2014.** We all have the scars.
