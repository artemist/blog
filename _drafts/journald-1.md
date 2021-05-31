---
layout: post
title: Jumping into journald
date: 2021-05-31
---

On many Linux systems, systemd-journald runs as a daemon at boot and collects your logs. You can access
them through journalctl but it turns out journald is a lot more complicated then just sending something to a text file.

While you'll mostly see entries as a terse error message on one line, journald stores a lot more information.
For example, here's an error from my audio server, [pipewire](https://pipewire.org/). Note that some fields are reordered to make it
easier to explain
```shell
artemis@starlight ~> journalctl --user -xeu pipewire.service -o export
...

_BOOT_ID=eba612c42f634b58b0484c42756ba712
_UID=1000
_GID=1000
_CAP_EFFECTIVE=0
_MACHINE_ID=b3bee68a0f884e6c982529efec408a61
_HOSTNAME=starlight
_TRANSPORT=journal
_SELINUX_CONTEXT=kernel
_AUDIT_SESSION=2
_AUDIT_LOGINUID=1000
_SYSTEMD_OWNER_UID=1000
_SYSTEMD_CGROUP=/user.slice/user-1000.slice/user@1000.service/session.slice/pipewire.service
_SYSTEMD_UNIT=user@1000.service
_SYSTEMD_SLICE=user-1000.slice
_SYSTEMD_USER_SLICE=session.slice
_SYSTEMD_USER_UNIT=pipewire.service
_PID=4046
_COMM=pipewire
_EXE=/nix/store/4qp4npwqabf3mnsy230w3z1nqdjl1gxr-pipewire-0.3.26/bin/pipewire
_CMDLINE=/nix/store/4qp4npwqabf3mnsy230w3z1nqdjl1gxr-pipewire-0.3.26/bin/pipewire
_SYSTEMD_INVOCATION_ID=a9477826e73747ac810f958f894c302e
_SOURCE_REALTIME_TIMESTAMP=1622336991484040
__CURSOR=s=ac8b24ddf2634038b49168edc5d6e544;i=b4233a3;b=4950abcba7ec46629fce878fb239a6e6;m=1408502e8be;t=5c381c41568ad;x=2af33cc0b675a511
__REALTIME_TIMESTAMP=1622336991488173
__MONOTONIC_TIMESTAMP=1376621095102
PRIORITY=4
SYSLOG_IDENTIFIER=pipewire
CODE_FILE=../src/pipewire/impl-node.c
CODE_LINE=957
CODE_FUNC=dump_states
MESSAGE=(PipeWire ALSA [.electron-wrapped]-110) client too slow! rate:256/48000 pos:4451017216 status:triggered
```

An entry consists of freeform variables with binary (though generally ASCII/US english) values. Values starting with an underscore
are "trusted" and generated by journald while others are sent by the process along with the primary message. This helps provide
context about what exact process failed and what state it was in during that failure. Unfortunately, the [official descriptions](https://www.freedesktop.org/software/systemd/man/systemd.journal-fields.html) of what these fields mean can be a bit obtuse.

While working on my prototype for a system-journald replacement, [rjournald](https://github.com/artemist/rjournald) I've discovered what many of these mean through context or reading the systemd code:
- **_BOOT_ID** is a UUID that changes every time you start your system. This is generated by the Linux kernel and is accessable at `/proc/sys/kernel/random/boot_id`. I've found it useful to help figure out if I've rebooted my system since an error occured
- **_UID** tells you what user executed the process. You can get this for your user with the command `id`.
- **_GID** tells you which group the process was using. While a user can have several groups, a process executes under one primary group ID
- **_CAP_EFFECTIVE** provides what [capabilities](https://linux.die.net/man/7/capabilities) a process can use. Capabilities give fine-grained privelaged access to processes without requiring them to be the root user. For example, binding to port 80 or 443 requires the CAP_NET_BIND_SERVICE capability. If _CAP_EFFECTIVE=0 then you know you've missed that capability.
- **_MACHINE_ID** is a unique ID to the system which you can find in `/etc/machine-id`. This should be set on the first boot of your system by systemd and helps you figure out if the logs could be from before a system was reinstalled
- **_HOSTNAME** is the name of the system. You probably set this in install. I'm not quire sure where systemd gets this, as there's multiple different places to set the hostname, including systemd-hostnamed (which lets you set non-ascii hostnames for some programs), `/etc/hostname` (which is where systemd will read your hostname and tell it to the kernel), and `/proc/sys/kernel/hostname` (what the kernel thinks your hostname is).
- **_TRANSPORT** How the log got to journald. As of writing I could find 6 transports: journal (using the native journald protocol), stdout (a process's standard output or error redirected to systemd), syslog (the legacy linux logging system), kernel (kernel messages you can get through the `dmesg` command), audit (logs the kernel generates about programs' activities), and driver (error messages from within journald). I'll discuss a few in more depth later