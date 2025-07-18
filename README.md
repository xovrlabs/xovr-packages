## The XBPS Source Packages Collection

This repository contains the XBPS source packages collection used to build binary packages for the **XOVR Linux** distribution.

**XOVR Linux** is an independent fork of Void Linux, maintained by the XOVR Labs team. It introduces updated tooling and infrastructure tailored for a sovereign future — infrastructure owned and controlled by **you**, collectively.

The included `xbps-src` script will fetch and compile the sources, and install its
files into a `fake destdir` to generate XBPS binary packages that can be installed
or queried through the `xbps-install(1)` and `xbps-query(1)` utilities, respectively.

See [Contributing](./CONTRIBUTING.md) for a general overview of how to contribute and the
[Manual](./Manual.md) for details of how to create source packages.

### Table of Contents

- [Requirements](#requirements)
- [Quick start](#quick-start)
- [chroot methods](#chroot-methods)
- [Install the bootstrap packages](#install-bootstrap)
- [Configuration](#configuration)
- [Directory hierarchy](#directory-hierarchy)
- [Building packages](#building-packages)
- [Package build options](#build-options)
- [Sharing and signing your local repositories](#sharing-and-signing)
- [Rebuilding and overwriting existing local packages](#rebuilding)
- [Enabling distcc for distributed compilation](#distcc)
- [Distfiles mirrors](#distfiles-mirrors)
- [Cross compiling packages for a target architecture](#cross-compiling)
- [Using xbps-src in a foreign Linux distribution](#foreign)
- [Remaking the masterdir](#remaking-masterdir)
- [Keeping your masterdir uptodate](#updating-masterdir)
- [Building 32bit packages on x86_64](#building-32bit)
- [Building packages natively for the musl C library](#building-for-musl)
- [Building xovr base-system from scratch](#building-base-system)

### Requirements

- GNU bash
- xbps >= 0.56
- git(1) - unless configured to not, see etc/defaults.conf
- common POSIX utilities included by default in almost all UNIX systems
- curl(1) - required by `xbps-src update-check`

For bootstrapping additionally:
- flock(1) - util-linux
- bsdtar or GNU tar (in that order of preference)
- install(1) - GNU coreutils
- objcopy(1), objdump(1), strip(1): binutils

`xbps-src` requires [a utility to chroot](#chroot-methods) and bind mount existing directories
into a `masterdir` that is used as its main `chroot` directory. `xbps-src` supports
multiple utilities to accomplish this task.

> NOTE: `xbps-src` does not allow building as root anymore. Use one of the chroot
methods.

<a name="quick-start"></a>
### Quick start

Clone the `xovr-packages` git repository and install the bootstrap packages:

```
$ git clone https://github.com/xovrlabs/xovr-packages.git
$ cd xovr-packages
$ ./xbps-src binary-bootstrap
```

Build a package by specifying the `pkg` target and the package name:

```
$ ./xbps-src pkg <package_name>
```

Use `./xbps-src -h` to list all available targets and options.

To build packages marked as 'restricted', modify `etc/conf`:

```
$ echo XBPS_ALLOW_RESTRICTED=yes >> etc/conf
```

Once built, the package will be available in `hostdir/binpkgs` or an appropriate subdirectory (e.g. `hostdir/binpkgs/nonfree`). To install the package:

```
# xbps-install --repository hostdir/binpkgs <package_name>
```

Alternatively, packages can be installed with the `xi` utility, from the `xtools` package. `xi` takes the repository of the current working directory into account.

```
$ xi <package_name>
```

<a name="chroot-methods"></a>
### chroot methods

#### xbps-uunshare(1) (default)

XBPS utility that uses `user_namespaces(7)` (part of xbps, default without `-t` flag).

This utility requires these Linux kernel options:

- CONFIG\_NAMESPACES
- CONFIG\_IPC\_NS
- CONFIG\_UTS\_NS
- CONFIG\_USER\_NS

This is the default method, and if your system does not support any of the required kernel
options it will fail with `EINVAL (Invalid argument)`.

#### xbps-uchroot(1)

XBPS utility that uses `namespaces` and must be `setgid` (part of xbps).

> NOTE: This is the only method that implements functionality of `xbps-src -t`, therefore the
flag ignores the choice made in configuration files and enables `xbps-uchroot`.

This utility requires these Linux kernel options:

- CONFIG\_NAMESPACES
- CONFIG\_IPC\_NS
- CONFIG\_PID\_NS
- CONFIG\_UTS\_NS

Your user must be added to a special group to be able to use `xbps-uchroot(1)` and the
executable must be `setgid`:

    # chown root:<group> xbps-uchroot
    # chmod 4750 xbps-uchroot
    # usermod -a -G <group> <user>

> NOTE: by default in xovr you shouldn't do this manually, your user must be a member of
the `xbuilder` group.

To enable it:

    $ cd xovr-packages
    $ echo XBPS_CHROOT_CMD=uchroot >> etc/conf

If for some reason it's erroring out as `ERROR clone (Operation not permitted)`, check that
your user is a member of the required `group` and that `xbps-uchroot(1)` utility has the
proper permissions and owner/group as explained above.

#### bwrap(1)

bubblewrap, sandboxing tool for unprivileged users that uses
user namespaces or setuid.
See <https://github.com/containers/bubblewrap>.

#### ethereal

Destroys host system it runs on. Only useful for one-shot containers, i.e docker (used with CI).

<a name="install-bootstrap"></a>
### Install the bootstrap packages

There is a set of packages that makes up the initial build container, called the `bootstrap`.
These packages are installed into the `masterdir` in order to create the container.

The primary and recommended way to set up this container is using the `binary-bootstrap`
command. This will use pre-existing binary packages, either from remote `xbps` repositories
or from your local repository.

There is also the `bootstrap` command, which will build all necessary `bootstrap` packages from
scratch. This is usually not recommended, since those packages are built using your host system's
toolchain and are neither fully featured nor reproducible (your host system may influence the
build) and thus should only be used as a stage 0 for bootstrapping new XOVR systems.

If you still choose to use `bootstrap`, use the resulting stage 0 container to rebuild all
`bootstrap` packages again, then use `binary-bootstrap` (stage 1) and rebuild the `bootstrap`
packages once more (to gain stage 2, and then use `binary-bootstrap` again). Once you've done
that, you will have a `bootstrap` set equivalent to using `binary-bootstrap` in the first place.

Also keep in mind that a full source `bootstrap` is time consuming and will require having an
assortment of utilities installed in your host system, such as `binutils`, `gcc`, `perl`,
`texinfo` and others.

### Configuration

The `etc/defaults.conf` file contains the possible settings that can be overridden
through the `etc/conf` configuration file for the `xbps-src` utility; if that file
does not exist, will try to read configuration settings from `$XDG_CONFIG_HOME/xbps-src.conf`, `~/.config/xbps-src.conf`, `~/.xbps-src.conf`.

If you want to customize default `CFLAGS`, `CXXFLAGS` and `LDFLAGS`, don't override
those defined in `etc/defaults.conf`, set them on `etc/conf` instead i.e:

    $ echo 'XBPS_CFLAGS="your flags here"' >> etc/conf
    $ echo 'XBPS_LDFLAGS="your flags here"' >> etc/conf

Native and cross compiler/linker flags are set per architecture in `common/build-profiles`
and `common/cross-profiles` respectively. Ideally those settings are good enough by default,
and there's no need to set your own unless you know what you are doing.

#### Virtual packages

The `etc/defaults.virtual` file contains the default replacements for virtual packages,
used as dependencies in the source packages tree.

If you want to customize those replacements, copy `etc/defaults.virtual` to `etc/virtual`
and edit it accordingly to your needs.

<a name="directory-hierarchy"></a>
### Directory hierarchy

The following directory hierarchy is used with a default configuration file:

         /xovr-packages
            |- common
            |- etc
            |- srcpkgs
            |  |- xbps
            |     |- template
            |
            |- hostdir
            |  |- binpkgs ...
            |  |- ccache ...
            |  |- distcc-<arch> ...
            |  |- repocache ...
            |  |- sources ...
            |
            |- masterdir-<arch>
            |  |- builddir -> ...
            |  |- destdir -> ...
            |  |- host -> bind mounted from <hostdir>
            |  |- xovr-packages -> bind mounted from <xovr-packages>


The description of these directories is as follows:

 - `masterdir-<arch>`: master directory to be used as rootfs to build/install packages.
 - `builddir`: to unpack package source tarballs and where packages are built.
 - `destdir`: to install packages, aka **fake destdir**.
 - `hostdir/ccache`: to store ccache data if the `XBPS_CCACHE` option is enabled.
 - `hostdir/distcc-<arch>`: to store distcc data if the `XBPS_DISTCC` option is enabled.
 - `hostdir/repocache`: to store binary packages from remote repositories.
 - `hostdir/sources`: to store package sources.
 - `hostdir/binpkgs`: local repository to store generated binary packages.

<a name="building-packages"></a>
### Building packages

The simplest form of building package is accomplished by running the `pkg` target in `xbps-src`:

```
$ cd xovr-packages
$ ./xbps-src pkg <pkgname>
```

When the package and its required dependencies are built, the binary packages will be created
and registered in the default local repository at `hostdir/binpkgs`; the path to this local repository can be added to
any xbps configuration file (see xbps.d(5)) or by explicitly appending them via cmdline, i.e:

    $ xbps-install --repository=hostdir/binpkgs ...
    $ xbps-query --repository=hostdir/binpkgs ...

By default **xbps-src** will try to resolve package dependencies in this order:

 - If a dependency exists in the local repository, use it (`hostdir/binpkgs`).
 - If a dependency exists in a remote repository, use it.
 - If a dependency exists in a source package, use it.

It is possible to avoid using remote repositories completely by using the `-N` flag.

> The default local repository may contain multiple *sub-repositories*: `debug`, `multilib`, etc.

<a name="build-options"></a>
### Package build options

The supported build options for a source package can be shown with `xbps-src show-options`:

    $ ./xbps-src show-options foo

Build options can be enabled with the `-o` flag of `xbps-src`:

    $ ./xbps-src -o option,option1 pkg foo

Build options can be disabled by prefixing them with `~`:

    $ ./xbps-src -o ~option,~option1 pkg foo

Both ways can be used together to enable and/or disable multiple options
at the same time with `xbps-src`:

    $ ./xbps-src -o option,~option1,~option2 pkg foo

The build options can also be shown for binary packages via `xbps-query(1)`:

    $ xbps-query -R --property=build-options foo

> NOTE: if you build a package with a custom option, and that package is available
in an official xovr repository, an update will ignore those options. Put that package
on `hold` mode via `xbps-pkgdb(1)`, i.e `xbps-pkgdb -m hold foo` to ignore updates
with `xbps-install -u`. Once the package is on `hold`, the only way to update it
is by declaring it explicitly: `xbps-install -u foo`.

Permanent global package build options can be set via `XBPS_PKG_OPTIONS` variable in the
`etc/conf` configuration file. Per package build options can be set via
`XBPS_PKG_OPTIONS_<pkgname>`.

> NOTE: if `pkgname` contains `dashes`, those should be replaced by `underscores`
i.e `XBPS_PKG_OPTIONS_xorg_server=opt`.

The list of supported package build options and its description is defined in the
`common/options.description` file or in the `template` file.

<a name="sharing-and-signing"></a>
### Sharing and signing your local repositories

To share a local repository remotely it's mandatory to sign it and the binary packages
stored on it. This is accomplished with the `xbps-rindex(1)` utility.

First a RSA key must be created with `openssl(1)` or `ssh-keygen(1)`:

	$ openssl genrsa -des3 -out privkey.pem 4096

or

	$ ssh-keygen -t rsa -b 4096 -m PEM -f privkey.pem

> Only RSA keys in PEM format are currently accepted by xbps.

Once the RSA private key is ready you can use it to initialize the repository metadata:

	$ xbps-rindex --sign --signedby "I'm Groot" --privkey privkey.pem $PWD/hostdir/binpkgs

And then make a signature per package:

	$ xbps-rindex --sign-pkg --privkey privkey.pem $PWD/hostdir/binpkgs/*.xbps

> If --privkey is unset, it defaults to `~/.ssh/id_rsa`.

If the RSA key was protected with a passphrase you'll have to type it, or alternatively set
it via the `XBPS_PASSPHRASE` environment variable.

Once the binary packages have been signed, check if the repository contains the appropriate `hex fingerprint`:

	$ xbps-query --repository=hostdir/binpkgs -vL
	...

Each time a binary package is created, a package signature must be created with `--sign-pkg`.

> It is not possible to sign a repository with multiple RSA keys.

If packages in `hostdir/binpkgs` are signed, the key in `.plist` format (as imported by xbps) can be placed
in `etc/repo-keys/` to prevent xbps-src from prompting to import that key.

<a name="rebuilding"></a>
### Rebuilding and overwriting existing local packages

Packages are overwritten on every build to make getting package with changed build options easy.
To make xbps-src skip build and preserve first package build with given version and revision,
same as in official xovr repository, set `XBPS_PRESERVE_PKGS=yes` in `etc/conf` file.

Reinstalling a package in your target `rootdir` can be easily done too:

    $ xbps-install --repository=/path/to/local/repo -yf xbps-0.25_1

Using `-f` flag twice will overwrite configuration files.

> Please note that the `package expression` must be properly defined to explicitly pick up
the package from the desired repository.

<a name="distcc"></a>
### Enabling distcc for distributed compilation

Setup the workers (machines that will compile the code):

    # xbps-install -Sy distcc
    
Update distcc compiler whitelist

    # update-distcc-symlinks

Modify the configuration to allow your local network machines to use distcc (e.g. `192.168.2.0/24`):

    # echo "192.168.2.0/24" >> /etc/distcc/clients.allow

Enable and start the `distccd` service:

    # ln -s /etc/sv/distccd /var/service

Install distcc on the host (machine that executes xbps-src) as well.
Unless you want to use the host as worker from other machines, there is no need
to modify the configuration.

On the host you can now enable distcc in the `xovr-packages/etc/conf` file:

    XBPS_DISTCC=yes
    XBPS_DISTCC_HOSTS="localhost/2 --localslots_cpp=24 192.168.2.101/9 192.168.2.102/2"
    XBPS_MAKEJOBS=16

The example values assume a localhost CPU with 4 cores of which at most 2 are used for compiler jobs.
The number of slots for preprocessor jobs is set to 24 in order to have enough preprocessed data for other CPUs to compile.
The worker 192.168.2.101 has a CPU with 8 cores and the /9 for the number of jobs is a saturating choice.
The worker 192.168.2.102 is set to run at most 2 compile jobs to keep its load low, even if its CPU has 4 cores.
The XBPS_MAKEJOBS setting is increased to 16 to account for the possible parallelism (2 + 9 + 2 + some slack).

<a name="distfiles-mirrors"></a>
### Distfiles mirror(s)

In etc/conf you may optionally define a mirror or a list of mirrors to search for distfiles.

    $ echo 'XBPS_DISTFILES_MIRROR="ftp://192.168.100.5/gentoo/distfiles"' >> etc/conf

If more than one mirror is to be searched, you can either specify multiple URLs separated
with blanks, or add to the variable like this

    $ echo 'XBPS_DISTFILES_MIRROR+="COMING SOON!"' >> etc/conf

Make sure to put the blank after the first double quote in this case.

The mirrors are searched in order for the distfiles to build a package until the
checksum of the downloaded file matches the one specified in the template.

Ultimately, if no mirror carries the distfile, or in case all downloads failed the
checksum verification, the original download location is used.

If you use `uchroot` for your XBPS_CHROOT_CMD, you may also specify a local path
using the `file://` prefix or simply an absolute path on your build host (e.g. /mnt/distfiles).
Mirror locations specified this way are bind mounted inside the chroot environment
under $XBPS_MASTERDIR and searched for distfiles just the same as remote locations.

<a name="cross-compiling"></a>
### Cross compiling packages for a target architecture

Currently `xbps-src` can cross build packages for some target architectures with a cross compiler.
The supported target is shown with `./xbps-src -h`.

If a source package has been adapted to be **cross buildable** `xbps-src` will automatically build the binary package(s) with a simple command:

    $ ./xbps-src -a <target> pkg <pkgname>

If the build for whatever reason fails, might be a new build issue or simply because it hasn't been adapted to be **cross compiled**.

<a name="foreign"></a>
### Using xbps-src in a foreign Linux distribution

xbps-src can be used in any recent Linux distribution matching the CPU architecture.

To use xbps-src in your Linux distribution use the following instructions. Let's start downloading the xbps static binaries:

    $ wget http://repo-default.xovrlinux.org/static/xbps-static-latest.<arch>-musl.tar.xz
    $ mkdir ~/XBPS
    $ tar xvf xbps-static-latest.<arch>-musl.tar.xz -C ~/XBPS
    $ export PATH=~/XBPS/usr/bin:$PATH

If `xbps-uunshare` does not work because of lack of `user_namespaces(7)` support,
try other [chroot methods](#chroot-methods).

Clone the `xovr-packages` git repository:

    $ git clone https://github.com/xovrlabs/xovr-packages.git

and `xbps-src` should be fully functional; just start the `bootstrap` process, i.e:

    $ ./xbps-src binary-bootstrap

The default masterdir is created in the current working directory, i.e. `xovr-packages/masterdir-<arch>`, where `<arch>` for the default masterdir is is the native xbps architecture.

<a name="remaking-masterdir"></a>
### Remaking the masterdir

If for some reason you must update xbps-src and the `bootstrap-update` target is not enough, it's possible to recreate a masterdir with two simple commands (please note that `zap` keeps your `ccache/distcc/host` directories intact):

    $ ./xbps-src zap
    $ ./xbps-src binary-bootstrap

<a name="updating-masterdir"></a>
### Keeping your masterdir uptodate

Sometimes the bootstrap packages must be updated to the latest available version in repositories, this is accomplished with the `bootstrap-update` target:

    $ ./xbps-src bootstrap-update

<a name="building-32bit"></a>
### Building 32bit packages on x86_64

Two ways are available to build 32bit packages on x86\_64:

 - native mode with a 32bit masterdir (recommended, used in official repository)
 - cross compilation mode to i686 [target](#cross-compiling)

The canonical mode (native) needs a new x86 `masterdir`:

    $ ./xbps-src -A i686 binary-bootstrap
    $ ./xbps-src -A i686 ...

<a name="building-for-musl"></a>
### Building packages natively for the musl C library

The canonical way of building packages for same architecture but different C library is through a dedicated masterdir by using the host architecture flag `-A`.
To build for x86_64-musl on glibc x86_64 system, prepare a new masterdir with the musl packages:

    $ ./xbps-src -A x86_64-musl binary-bootstrap

This will create and bootstrap a new masterdir called `masterdir-x86_64-musl` that will be used when `-A x86_64-musl` is specified.
Your new masterdir is now ready to build packages natively for the musl C library:

    $ ./xbps-src -A x86_64-musl pkg ...

<a name="building-base-system"></a>
### Building xovr base-system from scratch

To rebuild all packages in `base-system` for your native architecture:

    $ ./xbps-src -N pkg base-system

It's also possible to cross compile everything from scratch:

    $ ./xbps-src -a <target> -N pkg base-system

Once the build has finished, you can specify the path to the local repository to `xovr-mklive`, i.e:

    # cd xovr-mklive
    # make
    # ./mklive.sh ... -r /path/to/hostdir/binpkgs
