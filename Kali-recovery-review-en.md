# Kali VM Expansion and Recovery Review

> Reconstructed from this conversation, screenshots, and terminal output. The user ultimately reported success. This is a record of one VM, not a partition script to copy onto another disk.

## 1. Diagnostic approach: identify the failing layer

The incident involved several connected problems:

```text
Root filesystem ran out of space during package upgrades
  → MariaDB upgrade failed; 480 packages remained unfinished
  → Black screen after reboot; lightdm was reported failed
  → Virtual disk was 80 GiB, but the root partition was only 18.9 GiB
  → Partition changes were attempted from GParted Live
  → Shrinking the extended partition repeatedly failed boundary validation
  → Using None alignment for that container allowed the operation to be queued
  → Root partition was expanded and Kali booted again
  → Package repair then failed because networking was unavailable
  → Activating eth0 restored the default route and DNS resolution
  → Package repair completed
```

Two limits on the evidence matter:

- A full root filesystem, unfinished packages, and failed lightdm were directly observed. The space shortage and interrupted upgrade plausibly explain the black screen, but no lightdm logs were collected to establish a unique cause.
- Correct resize values still triggered GParted's error, while changing the extended container to `None` alignment allowed the operation. This implicates alignment or boundary calculation for that operation; it does not establish a specific confirmed software defect without internal diagnostics.

The original missing-VM investigation is separate. The Windows `Kali_Purple` directory was empty, but the full-disk search was not completed and VMware configuration access failed. That does not prove the original VM was deleted, or establish whether the later running Kali was the same VM.

### Learning priorities

The emphasis follows the topics you asked about repeatedly, rather than assigning a score to your understanding:

1. **Virtual disk, partition, and filesystem capacity are different; expansion requires adjacent free space.**
2. **An ISO is boot/install media. Installed Kali lives on the virtual hard disk; attaching an ISO does not mean booting it.**
3. **A GParted preview, pending operation, and applied disk change are different states.**
4. **A connected virtual adapter does not guarantee internet access: address, route, and DNS must work.**
5. **Fix the underlying storage or network problem before finishing an interrupted package installation.**

## 2. Complete procedure in chronological order

### 1. Locating the VM and entering installation

The directory `C:\Users\agg\Documents\Virtual Machines\Kali_Purple` existed but was empty. A Kali installer ISO was also found. An installer ISO is not the old virtual disk and does not contain the old VM's projects or settings. The investigation did not establish where the old VM went.

You subsequently entered the Kali installation workflow. Guidance covered NAT networking, a regular user, Xfce, and the default tool selection. Guided whole-disk partitioning was appropriate only for a confirmed empty virtual disk. The conversation does not prove that every suggested installer option was used.

### 2. MariaDB upgrade failed because space was unavailable

The command was:

```bash
sudo apt --fix-broken install
```

Important output included:

```text
480 not fully installed or removed.
ERROR: There's not enough space in /var/lib/mariadb/
new mariadb-server package preinst maintainer script subprocess failed
```

The failure occurred in a pre-installation script before unpacking MariaDB. It did not itself prove database corruption. This package version checks available space on the filesystem containing its data directory and exits below roughly 4 MiB. That is a minimum check, not an estimate of the space needed to finish the upgrade.

### 3. Investigating the black screen through a text console

After the black screen appeared, a text console still allowed login. The checks were:

```bash
df -h
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
sudo systemctl --failed --no-pager
```

| Item | Observed state | Meaning |
|---|---|---|
| `/dev/sda` | 80 GiB | The virtual disk had already been enlarged |
| `/dev/sda1` | 18.9 GiB, mounted at `/` | The system partition had not grown with it |
| Available space on `/` | 0; 100% used | The system filesystem was full |
| `/dev/sda5` | Approximately 1.1 GiB, swap | Swap followed the root partition |
| `lightdm.service` | failed | Graphical login failed, but text login still worked |

### 4. Clearing package cache and inspecting the partition table

Running `sudo apt clean` freed approximately 2.7 GB on the root filesystem. It removes downloaded package archives, not installed software. Subsequent repair therefore needed to download packages again.

At one point, `df -h /` and `fdisk -l /dev/sda` were entered on the same line. `df` interpreted `fdisk` as a path and reported `No such file or directory`. They should be separate commands:

