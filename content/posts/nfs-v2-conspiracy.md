+++
title = "The Great NFS v2 Conspiracy:"
date = "2026-03-06T15:14:43+01:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "Pawel S."
authorTwitter = "" #do not include @
cover = ""
tags = ["nfs", "sgi"]
keywords = ["nfsv2", "nfs sgi"]
description = ""
showFullContent = false
readingTime = false
hideComments = false
+++
How the modern kernel cabal buried a protocol and one brave soul fought back.

In the shadowy underbelly of the tech world, a silent coup is underway—led by the faceless overlords of the "Modern Kernel Cabal", a brotherhood of kernel maintainers and corporate suits (of course!) who whisper in dark corners of data centers: **"Don't use NFS v2..."**. That old-school file-sharing protocol from the '80s? Too free, too simple, too dangerous for their new world order of shiny v4 mandates and locked-down security theater. They buried it deep in code tombs, forcing truth-seekers to claw through OS hell to revive it! 
This is the tale of one brave soul's epic battle against the machine — a conspiracy driven odyssey to resurrect NFS v2 and mount an IRIX 6.5.22 share from a modern Ubuntu 24.04 server! Buckle up — it's a wild ride of failed compiles, VM voodoo, Docker disasters, and one very stubborn **SGI** workstation.

## The innocent beginning

It started so simply. A Indigo2 SGI IRIX 6.5.22 workstation, a shelf full of freeware tardists, and a perfectly reasonable desire: share `/srv/sgi` over NFS so the old beast could install `Pine`. Just an simple yet comfortable email client from the late 90's.
The server? A beefy Ubuntu 24.04 machine running kernel 6.17 HWE edge, bristling with modern capability. The setup should have taken ten minutes...
First attempt: the obvious one - `nfs-kernel-server` — the standard Linux kernel NFS server, battle-tested, universally supported. Config written, export added to /etc/exports:
```
/srv/sgi *(rw,sync,no_root_squash,no_subtree_check)
```
Service started. `showmount -e localhost` confirmed the export. From the IRIX side, mount attempts produced nothing — no traffic on port 2049, no response, just the workstation hanging with "server not responding." The syslog on IRIX told the first truth:
``` bash
unix: NFS2 unknown failed for server 192.168.200.10: Program/version mismatch
```
IRIX 6.5, bless its ancient heart, speaks NFSv2 as its mother tongue. It will try v2 first, always, regardless of what flags you pass it. The kernel NFS server's rpcinfo output was damning:
``` bash
100003    3   tcp   2049  nfs
100003    4   tcp   2049  nfs
100003    3   udp   2049  nfs
```
No version 2! Attempts to enable it via /etc/nfs.conf with `vers2=y` and `RPCNFSDARGS="--nfs-version 2,3,4"` in /etc/default/nfs-kernel-server failed silently. The config was accepted, the daemon restarted, and absolutely nothing changed. The kernel config revealed why:
```
# CONFIG_NFSD_V2 is not set
```
Ubuntu 24.04's kernel — including the bleeding-edge 6.17 HWE — had `CONFIG_NFSD_V2` compiled out entirely. **Not a module. Not disabled by a runtime flag**. Gone. The cabal had struck at the source!

## The Ganesha gambit

Surely a userspace NFS implementation would have more flexibility? Enjoy `nfs-ganesha` — the sophisticated userland NFS server that advertises NFSv2 support right there in its feature list. If the kernel won't serve v2, bypass the kernel entirely. The cabal hadn't anticipated this flanking maneuver. Or had they?

``` bash
apt install nfs-ganesha nfs-ganesha-vfs
```

Ganesha 4.3 installed. Config written with `NFS_Protocols = 2,3` and an EXPORT block pointing at `/srv/sgi`. Service started. But the logs immediately revealed the cabal's reach extended even here.
Ganesha now started cleanly, the export loaded, `showmount -e localhost` returned `/srv/sgi (everyone)`. Progress.

But the critical blow came when checking what Ganesha actually registered:
``` bash
100003    3   udp   2049  nfs
100003    3   tcp   2049  nfs
```

Version 3 only. Ganesha 4.x had dropped NFSv2 support entirely — the feature list was a lie, or rather, ancient history. The Ubuntu package was compiled without it. Setting `NFS_Protocols = 2,3` in the config caused the EXPORT block to fail validation with `1 (unknown param) errors` because `Protocols = 2,3` was simply not a valid value in this build. Ganesha started, but served nothing IRIX could use.

