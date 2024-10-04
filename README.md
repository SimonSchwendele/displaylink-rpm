# DisplayLink RPM

This is the recipe for building the [DisplayLink driver][displaylink]
in a RPM package for Fedora Linux. This driver supports the following
device families:

- DL-7xxx
- DL-6xxx
- DL-5xxx
- DL-41xx
- DL-3xxx

The package includes the Open Source [evdi] library.
Packages get automatically built by GitHub Actions and get uploaded to [releases] and [copr]

## Installation

The easiest way would be to install it using the [copr]

```bash
$ sudo dnf copr enable crashdummy/Displaylink
$ sudo dnf update 
$ sudo dnf install displaylink -y
```

This will install basically everything similiar to the new [apt repository](https://www.synaptics.com/products/displaylink-graphics/downloads/ubuntu) of synaptics.
You can verify everythings working.

```bash
$ dkms status
evdi/1.12.0, 6.3.0-0.rc7.20230423gt622322f5.361.vanilla.fc38.x86_64, x86_64: installed
evdi/1.12.0, 6.3.0-362.vanilla.fc38.x86_64, x86_64: installed

$ systemctl status displaylink-driver.service 
● displaylink-driver.service - DisplayLink Driver Service
     Loaded: loaded (/usr/lib/systemd/system/displaylink-driver.service; static)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Mon 2023-04-24 09:14:16 CEST; 2h 3min ago
    Process: 5512 ExecStartPre=/sbin/modprobe evdi (code=exited, status=0/SUCCESS)
   Main PID: 5515 (DisplayLinkMana)
      Tasks: 52 (limit: 37501)
     Memory: 199.0M
        CPU: 17min 43.068s
     CGroup: /system.slice/displaylink-driver.service
             └─5515 /usr/libexec/displaylink/DisplayLinkManager

Apr 24 09:14:16 crashphyrus systemd[1]: Starting displaylink-driver.service - DisplayLink Driver Service...
Apr 24 09:14:16 crashphyrus systemd[1]: Started displaylink-driver.service - DisplayLink Driver Service.
```

## Usage

> NOTE: Now buildable cleanly via .spec file (in mock f.e.). Download files
> via `make srpm`.

In order to create the driver rpm package you can run the command `make` from
within the checked out directory. The Makefile should download the files needed
for you and create an RPM.

A default `make` will use the evdi driver that is bundled with the Displaylink
driver package. If you need to use a newer released version from the evdi Github
repo and it is not currently present in the Displaylink driver package, you can
do so by running:

```bash
make github-release
```

### Secure boot on Fedora

If you dont have secure boot activated you can skip this part.
```
$ mokutil --sb-state
SecureBoot disabled
```

#### dkms >= 3.0.4

Since [dkms-3.0.4](https://github.com/dell/dkms/releases/tag/v3.0.4) signing now became a lot easier.
You can now dont need to create your signing key yourself.

```bash
$ sudo dnf install mokutil dkms

# Enroll the key dkms created automatically and enter a passphrase
$ sudo mokutil --import /var/lib/dkms/mok.pub

$ reboot
```

After the reboot you might be prompted at boot to enroll the key.
Enter the passphrase you provided in the last step.

Afterwards you can let dkms sign your module.

```bash
# Verify the enrollment success
$ mokutil --list-enrolled | grep DKMS
             Subject: CN=DKMS module signing key

# Signing should be done automatically during upgrades.
# If the module is not signed you can invoke it manually 
$ sudo dkms autoinstall
```

your evdi should now always be signed

```bash
$ sudo modinfo evdi
filename:       /lib/modules/6.11.1-206.fsync.fc40.x86_64/extra/evdi.ko.xz
license:        GPL
description:    Extensible Virtual Display Interface
author:         DisplayLink (UK) Ltd.
import_ns:      DMA_BUF
import_ns:      DMA_BUF
depends:        
retpoline:      Y
name:           evdi
vermagic:       6.11.1-206.fsync.fc40.x86_64 SMP preempt mod_unload 
sig_id:         PKCS#7
signer:         DKMS module signing key
sig_key:        53:D7:A1:A6:97:6A:CC:3F:46:FB:42:D1:17:B6:73:4F:F7:95:03:DA
sig_hashalgo:   sha512
signature:      9C:03:24:9C:DB:A8:85:8C:87:D6:09:0D:D1:80:17:18:A6:8A:3D:BB:
		FC:77:CD:A5:69:F7:7E:37:FC:AE:2E:52:B9:36:3F:4E:43:D5:08:19:
		A5:8B:86:A2:98:2C:83:64:A7:7A:BB:30:49:18:3F:B5:CF:3D:48:FF:
		C5:01:16:13:63:29:87:BA:1A:F2:AD:50:ED:F2:82:C2:95:B9:BC:E9:
		5C:F6:E6:3F:E2:EB:88:02:F9:9F:6B:84:1E:18:E9:DD:C3:B6:B7:89:
		5B:8F:53:74:BC:8A:34:D4:40:AB:C9:E6:C2:55:41:EA:A3:3E:59:0F:
		62:4C:8E:FC:35:6D:57:4C:8D:F0:3D:5F:86:35:02:12:E7:91:A2:48:
		04:5C:A4:AC:2A:1A:2A:A7:C2:05:0A:8E:21:CF:B3:F4:4F:76:7D:F9:
		99:85:98:83:B9:24:17:CF:A6:5A:E5:46:CE:DF:FC:63:10:15:7F:28:
		C0:8C:10:00:1B:B3:4D:8A:3B:F2:6E:7F:76:47:BC:F5:44:28:0F:4E:
		31:67:BC:4C:B7:07:D3:26:D9:79:99:74:99:E2:8B:8C:81:C1:FE:24:
		D4:8C:5A:77:E3:6A:3E:98:89:69:5E:75:9A:66:90:CC:48:69:BB:A4:
		91:31:25:1F:B6:B8:72:63:8A:12:C8:C7:6E:B8:E2:DB
parm:           initial_loglevel:Initial log level (int)
parm:           initial_device_count:Initial DRM device count (default: 0) (ushort)
```

#### legacy

To use displaylink-rpm and the evdi kernel module with secure boot enabled on
Fedora you need to sign the module with an enrolled Machine Owner Key (MOK).

First create a self signed MOK:

``` bash
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out \
MOK.der -nodes -days 36500 -subj "/CN=Displaylink/"
```

Then register the MOK with secure boot:

``` bash
sudo mokutil --import MOK.der
```

Then reboot your Fedora host and follow the instructions to enroll the key.

Now you can sign the evdi module. This must be done for every kernel upgrade:

``` bash
sudo modinfo -n evdi /lib/modules/5.10.19-200.fc33.x86_64/extra/evdi.ko.xz

sudo unxz $(modinfo -n evdi)

sudo /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 ./MOK.priv \
  ./MOK.der /lib/modules/$(uname -r)/extra/evdi.ko

sudo xz -f /lib/modules/$(uname -r)/extra/evdi.ko
```

Now any display, hdmi and/or dvi ports on your docking station should work,
and the displaylink-driver.service should run.

## Hardware-specific behavior

### Dell D6000

When used with the Dell D6000 docking station, DisplayLink 5.1.26 regularly
loses communication with attached monitors, causing them to go blank and enter
power-saving mode.  At the time the monitors blank, the kernel logs two error
messages:

``` bash
kernel: usb <xxx>: Disable of device-initiated U1 failed.
kernel: usb <xxx>: Disable of device-initiated U2 failed.
```

To [work around this issue][workaround], disable power management for the audio
device by commenting out a line in `/etc/pulse/default.pa`:

``` bash
### Automatically suspend sinks/sources that become idle for too long
# load-module module-suspend-on-idle
```

[workaround]: https://displaylink.org/forum/showpost.php?p=85116

## Development Builds

Generally we want to track the current stable release of the evdi library.
However, Fedora kernels are often much newer than those officially supported by
that release and it is not uncommon for a new kernel to completely break the
build. This can leave you in a situation where you cannot upgrade your kernel
without sacrificing your displaylink devices. This is not great if the new
kernel has important security or performance fixes.

The evdi developers use the `main` branch as their main branch for all changes.

To pull the latest code from the `main` branch and use it to build, do the
following:

``` bash
make main

make github-release
```

Of course this `main` branch will also include some experimental and less
tested changes that may break things in other unexpected ways. So you should
prefer the mainline build if it works, but if it breaks, you have the option of
making a `main` build.

If you are using Fedora Rawhide, you can create a build which will automatically
download from the `main` branch and build by running:

``` bash
make rawhide
```

> In the past, code in the `main` branch would be tagged and that version is what
> would be included in the Displaylink driver package.
>
> Recently, we are seeing newer changes appear in the Displaylink driver package
> without the evdi library version being changed. This has created some confusion
> and difficulty when it comes to maintenance updates.
>
> The evdi folks have [acknowledged this issue][roadmap_discussion] and are
> working on making the process more transparent.

[roadmap_discussion]: https://github.com/DisplayLink/evdi/issues/309#issuecomment-979831346

## Contributing

The easiest way to contribute with the package is to fork it and send
a pull request in GitHub.

There are two main kind of contributions: either a new upstream
version is released or a modification in the packaging is proposed.

There is a variable called `RELEASE` for packaging purposes. That
variable should be set to 1 when contributing a new upstream version
release, and incremented in one when adding any other functionality to
the specfile for the same upstream version.

### New Upstream release

From time to time, DisplayLink will update their driver. We try to do
so, but for that we usually rely on pull requests.

We manage three different upstream numbers for versioning:

1. evdi kernel driver version
2. DisplayLinkManager daemon and libraries version
3. Download ID number from DisplayLink (for automatic zip retrieval)

These variables need to be changed in the following places:

- Makefile
  - `DAEMON_VERSION` is the DisplayLinkManager version
  - `VERSION` is currently the evdi driver version
  - `DOWNLOAD_ID` is the `?download_id=` query parameter in
    DisplayLink website to download the zip

Also, please update the changelog at the bottom of the
displaylink.spec file.

### Packaging change

When changing a packaging rule, please increment the `RELEASE`
variable by one in displaylink.spec


[displaylink]: http://www.displaylink.com/
[evdi]: https://github.com/DisplayLink/evdi
[releases]: https://github.com/SimonSchwendele/displaylink-rpm/releases
[copr]: https://copr.fedorainfracloud.org/coprs/crashdummy/Displaylink/