```bash
df -h /
sudo fdisk -l /dev/sda
```

The `-l` is a lowercase L, meaning list.

The original disk used an MBR/msdos partition table:

```text
[ sda1: system, ~19 GiB ][ sda2: contains sda5 swap, ~1 GiB ][ unallocated: 60 GiB ]
```

An 80 GiB disk does not automatically give the root filesystem 80 GiB. Both its partition boundary and filesystem must accommodate the additional space.

### 5. Booting GParted Live from the virtual CD/DVD drive

Initially, selecting the GParted ISO still booted Kali. The file manager showed files on the virtual CD. This established that the ISO was attached, not that the VM had booted from it.

The procedure was:

1. Shut down the VM; prepare a VM backup before applying partition changes.
2. Select the GParted Live amd64 ISO in VMware's CD/DVD settings and enable `Connect at power on`.
3. Use `Power On to Firmware`.
4. In PhoenixBIOS, move `CD-ROM Drive` above the hard disk under Boot, then save with F10.
5. Select `Don't touch keymap`, retain the default language, and choose mode `0` to start the graphical environment.

A Live environment runs from the ISO, allowing the installed root filesystem to remain unused during partition work. Reinstalling Kali was unnecessary.

### 6. Moving swap before expanding the root partition

The unallocated space was not adjacent to `sda1`. `sda2` is an extended-partition container holding logical partition `sda5`; it is not another physical disk.

The intended sequence was:

```text
1. Extend the sda2 container into the unallocated space.
2. Move sda5 to the far right while keeping swap around 1.08 GiB.
3. Shrink sda2 from its left edge, releasing the space outside the container.
4. Extend the right edge of sda1 into that adjacent space.
```

Growing `sda2` to 61.08 GiB did **not** allocate 61 GB of swap. The inner `sda5` determines swap capacity.

Mistakes or ambiguous intermediate states were corrected along the way:

- In one preview, `sda5` filled almost the whole container, preventing it from shrinking; that pending operation was undone.
- Another dialog showed swap reduced to 1000 MiB; it was subsequently restored to approximately 1102 MiB and positioned at the end.
- Dragging the middle moves a partition; dragging an edge resizes it. After editing preceding free space, the resulting New size must also be checked.
- A later extra pending operation moved `sda5` left. That was not the intended next step and needed to be undone.

### 7. Repeated GParted boundary error and the successful workaround

Shrinking `sda2` repeatedly produced:

```text
Could not add this operation to the list
GParted Bug: A partition cannot end (165511168) after the end of the device (%2)
```

This means validation rejected the proposed operation before adding it to the queue. It is not proof that the disk was damaged. The unreplaced `%2` also means this dialog should not be treated as a reliable report of disk capacity.

Numeric edits, edge dragging, applying earlier operations separately, and refreshing did not eliminate the error. Earlier guidance repeatedly attributing the issue to dragging was insufficient once correct values reproduced it. The next useful step was to inspect actual on-disk boundaries.

Read-only checks in the Live terminal were:

```bash
sudo fdisk -l /dev/sda
sudo parted /dev/sda unit s print free
```

The actual recorded layout was:

| Object | Start sector | End sector |
|---|---:|---:|
| Whole disk | 0 | 167772159 |
| sda1 | 2048 | 39684095 |
| sda2 | 39686142 | 167772159 |
| sda5 | 165515262 | 167772159 |

Sectors were 512 bytes. These readings proved that swap had actually moved to the disk's end, rather than merely appearing there in a pending preview. The extended container also ended at the disk's last sector.

The successful sequence was:

1. Select `Align to: None` for the **sda2 container only**.
2. Keep its right edge fixed and shrink from the left, retaining about 1104 MiB.
3. The operation was accepted. The preview retained approximately 2 MiB inside the container before swap.
4. Keep `MiB` alignment for **sda1** and expand only its right edge, targeting approximately 78.92 GiB.
5. Review and apply the queued operations. The user reported completion.

**The failure arose during boundary validation when shrinking the previously expanded container. Temporarily expanding the container was not shown to damage it.** Using `None` was a targeted workaround after inspecting the layout, not a recommended default for every partition. The official manual warns that this mode does not guarantee space for boot records; retaining the internal gap therefore mattered.