The userspace escape hatch was welded shut. The cabal had gotten to the package maintainers too...

## The Portmapper labyrinth

With both kernel and userspace NFS servers neutralized on the v2 front, attention turned to at least making NFSv3 work — maybe IRIX could be persuaded to speak it.

The breakthrough came from IRIX's own syslog. Every single mount attempt, regardless of `-o nfsv3` or `-o vers=3` flags, showed IRIX sending NFSv2 `getattr` requests and receiving `PROG_MISMATCH`. The version flags were being cheerfully ignored. IRIX has a v2 file handle call and was hammering the server with it like a golden ticket that absolutely should work. A `tcpdump` session during a mount attempt finally showed the full picture:
``` bash
192.168.201.33.682 > 192.168.200.10.2049: NFS request xid 940617122 116 
    getattr fh Unknown/010004000F000200FD167A40...
192.168.200.10.2049 > 192.168.201.33.682: NFS reply xid 940617122 
    reply ok 32 getattr PROG_MISMATCH
```

The file handle format `010004000F000200...` — a v2 file handle. Repeated five times per mount attempt. IRIX was not going to give up on v2 quietly.

## The Docker detour

Surely a container running an older userspace NFS implementation could paper over the kernel's deficiency? A Docker image was summoned — `erichough/nfs-server` — promising NFS in a box. Port conflicts with the host's rpcbind required stopping system services first. Then it only served NFSv4. Then it needed kernel modules loaded on the host. Bummer. And of course those kernel modules — specifically `nfsd` with v2 support — didn't exist in the host kernel.
```
----> kernel module nfs is missing
----> ERROR: nfs module is not loaded in the Docker host's kernel
```

The container shares the host kernel. There is no escape from `# CONFIG_NFSD_V2 is not set`. The cabal had sealed this exit before the container revolution even began.

## The VM gauntlet

Perhaps a virtual machine running an older distro? The conspiracy's reach would surely not extend back through time itself.

**Ubuntu 20.04 (Focal)** was stood up with `virt-install`, bridged to `br0` so IRIX could reach it directly. The kernel config verdict:
``` bash
CONFIG_NFSD_V2_ACL=y
# CONFIG_NFSD_V2 is not set
```

`NFSD_V2_ACL` — *ACL support* for NFSv2 — was present. But NFSv2 itself was not. A cruel joke from the cabal. Focal had dropped it too. Exactly when this happened remains murky — the cabal does not publish press releases.

**Alpine Linux** was next — lightweight, fast to deploy. `apk add nfs-utils`, exports configured, `rc-service nfs start`. The rpcinfo output showed mountd versions 1, 2, and 3 registered, raising hopes. But `/proc/fs/nfsd/versions` told the real story:
```
+3 +4 +4.1 +4.2
```

No v2. Alpine's LTS kernel had quietly dropped it too. The kernel config couldn't even be verified — Alpine ships without `/boot/config-*` and `/proc/config.gz` was absent. The cabal had hidden even the evidence of the crime.

## Small RPi strikes back

**The Raspberry Pi** running Raspbian Bullseye appeared as a wildcard — small ARM device running a few services at home (who needs gps corrected ntp at home?) a custom kernel, perhaps overlooked by the cabal's reach. 
Raspbian's custom kernel also had `CONFIG_NFSD_V2` compiled out. The Pi's export also required `fsid=` because `/srv/sgi` wasn't a native mountpoint. All pain, no v2.

## The source code saga

If the kernel won't include it, compile it yourself. Revolutionary thinking. The Ubuntu 6.17 kernel source was obtained — but the running kernel was `6.17.0-14` while the available source package in the repos was only for `6.17.0-16`. This version mismatch caused cascading failures:
``` bash
ERROR: modpost: "svc_find_listener" [nfsd.ko] undefined!
ERROR: modpost: "getboottime64" [nfsd.ko] undefined!
WARNING: modpost: suppressed 523 unresolved symbol warnings
```

