# gokrazy rsync

[![tests](https://github.com/gokrazy/rsync/actions/workflows/main.yml/badge.svg)](https://github.com/gokrazy/rsync/actions/workflows/main.yml)

Package rsync contains a native Go rsync implementation.

⚠ **Beware:** very fresh. Might eat your data. You have been warned! ⚠

The only component currently is gokr-rsyncd, a read-only rsync daemon
sender-only Go implementation of rsyncd. rsync daemon is a custom
(un-standardized) network protocol, running on port 873 by default.

This project accepts contributions as time permits to merge them (best effort).

## Existing rsync implementation survey

| Language | URL                                                                             | Note                                                                                                                                  | Max Protocol                                                                                                        | Server mode? |
|----------|---------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|--------------|
| C        | [WayneD/rsync](https://github.com/WayneD/rsync)                                 | original “tridge” implementation; I found [older versions](https://github.com/WayneD/rsync/tree/v2.6.1pre2) easier to study           | [31](https://github.com/WayneD/rsync/blob/592c6bc3e5e93f36c2fdc0a491a9fb43a41cf688/rsync.h#L113)                    | ✔ yes        |
| C        | [kristapsdz/openrsync](https://github.com/kristapsdz/openrsync)                 | OpenBSD, good docs                                                                                                                    | [27](https://github.com/kristapsdz/openrsync/blob/e54d57f7572381da2b549d39c7968fc79dac8e1d/extern.h#L30)            | ✔ yes        |
| **Go**   | [gokrazy/rsync](https://github.com/gokrazy/rsync)                               | → you are here ←                                                                                                                      | [27](https://github.com/gokrazy/rsync/blob/b3b58770b864613551036a2ef2827b74ace77749/internal/rsyncd/rsyncd.go#L317) | ✔ yes 🎉     |
| **Go**   | [jbreiding/rsync-go](https://github.com/jbreiding/rsync-go)                     | rsync algorithm                                                                                                                       |                                                                                                                     | ❌ no        |
| **Go**   | [kaiakz/rsync-os](https://github.com/kaiakz/rsync-os)                           | only client/receiver                                                                                                                  | [27](https://github.com/kaiakz/rsync-os/blob/64e84daeabb1fa4d2c7cf766c196306adfba6cb2/rsync/const.go#L4)            | ❌ no        |
| **Go**   | [knight42](https://gist.github.com/knight42/6ad35ce6fbf96519259b43a8c3f37478)   | proxy                                                                                                                                 |                                                                                                                     | ❌ no        |
| **Go**   | [c4milo/gsync](https://github.com/c4milo/gsync)                                 |                                                                                                                                       |                                                                                                                     | ❌ no        |
| Java     | [APNIC-net/repositoryd](https://github.com/APNIC-net/repositoryd)               | archived                                                                                                                              |                                                                                                                     | ✔ yes        |
| Java     | [JohannesBuchner/Jarsync](https://github.com/JohannesBuchner/Jarsync/)          | archived, [internet draft RFC “The rsync Network Protocol”](https://github.com/JohannesBuchner/Jarsync/blob/master/jarsync/rsync.txt) |                                                                                                                     | ✔ yes        |
| Java     | [perlundq/yajsync](https://github.com/perlundq/yajsync#example)                 |                                                                                                                                       |                                                                                                                     | ✔ yes        |
| C++      | [gilbertchen/acrosync-library](https://github.com/gilbertchen/acrosync-library) | commercial                                                                                                                            |                                                                                                                     | ❌ no        |
| Rust     | [sourcefrog/rsyn](https://github.com/sourcefrog/rsyn#why-do-this)               | client, “rsyn is rsync with no c”                                                                                                     | [27](https://github.com/sourcefrog/rsyn/blob/2ebbfcfe999fdf2d1a434d8614d07aa93873461b/src/connection.rs#L38)        | ❌ no        |

## Getting started

To serve the current directory via rsync on `localhost:8730`, use:

```
go install github.com/gokrazy/rsync/cmd/gokr-rsyncd
gokr-rsyncd -modulemap=pwd=$PWD
```

You can then copy the contents of the current directory with clients such as
`rsync(1)`:

```
% rsync -v --archive --port 8730 rsync://localhost/pwd/ quine
receiving file list ... done
created directory quine
./
.git/
[…]
.github/workflows/main.yml
LICENSE
Makefile
README.md
cmd/gokr-rsyncd/rsyncd.go
doc.go
go.mod
go.sum
internal/rsyncd/connection.go
internal/rsyncd/rsyncd.go
interop_test.go

sent 1,234 bytes  received 5,678 bytes  13,824.00 bytes/sec
total size is 666  speedup is 0.10

```

…or [`openrsync(1)`](https://github.com/kristapsdz/openrsync), shown doing a
differential update:

```
% openrsync -v --archive --port 8730 rsync://localhost/pwd/ quine
socket.c:109: warning: connect refused: ::1, localhost
Transfer starting: 369 files
.git/index (1.1 KB, 100.0% downloaded)
Transfer complete: 5.5 KB sent, 1.2 KB read, 666 B file size

```

## Limitations

### Bandwidth

In my tests, `gokr-rsyncd` can easily transfer data at > 6 Gbit/s. The current
bottleneck is the MD4 algorithm itself (not sure whether in the “tridge” rsync
client, or in `gokr-rsyncd`). Implementing support for more recent protocol
versions would help here, as these include hash algorithm negotiation with more
recent choices.

### Protocol related limitations

* xattrs (including acls) was introduced in rsync protocol 30, so is currently
  not supported.

## Supported environments and privilege dropping

Supported environments:

1. systemd (Linux)
1. Docker (Linux)
1. privileged Linux
1. privileged non-Linux

In all environments, the default instructions will take care that:

* (On Linux only) The host file system is made **read-only** for `gokr-rsyncd`,
  to guard against accidental data exfiltration.
* `gokr-rsyncd` is running without privileges, as user `nobody`, to limit the
  scope of what an attacker can do when exploiting a vulnerability.

Known gaps:

* `gokr-rsyncd` does not guard against denial of service attacks, i.e. consuming
  too many resources (connections, bandwidth, CPU, …).
  * See also [Per-IP rate limiting with
    iptables](https://making.pusher.com/per-ip-rate-limiting-with-iptables/).


### systemd (unprivileged)

We provide [a `gokr-rsyncd.socket` and `gokr-rsyncd.service`
file](https://github.com/gokrazy/rsync/tree/main/systemd/) for systemd. These
files enables most of systemd’s security features. You can check by running
`systemd-analyze security gokr-rsyncd.service`, which should result in an
exposure level of “0.2 SAFE” as of systemd 249 (September 2021).

First, configure your server flags by creating a systemd service override file:

```shell
systemctl edit gokr-rsyncd.service
```

In the opened editor, change the file to:
```
[Service]
ExecStart=
ExecStart=/usr/bin/gokr-rsyncd -modulemap=pwd=/etc/tmpfiles.d
```

Close the editor and install the service using:

```shell
systemctl enable --now gokr-rsyncd.socket
```

Additional hardening recommendations:

* Restrict which IP addresses are allowed to connect to your rsync server, for example:
  * using iptables or nftables on your host system
  * using [`gokr-rsyncd`’s built-in IP allow/deny mechanism](https://github.com/gokrazy/rsync/issues/4) (once implemented)
  * using [systemd’s `IPAddressDeny` and `IPAddressAllow`](https://manpages.debian.org/systemd.resource-control.5) in `gokr-rsyncd.socket`
* To reduce the impact of Denial Of Service attacks, you can restrict resources
  with systemd, see [Managing
  Resources](http://0pointer.de/blog/projects/resources.html).
* To hide system directories not relevant to any rsync module, use [systemd’s
  `TemporaryFileSystem=` and
  `BindReadOnlyPaths=`](https://manpages.debian.org/systemd.exec.5) directives
  as described in [Use TemporaryFileSystem to hide files or directories from
  systemd
  services](https://www.sherbers.de/use-temporaryfilesystem-to-hide-files-or-directories-from-systemd-services/). Note
  that you [may need to disable `ProtectSystem=strict` due to a
  bug](https://github.com/systemd/systemd/issues/18999).

### Docker (unprivileged)

We provide [a `Dockerfile` for
`gokr-rsyncd`](https://github.com/gokrazy/rsync/tree/main/docker/).

```shell
docker run \
  --read-only \
  -p 127.0.0.1:8730:8730 \
  -v /etc/tmpfiles.d:/srv/rsync:ro \
  stapelberg/gokrazy-rsync:latest \
    -modulemap=pwd=/srv/rsync
```

Additional hardening recommendations:

* Restrict which IP addresses are allowed to connect to your rsync server, for example:
  * using iptables or nftables on your host system
  * using [`gokr-rsyncd`’s built-in IP allow/deny mechanism](https://github.com/gokrazy/rsync/issues/4) (once implemented)
    * Be sure to set up Docker such that the remote IPv4 or IPv6 address is available inside the container, see https://michael.stapelberg.ch/posts/2018-12-12-docker-ipv6/

### privileged Linux (including gokrazy.org)

When started as `root` on Linux, `gokr-rsyncd` will create a mount namespace,
mount all configured rsync modules read-only into the namespace, then change
into the namespace using [`chroot(2)`](https://manpages.debian.org/chroot.2) and
drop privileges using [`setuid(2)`](https://manpages.debian.org/setuid.2).

**Tip:** you can verify which file system objects the daemon process can see by
using `ls -l /proc/$(pidof gokr-rsyncd)/root/`.

Additional hardening recommendations:

* Restrict which IP addresses are allowed to connect to your rsync server, for example:
  * using iptables or nftables on your host system
  * using [`gokr-rsyncd`’s built-in IP allow/deny mechanism](https://github.com/gokrazy/rsync/issues/4) (once implemented)

### privileged non-Linux (e.g. Mac)

When started as `root` on non-Linux (e.g. Mac), `gokr-rsyncd` will drop
privileges using [`setuid(2)`](https://manpages.debian.org/setuid.2).

### unprivileged with write permission (e.g. from a shell)

To prevent accidental misconfiguration, `gokr-rsyncd` refuses to start when it
detects that it has write permission in any configured rsync module.