### 8. Returning to Kali: no installer ISO was required

After finishing, the Live environment was shut down and the virtual optical drive disconnected so the VM could boot from its hard disk.

You repeatedly checked whether the ISO needed to be changed back to Kali. **It did not.** The saved ISO path could still say GParted; a disconnected optical drive does not participate in booting.

| Setting | Meaning |
|---|---|
| Connected | Whether the virtual optical drive is currently connected |
| Connect at power on | Whether it reconnects automatically at the next power-on |
| ISO path | Which image the connected drive uses |
| BIOS boot order | Which device firmware tries first |

Selecting a Kali ISO does not itself erase the system. Booting its installer and confirming formatting or installation can overwrite it. Installed Kali does not need the installer ISO for everyday operation.

Keep backups until capacity, reboot, packages, and project files are checked. Delete only the separate backup copy when appropriate, not the active `.vmdk` virtual disks or `.vmx` configuration. VMware snapshots should be managed through Snapshot Manager, not by deleting their files manually.

### 9. Package repair then failed because networking was unavailable

The next repair attempt reported:

```text
Temporary failure resolving 'http.kali.org'
```

Although presented as a name-resolution error, further testing showed no default route:

```bash
ip route
ping -c 3 1.1.1.1
getent hosts http.kali.org
nmcli device status
ip -br address
```

`eth1` had `192.168.186.129/24`, while `eth0` was disconnected with no IP address. Ping reported `Network is unreachable`. VMware's NAT adapter was connected, but its connection inside Kali was not activated.

The repair was:

```bash
sudo nmcli device connect eth0
```

Afterward:

- `eth0` acquired `192.168.236.128`.
- The default route became `default via 192.168.236.2 dev eth0`.
- All three pings to `1.1.1.1` succeeded.
- `getent hosts http.kali.org` returned an address.

This verified internet connectivity and name resolution at that time. These IP addresses were specific to this VMware NAT network and must not be copied blindly elsewhere. The final route listing no longer contained the earlier eth1 route; its reason was not established, and Host-only connectivity was not separately validated.

### 10. Completing package repair

The final instructed sequence was:

```bash
sudo apt --fix-broken install
sudo dpkg --configure -a
sudo apt-get check
sudo reboot
```

Each command was to finish successfully before the next. The user then reported success. Final command logs and the post-expansion `df` output were not supplied, so the precise record is user-confirmed overall recovery rather than individually archived verification of every final state.

## 3. Terminology, parameters, and reasons

### 1. Most important: three layers of capacity

**Virtual disk → partition → filesystem.** Enlarging the VMware disk exposes more sectors. The partition boundary and ext4 filesystem must also expand before Kali can use them. GParted coordinates partition and supported filesystem resizing.

`/dev/sda` is the whole disk; `sda1` is a partition; `/` is the root filesystem's mount point. `sda2` is an MBR extended container and `sda5` is its logical swap partition. Swap provides disk-backed memory support under memory pressure; it is not a project directory or equivalent to adding physical RAM.

A GiB is 1024³ bytes and a MiB is 1024² bytes. Filesystem metadata, reserved blocks, and existing files mean `df` available space will differ from partition capacity.

### 2. Preview versus execution

`Resize/Move` queues an operation. Green `Apply` executes disk changes. Pending operations can be undone; applied operations cannot be rolled back with the undo arrow. If applying fails, inspect Details rather than assuming nothing was changed.

Adjacent free space must be at the correct partition level. Space inside `sda2` cannot directly extend the external `sda1`. List indentation and container outlines help show this distinction.

### 3. Command reference

| Command or option | Purpose |
|---|---|
| `df -h /` | Root filesystem usage; `-h` selects readable units |
| `lsblk -o ...` | Disk/partition/mount relationships; `-o` selects columns |
| `fdisk -l /dev/sda` | Read-only listing of that disk's partition table |
| `parted ... unit s print free` | Read-only sector-based partition and free-space listing |
| `systemctl --failed --no-pager` | Failed services without an interactive pager |
| `apt clean` | Remove downloaded package archives, not installed packages |
| `apt --fix-broken install` | Attempt dependency repair and required package actions; review the proposed changes |
| `dpkg --configure -a` | Configure unpacked packages awaiting configuration |
| `apt-get check` | Check package dependency state |
| `ip route` | Display routes, particularly the default gateway |
| `ip -br address` | Brief interface state and address listing |
| `ping -c 3` | Send three probes; failure alone does not always prove total disconnection |
| `getent hosts` | Resolve a hostname through the system's name-service configuration |
| `nmcli device connect eth0` | Ask NetworkManager to activate a connection on that interface |

