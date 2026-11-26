# esystemd: Keep the Interface, Ditch the Empire

systemd was a major boon to Linux. It brought order to chaos.

It also brought a holy war.

systemd won the interface war and lost the Unix-philosophy argument. This project keeps the first and offers a solution for the second.

## What systemd got right

Before systemd, each distro packaged its own init scripts for every service, with subtle configuration and boot ordering, all while keeping in sync with each upstream project's changing functionality. systemd gave upstream packages a common way to declare what their services need.

In one declarative file, systemd offers powerful features to a service: cgroup limits, out-of-memory priority, watchdogs, signal filtering, and the many security features that containerization requires. Services, slices, scopes, targets and mounts share one vocabulary: units. Local administrators can cleanly override what a package maintainer shipped.

The modern Linux desktop is built around D-Bus, and systemd hooks deeply into that scaffold. The same program is the system manager at PID 1 and a user manager for each logged-in user, and it keeps a kernel cgroup tree that mirrors the running service hierarchy. When you log in to KDE or GNOME, close your laptop lid, pick a WiFi network or connect your Bluetooth headphones, systemd is orchestrating much of what happens behind the scenes.

Unit files are now how Linux describes services. Every alternative init asks maintainers to write a second set of definitions, which is why the alternatives stay niche.

## What went wrong

systemd is not just an init daemon. It never was, but it was adopted as an alternative init system, and that's why there has been so much pushback. Those who believe each program should do one thing well are offended by the inversion of control: one program now does a great many things, with the highest privileges. Worse, there's only one implementation, with some rather unpersonable leadership at the top. It's a spit in the face of the anarcho-democratic ideals of much of the Linux community.

Even without the philosophy, systemd projects an enormous surface area into modern Linux. We deserve to be able to audit this all-important piece of the puzzle.

That's all been said before. Here's something new.

## A new vision

**Keep the systemd interface, and provide a small alternative implementation.**

Better yet, **contain** the interface, and sort its many features into named profiles.

This is an urban growth boundary, drawn not around systemd itself but around the interface systemd exposes, and which the Linux community actually uses.

POSIX has many implementations. Why should "the thing that starts sshd" have exactly one? systemd's own stability promise says these interfaces are public, and we're taking it at its word.

## The interface is bigger than it looks

It's a lot more than INI files. By Hyrum's law, the real interface is everything clients can observe:

- **Unit files:** drop-ins, `@` templates, specifiers, masking, generators, and implicit dependencies quietly added by `DefaultDependencies=`.
- **Transactions:** Requires, Wants, BindsTo, Conflicts, After and Before, job merging, and exactly which jobs get dropped when there's a dependency cycle.
- **D-Bus:** `org.freedesktop.systemd1`, including the `JobRemoved` signal every blocking `systemctl` call waits on.
- **cgroup path names:** libsystemd finds your login session by parsing `/proc/PID/cgroup` for strings like `session-3.scope`. Yes, really.
- **Wire protocols:** `sd_notify`, socket activation, the `WATCHDOG_USEC` watchdog handshake, and the journal's stdout stream.
- **Filesystem tells:** if `/run/systemd/system/` exists, you were "booted with systemd", and many programs branch on that.

Writing down the unwritten parts is worth doing whether or not anyone ever runs esystemd.

## Two projects

### systemd-conformance

An effort to document, clarify and test the systemd interface.

**Profiles.** It sorts unit file directives, D-Bus methods and signals, and the mechanisms behind them into named profiles:

- **Core** alone is enough to boot and run a stock Debian server.
- **Core + Hardening + Containers** hosts modern, security-conscious container runtimes.
- **Core + Session + Settings** runs a modern desktop with all the trimmings.
- **Internal** covers interfaces that only the systemd project itself uses, though they're publicly documented and exposed.
- **Extended** carries the long tail.

**Corpus.** It measures what's actually used across every package in Debian main: directives in unit files, libsystemd symbols linked by programs, and the D-Bus calls and command lines used at runtime. That lets us say with confidence which profiles a given system or application needs.

**Tests.** Stock systemd is the reference. Every check runs against it first, so the specification describes what systemd actually does.

### esystemd

A small, from-scratch implementation of the systemd interface. It's named in the spirit of elogind and eudev, the stripped-down systemd components that smaller distros use to avoid depending on systemd.

Keep the scary parts tiny, and lean on modern kernel features instead of decades of fallbacks:

- **PID 1 runs to a few hundred lines.**
- **The manager orchestrates and does nothing else:** loading units, running transactions, tracking state, owning the cgroup tree.
- **Text in, kernel mechanism out happens elsewhere.** Instead of compiling `DeviceAllow=` into eBPF inside the most privileged process, the manager passes the text to a small standalone tool that does it.
- **D-Bus stays out of the manager.** A separate endpoint process speaks it.
- **Separate services stay separate.** journald, logind and friends couple only through interfaces, not a private shared library and lockstep versions.

It's not a fork. We share no code with systemd, and upstream's libsystemd stays.

## How we get there

The test is simple: if systemd's own tools can't tell the difference, the interface holds.

1. **Boot stock Debian** under esystemd, with unmodified upstream `systemctl`, logind, journald and udevd.
2. **Test the whole archive.** Load every unit file in Debian under both implementations and diff the results, then diff whole boots.
3. **Desktop.** User managers and logind integration, with GNOME and KDE working.
4. **Shed systemd's binaries** and offer esystemd to Debian as an init alternative.

## What this isn't

- **Not the rest of the empire.** networkd, resolved, homed and systemd-boot can stay as they are.
- **Not a new config format.** That's the whole point.
- **Not relitigating 2014.** We all have the scars.

## Why

I care deeply about open source software, and I can recognize a good thing when I see it. systemd provided an important vision of order and consistency, and a reliable reimplementation of several incoherent tools, daemons and init scripts from a different era.

But it's just so damn big! systemd is a project carried away by its own innovation. I want a small, understandable, auditable core running our most important infrastructure. The Linux community deserves more than one implementation of its core services, and it deserves a clean map of the land they occupy.

PID 1, the web of runtime interfaces between the programs on our computers, and the language we use to describe services are all too important to be left to a single implementation.
