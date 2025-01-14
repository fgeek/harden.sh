harden.yml
==========

Ansible playbook to harden your Linux system.

Supported distros
-----------------

* Debian (Bullseye)
    * Kali
    * Raspbian
* Slackware (>= 15.0)

Why I made this
---------------

* [Bastille](http://bastille-linux.sourceforge.net/) is obsolete
* Not a member of [CIS](http://www.cisecurity.org/), so no downloading of the ready made scripts
* For learning
* For minimizing the effort needed to tweak fresh installations
    * Also for consistency

What does it do?
----------------

For a complete list you can run `ansible-playbook --list-tasks harden.yml`.

### Network

* Enables [TCP wrappers](https://en.wikipedia.org/wiki/TCP_Wrapper)
* IP stack hardening via sysctl settings
* Creates a basic firewall

### Logging

* Configure log retention time to be 6 months
* Run `ansible-playbook --list-tasks --tags logging harden.yml` for a full list

### Accounting

* Enables system accounting ([sysstat](http://sebastien.godard.pagesperso-orange.fr/))
    * Sets it's log retention to 99999 days (the logs are really small, so it doesn't eat up disk space)
* Enables process accounting

### Kernel

* Disables the use of certain kernel modules via `modprobe`
    * Disable [Firewire](http://www.hermann-uwe.de/blog/physical-memory-attacks-via-firewire-dma-part-1-overview-and-mitigation)
* [sysctl](https://en.wikipedia.org/wiki/Sysctl) settings hardening
    * Enables [SAK](https://en.wikipedia.org/wiki/Secure_attention_key) and disables the other [magic SysRq stuff](https://www.kernel.org/doc/Documentation/sysrq.txt)
    * Restricts the use of `dmesg` by regular users
    * Enable [YAMA](https://www.kernel.org/doc/Documentation/security/Yama.txt)
    * For the complete list, see [sysctl.conf.new](https://github.com/pyllyukko/harden.sh/blob/master/newconfs/sysctl.d/sysctl.conf.new)

### Filesystem

* Hardens mount options (creates `/etc/fstab.new`)
* Sets strict permissions to users home directories
* Limits permissions to various configuration files and directories that might contain sensitive content (see `permissions` tag for a complete list)
* Clean up `/tmp` during boot

### Application specific

* Configures basic auditing based on [stig.rules](https://fedorahosted.org/audit/browser/trunk/contrib/stig.rules) if audit is installed
* Configures `sshd_config` and `ssh_config`
* Configures [sudo](https://www.sudo.ws/)
* [ClamAV](https://www.clamav.net/) configuration
* [rkhunter](https://sourceforge.net/projects/rkhunter/) configuration
* [Lynis](https://cisofy.com/lynis/) configuration
* Display managers:
    * Disables user lists in GDM3 & LightDM
    * Disables guest sessions and VNC in LightDM

### User accounts / authentication / authorization

* Create a strict `securetty`
* Sets default [umask](https://en.wikipedia.org/wiki/Umask) to a more stricter `077`
* Sets console session timeout via `$TMOUT` (Bash)
* Creates `/etc/ftpusers`
* Restricts the use of [cron](https://en.wikipedia.org/wiki/Cron) and `at`
* Properly locks down system accounts (0 - `SYS_UID_MAX` && !`root`)
    * Lock the user's password
    * Sets shell to `/sbin/nologin`
    * Expire the account
* Configures the default password inactivity period
    * Run `ansible-playbook --list-tasks --tags passwords harden.yml` to list all password related tasks

#### PAM

* Configures `/etc/security/namespace.conf`
* Configures `/etc/security/access.conf`
* Configures `/etc/security/pwquality.conf` if available
* Require [pam\_wheel](http://linux-pam.org/Linux-PAM-html/sag-pam_wheel.html) in `/etc/pam.d/su`
* Creates a secure [/etc/pam.d/other](http://linux-pam.org/Linux-PAM-html/sag-security-issues-other.html)
* Run `ansible-playbook --list-tasks --tags pam harden.yml` to list all PAM related tasks

### Miscellaneous

* Creates legal banners
* Disable [core dumps](https://en.wikipedia.org/wiki/Core_dump) in `/etc/security/limits.conf`
* Reduce the amount of trusted [CAs](https://en.wikipedia.org/wiki/Certificate_authority)

### Slackware specific

Run `ansible-playbook --list-tasks --tags slackware harden.yml` for a list.

### Debian specific

* Configure AIDE
* Disables unnecessary systemd services
* Enables AppArmor
* Configure `SUITE` in `debsecan`
* Installs a bunch of security related packages
* Configures `chkrootkit` and enables daily checks

Creates bunch of `pam-config`s that are toggleable with `pam-auth-update`:

| PAM module                                                                                   | Type           | Description                                                                             |
| -------------------------------------------------------------------------------------------- | -------------- | --------------------------------------------------------------------------------------- |
| [pam\_wheel](http://www.linux-pam.org/Linux-PAM-html/sag-pam_wheel.html)[<sup>1</sup>](#fn1) | auth           | Require `wheel` group membership (`su`)                                                 |
| [pam\_succeed\_if](http://www.linux-pam.org/Linux-PAM-html/sag-pam_succeed_if.html)          | auth & account | Require UID >= 1000 && UID <= 60000 (or 0 & `login`)                                    |
| [pam\_unix](http://www.linux-pam.org/Linux-PAM-html/sag-pam_unix.html)[<sup>1</sup>](#fn1)   | auth           | Remove `nullok`                                                                         |
| [pam\_faildelay](http://www.linux-pam.org/Linux-PAM-html/sag-pam_faildelay.html)             | auth           | Delay on authentication failure                                                         |
| `pam_faillock`                                                                               | auth & account | Deter brute-force attacks                                                               |
| [pam\_access](http://linux-pam.org/Linux-PAM-html/sag-pam_access.html)                       | account        | Use login ACL (`/etc/security/access.conf`)                                             |
| [pam\_time](http://www.linux-pam.org/Linux-PAM-html/sag-pam_time.html)                       | account        | `/etc/security/time.conf`                                                               |
| [pam\_lastlog](http://www.linux-pam.org/Linux-PAM-html/sag-pam_lastlog.html)                 | account        | Lock out inactive users (no login in 90 days)                                           |
| [pam\_namespace](http://www.linux-pam.org/Linux-PAM-html/sag-pam_namespace.html)             | session        | Polyinstantiated temp directories                                                       |
| [pam\_umask](http://www.linux-pam.org/Linux-PAM-html/sag-pam_umask.html)                     | session        | Set file mode creation mask                                                             |
| [pam\_lastlog](http://www.linux-pam.org/Linux-PAM-html/sag-pam_lastlog.html)                 | session        | Display info about last login and update the lastlog and wtmp files[<sup>2</sup>](#fn2) |
| [pam\_pwhistory](http://www.linux-pam.org/Linux-PAM-html/sag-pam_pwhistory.html)             | password       | Limit password reuse                                                                    |

1. <span id="fn1"/>Not a `pam-config`, but a modification to existing `/etc/pam.d/` files
2. <span id="fn2"/>For all login methods and not just the console login

LXC tests
---------

* In order to build Debian container in Slackware you need [debootstrap](https://slackbuilds.org/repository/14.2/system/debootstrap/)
* It doesn't work the other way around, so it's not currently possible to build the Slackware container in Debian because it lacks Slackware's `pkgtools`

In order to run the LXC tests (`lxc.yml`), you need to configure SSH as described in [this post](https://gauvain.pocentek.net/ansible-to-deploy-lxc-containers.html):

```
Host 10.0.3.*
        StrictHostKeyChecking no
        UserKnownHostsFile=/dev/null
```

Tags
----

Tags that you can use with `ansible-playbook --tags`:

* `pki`
* `kernel`
* `rng`
* Specific software:
    * `sysstat`
    * `ssh`
    * `rkhunter`
    * `aide`
* `passwords`
* `pam`?

Other tags are just metadata for now.

References
----------

### Hardening guides

Some of these documents are quite old, but most of the stuff still applies.

* [CIS Slackware Linux 10.2 Benchmark v1.1.0][1]
* [Slackware System Hardening][2] by Jeffrey Denton
* [CIS Debian Linux Benchmark](https://www.cisecurity.org/benchmark/debian_linux/)
* [CIS CentOS Linux 7 Benchmark](https://www.cisecurity.org/benchmark/centos_linux/)
* [SlackDocs: Security HOWTOs](http://docs.slackware.com/howtos:security:start)
* [Alien's Wiki: Security issues](http://alien.slackbook.org/dokuwiki/doku.php?id=linux:admin#security_issues)
* [SlackWiki: Basic Security Fixes](http://slackwiki.com/Basic_Security_Fixes)
* [Wikipedia: Fork bomb Prevention](https://en.wikipedia.org/wiki/Fork_bomb#Prevention)

### Other docs

* [Linux Standard Base Core Specification 4.1](http://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/book1.html)
  * [Chapter 21. Users & Groups](http://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/usernames.html)
* [Filesystem Hierarchy Standard 2.3](http://refspecs.linuxfoundation.org/FHS_2.3/fhs-2.3.html)
* <https://iase.disa.mil/stigs/os/unix-linux/Pages/index.aspx>
* [PAM Mastery book](https://www.tiltedwindmillpress.com/?product=pam) by [Michael W Lucas](https://www.michaelwlucas.com/)
* [The Linux-PAM System Administrators' Guide](http://linux-pam.org/Linux-PAM-html/Linux-PAM_SAG.html)
* [Sudo Mastery, 2nd Edition](https://www.tiltedwindmillpress.com/product/sudo-mastery-2nd-edition/)
* [Linux Firewalls](https://nostarch.com/firewalls.htm)
* [Secure Secure Shell](https://stribika.github.io/2015/01/04/secure-secure-shell.html)
* [Securing Debian Manual](https://www.debian.org/doc/manuals/securing-debian-manual/index.en.html)

[1]: http://benchmarks.cisecurity.org/downloads/browse/index.cfm?category=benchmarks.os.linux.slackware
[2]: http://dentonj.freeshell.org/system-hardening-10.2.txt