A root `#` prompt normally needs no additional sudo. At a regular user's `$` prompt, administrative commands use sudo. Do not copy the prompt itself.

### 4. Four networking layers

VMware's Connected setting is the virtual cable. Kali still needs a connection profile, an address, and routes. NAT normally supplies external access through the host; Host-only supplies a private host/guest network. A default route sends traffic to destinations outside directly connected networks. DNS translates names into addresses.

Activating eth0 restored both routing and resolution here. Changing package mirrors or manually overriding DNS was therefore unnecessary. `--fix-missing` cannot supply a missing default route.

## 4. Issues encountered: chronological list, then categories

### Chronological index

| ID | Issue | Resolution or status |
|---|---|---|
| 01 | Original Kali VM could not be found | Old folder empty; original files' location remained unconfirmed |
| 02 | Uncertainty about installation and capacity | Configuration guidance and project-oriented capacity planning |
| 03 | MariaDB failed its free-space check | Cleared cache, then expanded root storage |
| 04 | Black screen and failed lightdm | Used text console; addressed space and unfinished packages |
| 05 | 80 GiB disk but only 19 GiB root partition | Distinguished disk, partition, and filesystem capacity |
| 06 | Two commands entered as one | Ran df and fdisk on separate lines |
| 07 | Attached GParted ISO still booted Kali | Changed firmware boot order |
| 08 | Unfamiliar Live keyboard/mode choices | Default keymap and graphical mode 0 |
| 09 | Why not expand sda1 directly? | Swap/container separated it from free space |
| 10 | Confusion between moving and resizing swap | Undid pending changes and checked size/location |
| 11 | Repeated GParted boundary error | Inspected sectors; None alignment on sda2 allowed the operation |
| 12 | Uncertainty about preview versus disk state | Used pending-operation list and fdisk/parted readings |
| 13 | Whether to replace ISO or delete backups | Disconnect optical media; validate before backup cleanup |
| 14 | APT hostname resolution failure | Investigated routing rather than only DNS |
| 15 | Connected NAT adapter but no eth0 address | Activated eth0 using nmcli |
| 16 | 480 unfinished packages | Completed package repair after restoring networking; user confirmed success |

### Classification, retaining chronological IDs

- **VM files and backups:** 01, 13.
- **Installation and capacity planning:** 02, 05.
- **Storage exhaustion, packages, and desktop:** 03, 04, 16.
- **Command entry and tool interpretation:** 06, 08, 12.
- **Boot media and firmware:** 07, 13.
- **Partition structure and GParted:** 09, 10, 11, 12.
- **Networking and DNS:** 14, 15.

## 5. Avoiding repeated troubleshooting

- Back up the whole VM directory while powered off and distinguish that copy from the active VM.
- Save `fdisk -l`, `lsblk`, and `df -h` output before partition changes. Never reuse this disk's sector numbers on another disk.
- Allocating 100–120 GB for ordinary projects, or 150–200 GB for more containers and packet captures, is a planning recommendation, not a Kali requirement. This VM's actual disk remained 80 GiB.
- Keeping roughly 20 GB free is a practical maintenance target, not a fixed MariaDB requirement.
- When correct inputs reproduce an error, collect underlying state rather than repeatedly dragging the same boundary.
- Finish repair before starting another broad upgrade. `781 not upgrading` did not mean 781 broken packages.

## References

- [Kali installation sizes](https://www.kali.org/docs/installation/installation-sizes/)
- [MariaDB 11.8.8-1 pre-installation script](https://sources.debian.org/src/mariadb/1%3A11.8.8-1/debian/mariadb-server.preinst)
- [GParted manual: operations, resizing, and alignment](https://gparted.org/display-doc.php?name=help-manual)
- [GParted: moving space between partitions](https://gparted.org/display-doc.php?name=moving-space-between-partitions)

Observed states and values come from the supplied screenshots and terminal output. Unconfirmed causal explanations are explicitly identified above.
