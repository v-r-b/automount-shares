# automount-shares

Mount different network shares on linux machines via /etc/network/if-up, if-down (sshfs, cifs, ...)

## Abstract

automount-shares works with shell scripts in /etc/network/if-up, if-down, which are automatically called whenever a network connection comes up or goes down.
 - [`automountshares`](etc/network/if-up.d/automountshares) will be installed in `/etc/network/if-up.d/`
 - [`autoumountshares`](etc/network/if-down.d/autoumountshares) will be installed in `/etc/network/if-down.d/` and `/etc/network/if-post-down.d/`
 - [`automountshares-common`](etc/network/if-up.d/automountshares-common) contains functions which are used by the mount- and unmount-scripts. This file will be installed in `/etc/network/if-up.d/`

The scripts will automatically mount/unmount network shares which are tagged inside `/etc/fstab` in a special way (see below: [Markup](./README.md#marking-down-shares-for-automatic-mountingunmounting)). 

**Mounting**: If the fstab entry is properly tagged, than the scripts will only try to mount shares if the apropriate route is established / if the target server is up. 

**Unmounting**: Unmounting is only done if the target network goes down. If the computer is connected to different networks, than the shares on networks which are still available, remain untouched.

## Credits 

The original idea can be found [on ubuntuforums.org](http://ubuntuforums.org/showthread.php?t=430312). It was published in 2010 by "Darwin Award Winner" and deals with sshfs connections in particular. The orginal scripts can be found inside the repository (etc/network/if-up.d/mountsshfs and etc/network/if-down.d/umountsshfs). This is for documentary reasons only. The files will not be used nor installed.

## Prerequisites

The mounting script uses fping, since it is "unwise", according to ping's [manpage](https://manpages.org/ping), to use ping from automated scripts. Unlike ping, fping is meant to be used in scripts ([see here](https://manpages.org/fping/8)). To use automount-shares, you will have to install fping:
```
sudo apt install fping
```

## Installation

After downloading or cloning the repository, call [`install.sh`](install.sh) as a superuser, since the files will be installed in system directories:
```
sudo install.sh
```
Then, tag the `fstab` entries for automounting (see below: [Markup](./README.md#marking-down-shares-for-automatic-mountingunmounting)). Done.

## Configuration

After installing fping, the scripts can be used out of the box. If you want to, you can configure the two scripts in the following ways:

 - logging: The scripts use logger. You can adjust
   - the log level by calling `set_log_level <level>` inside the scripts. 
     - `0` is no logging at all,
     - `1` logs the main information (default)
     - `2`..`4` increase verbosity.
   - if you use higher levels than 1, you can turn on indenting the log messages by `indent_logs true`.
   - if you want to use other tags for logger than the default ones (the script names), you can call `set_logtag <tag>`
 - selection of the mount points to be handled by the scripts. This tag will be used in `/etc/fstab` (see below: [Description](./README.md#description))
   - adjust `SELECTOR_STRING` in [`automountshares-common`](etc/network/if-up.d/automountshares-common) (default is `"netmount"`)
 - adjust the list of filesystems for which the scripts shall be used (see below: [Description](./README.md#description))
   - adjust `MOUNT_TYPES` in [`automountshares-common`](etc/network/if-up.d/automountshares-common) (default is `"fuse.sshfs,cifs"`)

## Marking down shares for automatic mounting/unmounting

Shares that shall be handled by the mounting/unmounting scripts, must be marked down in `/etc/fstab`. Therefore adjust the mounting options (4th column in `fstab`, usually containing entries like `defaults,noauto`etc.) as follows:
 - add `noauto`, since the shares shall not be mounted by the system but by our scripts
 - add `comment=netmount` to tell the scripts that they shall handle this `fstab` entry. <br/>
   You can use another tag than `"netmount"`by adjusting `SELECTOR_STRING` in [`automountshares-common`](etc/network/if-up.d/automountshares-common) (see above: [Configuration](./README.md#configuration))
 - optionally, `"netmount"`can be followed by a route and a host, e.g. `comment=netmount:192.168.0.0:my.host` or `comment=netmount:192.168.0.0:192.168.0.200`. For details see below: [Description](./README.md#description).

## Description

Each time a network interface comes up / goes down, the system calls the scripts inside `/etc/network/if-pre-up.d, if-up.d, if-down.d, if-post-down.d/`. This is where our scripts [`automountshares`](etc/network/if-up.d/automountshares) and [`autoumountshares`](etc/network/if-down.d/autoumountshares) are installed. They work as follws:

### automountshares - automatic mounting

The scripts traverses `/etc/fstab` line by line. If it finds a mount point whose options are tagged by `comment=netmount`, 
 - it looks for an optional **route** entry. This is the IP of the destination network, appended to `"netmount"`, together with a colon, so we get, e.g. `comment=netmount:192.168.0.0`. If such a route entry is found, the script checks for the existance of the route, before attempting to mount the share. The test is done by calling `route -n | awk '{ print $1 }' | grep <route>`
 - it looks for an optional **host** entry. This can be the IP adress or the hostname of the server containing the share. The host entry is either appended to an existing route entry (e.g. `comment=netmount:192.168.0.0:192.168.0.200` or `comment=netmount:192.168.0.0:my.host.name`) or, if the route is omitted, to the "`netmount`" tag, using two colons (e.g. `comment=netmount::192.168.0.200` or `comment=netmount::my.host.name`). If a host entry is found, the script only attempts to mount the share if the given host is up. The test is done by calling `fping -a -r 1 -t 500 <host>`. This is wny you need to install fping if you use host checking.

 After the optional tests are passed, the script makes sure that the share is not already mounted. This is done by getting the list of all mounts of the defined `MOUNT_TYPES`, which are by default `fuse.sshfs` and `cifs`. You can extend this list by configuring the variable `MOUNT_TYPES` (see above: [Configuration](./README.md#configuration)).

 If the share is not already mounted, a job is launched to do the mount: `sh -c "mount <mount point>`.

### autoumountshares - automatic unmounting

The scripts traverses `/etc/fstab` line by line. If it finds a mount point whose options are tagged by `comment=netmount`, 
 - it first makes sure that the share is indeed mounted. If this is not the case, nothing is to be done.
 - it looks for an optional **route** entry. This is the IP of the destination network, appended to `"netmount"`, together with a colon, so we get, e.g. `comment=netmount:192.168.0.0`. If such a route entry is found, the script checks for the existance of the route. If the route does not exist anymore (due tu disconnecting the interface), the share is unmounted. If the route still exists, the share remains untouched.

 If the share is mounted and no route is given, the share is unmounted.
 
 For unmounting, a job is launched to do a lazy unmount: `sh -c "umount -l <mount point>`.