The exact matching source was manually downloaded from Launchpad. With `Module.symvers` copied from the running kernel headers and `scripts/config --enable NFSD_V2` applied, compilation proceeded — until:
``` bash
make[3]: *** No rule to make target 'nfsd.ko', needed by '__modfinal'. Stop.
```
The fundamental problem: `CONFIG_NFSD_V2=y` marks it as built-in, but `CONFIG_NFSD=m` makes the whole nfsd a module. Setting `NFSD_V2=m` was rejected by Kconfig — it cannot be a standalone module, only a built-in component of nfsd. These two constraints are irreconcilable without building an entire kernel. Disabling `MODULE_SIG`, `DEBUG_INFO_BTF`, and various other options made no difference. The .o files compiled fine but nfsd.ko was never linked. The cabal had made NFSv2 re-enabling structurally impossible without a full kernel build.

## FreeBSD to the rescue

In desperation, a FreeBSD 14.3 VM was deployed. FreeBSD — that proud BSD rebel, untouched by the Linux kernel cabal's machinations, marching to its own drummer since 1993.
After installation and static IP assignment:
```
cat >> /etc/rc.conf << 'EOF'
rpcbind_enable="YES"
nfs_server_enable="YES"
nfsv2_server_enable="YES"
mountd_enable="YES"
mountd_flags="-r"
EOF
```

And the glorious output of `rpcinfo -p localhost | grep nfs`:
``` bash
100003    2   udp   2049  nfs
100003    3   udp   2049  nfs
100003    2   tcp   2049  nfs
100003    3   tcp   2049  nfs
```

**Version 2. Right there. In broad daylight.** FreeBSD hadn't drunk the Kool-Aid.

But the cabal had one more trick. The actual SGI software lived on the Ubuntu engine at `/srv/sgi` — 278GB of IRIX freeware, overlays, and tardists. FreeBSD needed to re-export it. An NFS mount of engine's `/srv/sgi` followed by a ZFS dataset and nullfs bind mount to `/zroot/sgi` seemed elegant — until FreeBSD's mountd detected the re-export:
``` bash
/srv/sgi on /zroot/sgi (nullfs, NFS exported)
can't get fh for /zroot/sgi
```

FreeBSD, admirably paranoid, refused to re-export an NFS mount over NFS. A new 300GB virtual disk was attached to the VM via `virsh attach-disk`, a new ZFS pool created with `zpool create data /dev/vtbd1`, and the data rsynced locally from the already-mounted engine share. With the data living on genuine local ZFS:
```
Exports list on localhost:
/data/sgi                         192.168.201.33
```

## The triumphant mount

After rebooting IRIX to flush cached v2 file handles, the moment of truth:
``` bash
IRIS# mount 192.168.200.13:/data/sgi /mnt/sgi
```

No error. No "giving up." Silence — the good kind. The treasure was there:
``` bash
-rw-r--r--  1 root  sys  4487224  Apr 7 2003  fw_pine.sw
-rw-r--r--  1 root  sys  3763858  Apr 7 2003  fw_pine.src
-rw-r--r--  1 root  sys   212272  Apr 7 2003  fw_pine.man
```

Installing Pine required opening multiple freeware distributions simultaneously in `inst` to resolve dependencies — `fw_openssl` from freeware2, `fw_common` from freeware3:
```
Inst> open /mnt/sgi/freeware2
Inst> open /mnt/sgi/freeware3
Inst> conflicts 1b 2b
Inst> go
```

Pine installed. The SGI workstation could check email again. (as Pine is no longer Elm)

## The verdict
Here is what the "great NFS v2 conspiracy" taught me:
`CONFIG_NFSD_V2` has been removed from every major Linux distribution's kernel — Ubuntu 20.04, Ubuntu 24.04, Debian Bullseye, Debian Bookworm, Raspbian, Alpine. The option exists in `Kconfig` but cannot be enabled as a module, and enabling it as built-in while `NFSD=m` creates an irreconcilable build conflict without a full kernel recompile. The userspace escape via Ganesha is blocked because the Ubuntu Ganesha package is compiled without v2 support. Docker containers inherit the host kernel. VMs running modern distros have the same kernel. The walls are complete.
NFSv2 client support (`CONFIG_NFS_V2=m`) still exists in Linux — you can mount NFSv2 shares, the kernel just won't serve them. Make of that asymmetry what you will.

FreeBSD 14.x remains the last bastion of NFSv2 server support on modern hardware, requiring nothing more than nfsv2_server_enable="YES" in rc.conf. Treasure it.
And IRIX 6.5? It will speak NFSv2 until the sun burns out, `-o nfsv3` be damned. Some protocols cannot be modernized. Some shouldn't be.
