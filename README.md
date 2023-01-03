This Ansible project does nothing really fancy but showcases a severe
performance degradation I was experiencing when upgrading Ansible Core
on my workstation (which is the Ansible controller in this case).

My workstation is on Fedora Linux release 37 which originally shipped with

- [ansible-core-2.13.4-1.fc37.noarch.rpm](https://download-ib01.fedoraproject.org/pub/fedora/linux/releases/37/Everything/x86_64/os/Packages/a/ansible-core-2.13.4-1.fc37.noarch.rpm)

After upgrding the `ansible-core` package to it's latests releases

- [ansible-core-2.14.1-1.fc37.noarch.rpm](https://download-ib01.fedoraproject.org/pub/fedora/linux/updates/37/Everything/x86_64/Packages/a/ansible-core-2.14.1-1.fc37.noarch.rpm)

operations which were pretty quick before now take magniutes longer. The issue
can be pinpointed to this package upgrade/downgrade with no other changes done
to the system. Just think of ...

```
$ sudo rpm -Uvh ansible-core-2.14.1-1.fc37.noarch.rpm

$ sudo rpm -Uvh --oldpackage ansible-core-2.13.4-1.fc37.noarch.rpm
```

Not even talking about `ansible-playbook` runs, it's just simple
operations like listing the Ansible inventory which take ages with
Ansible Core 2.14.1 compared to 2.13.4 (we are talking about minutes vs.
a few seconds). Of course this affects all sort of Ansible operations as
parsing and constructing the inventory is an essential part of it.

After some trail-and-error I was able to narrow down to reasons for
the bad performance to the use of the `ansible.builtin.constructed`
inventory plugin. I use this plugin to construct some host groups
based on hostnames as defined in the `hosts.yaml` inventory file
(`inventory_hostname_short`) as well as host-specific variables which
are also defined there. There is no reference to any `host_vars` or
`group_vars`.

In the `hosts.constructed.yaml` (which is the inventory file utilizing
the `ansible.builtin.constructed` inventory plugin) is use several
methods to construct those host groups, and to my surprise it's not
the methods which I would have considered to be compute-heavy by
their nature (regular expression matching or using some IP address
validation) but simple string compares, like `foo in ['bar']` (I also
tried alternative forms like `foo == 'bar'` which would work if there
is just one element, it doesn't really matter). Evaluation just done by
selectively commenting out certain type of methods.

By doing some statistics on the number of system calls done (by using
`strace(8)`) I found out that the younger version of Ansible Core does
massively more `newfstatat` system calls which are spend by scanning
the filter plugin directory of Ansible Core installation over and over
again.

In order to eleminate any side-effects by the RPM packages
provided by the Fedora project I did a [Git clone
installation](https://docs.ansible.com/ansible/latest/installation_guide
/intro_installation.html#running-the-devel-branch-from-a-clone), running
Ansible from the `devel` branch. The same bad performance can be seen
here.

By picking several commits on the timeline of the last 6 months it
looks like there wasn't just one single commit which decreased the
performance. It rather declined gradually, but neverthelss I was able to
identify two commits by using `git bisect` which had massive impact on
the runtime duration.

- https://github.com/ansible/ansible/commit/4260b71cc77b7a44e061668d0d408d847f550156
- https://github.com/ansible/ansible/commit/3b937123d26473b7005674b8915423e3f38f6105

Therefore I will focus on those commits and their parents below.

In order to make this issue reproducable, I created this *synthetic*
use case which comes pretty close to my real use case (which I cannot
publish due to data privacy concerns). I just created a minor subset
of 38 hosts which is already enough critical mass to clearly expose
the issue without making it really painful as it is with hosts in the
hundreds.

I also vendored in the `ansible.utils` collection to make this test
really self-contained. It's only used by some 'is ansible.utils.ipv4`
checks, commenting them out drops the dependency on this collection.

Here are some messurements with the various versions of Ansible Core.
I order to make the output more readble, I redirected `stdout` to
`/dev/null`. As `strace(8)` also comes with some overhead, I'm executing
the command with and without `strace`.

### Ansible Core 2.13.4 from Fedora 37

```
$ ansible --version
ansible [core 2.13.4]
  config file = /home/cschug/wrk/ansible_issue/ansible.cfg
  configured module search path = ['/home/cschug/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.11/site-packages/ansible
  ansible collection location = /home/cschug/wrk/ansible_issue/collections
  executable location = /usr/bin/ansible
  python version = 3.11.1 (main, Dec  7 2022, 00:00:00) [GCC 12.2.1 20221121 (Red Hat 12.2.1-4)]
  jinja version = 3.0.3
  libyaml = True

$ time ansible all --list-hosts >/dev/null

real    0m0.869s
user    0m0.813s
sys     0m0.052s

$ time strace -c -e newfstatat ansible all --list-hosts >/dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.005415           1      4612       346 newfstatat
------ ----------- ----------- --------- --------- ----------------
100.00    0.005415           1      4612       346 total

real    0m1.331s
user    0m0.991s
sys     0m0.316s
```

### Ansible Core 2.14.1 from Fedora 37

```
$ ansible --version
ansible [core 2.14.1]
  config file = /home/cschug/wrk/ansible_issue/ansible.cfg
  configured module search path = ['/home/cschug/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.11/site-packages/ansible
  ansible collection location = /home/cschug/wrk/ansible_issue/collections
  executable location = /usr/bin/ansible
  python version = 3.11.1 (main, Dec  7 2022, 00:00:00) [GCC 12.2.1 20221121 (Red Hat 12.2.1-4)] (/usr/bin/python3)
  jinja version = 3.0.3
  libyaml = True

$ time ansible all --list-hosts >/dev/null

real    0m5.402s
user    0m4.605s
sys     0m0.773s

$ time strace -c -e newfstatat ansible all --list-hosts >/dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.712552           1    518664       349 newfstatat
------ ----------- ----------- --------- --------- ----------------
100.00    0.712552           1    518664       349 total

real    0m16.339s
user    0m8.314s
sys     0m7.369s
```

### Ansible Core `devel` branch at commit `2464e1e91c` (Wed Aug 31 12:15:58 2022 -0400)

Still performing rather well, number of `newfstatat` system calls only a
slightly bit higher compared to 2.13.4:

```
$ ansible --version
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.
ansible [core 2.14.0.dev0] (detached HEAD 2464e1e91c) last updated 2023/01/03 17:35:42 (GMT +200)
  config file = /home/cschug/wrk/ansible_issue/ansible.cfg
  configured module search path = ['/home/cschug/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/cschug/ansible/lib/ansible
  ansible collection location = /home/cschug/wrk/ansible_issue/collections
  executable location = /home/cschug/ansible/bin/ansible
  python version = 3.11.1 (main, Dec  7 2022, 00:00:00) [GCC 12.2.1 20221121 (Red Hat 12.2.1-4)] (/usr/bin/python)
  jinja version = 3.0.3
  libyaml = True

$ time ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.

real    0m0.998s
user    0m0.935s
sys     0m0.059s

$ time strace -c -e newfstatat ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.004906           0      5001       355 newfstatat
------ ----------- ----------- --------- --------- ----------------
100.00    0.004906           0      5001       355 total

real    0m1.519s
user    0m1.158s
sys     0m0.336s
```

### Ansible Core `devel` branch at commit `4260b71cc7` (Thu Sep 1 14:16:05 2022 -0400)

First drastic jump in the number of `newfstatat` calls also resulting on
much longer runtime.
```
$ ansible --version
[WARNING]: You are running the development version of Ansible. You should only run Ansible from "devel" if you are modifying
the Ansible engine, or trying out features under development. This is a rapidly changing source of code and can become unstable
at any point.
ansible [core 2.14.0.dev0] (detached HEAD 4260b71cc7) last updated 2023/01/03 17:38:59 (GMT +200)
  config file = /home/cschug/wrk/ansible_issue/ansible.cfg
  configured module search path = ['/home/cschug/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/cschug/ansible/lib/ansible
  ansible collection location = /home/cschug/wrk/ansible_issue/collections
  executable location = /home/cschug/ansible/bin/ansible
  python version = 3.11.1 (main, Dec  7 2022, 00:00:00) [GCC 12.2.1 20221121 (Red Hat 12.2.1-4)] (/usr/bin/python)
  jinja version = 3.0.3
  libyaml = True

$ time ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.

real    0m3.587s
user    0m3.195s
sys     0m0.378s

$ time strace -c -e newfstatat ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.208801           1    195381       355 newfstatat
------ ----------- ----------- --------- --------- ----------------
100.00    0.208801           1    195381       355 total

real    0m9.611s
user    0m5.551s
sys     0m3.794s
```

### Ansible Core `devel` branch at commit `24a42bb25e` (Tue Sep 20 23:04:24 2022 +0900)

Over several commits the situation got worse again with being able to
identify a specific one:
```
$ ansible --version
[WARNING]: You are running the development version of Ansible. You should only run Ansible from "devel" if you are modifying
the Ansible engine, or trying out features under development. This is a rapidly changing source of code and can become unstable
at any point.
ansible [core 2.14.0.dev0] (detached HEAD 24a42bb25e) last updated 2023/01/03 17:41:59 (GMT +200)
  config file = /home/cschug/wrk/ansible_issue/ansible.cfg
  configured module search path = ['/home/cschug/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/cschug/ansible/lib/ansible
  ansible collection location = /home/cschug/wrk/ansible_issue/collections
  executable location = /home/cschug/ansible/bin/ansible
  python version = 3.11.1 (main, Dec  7 2022, 00:00:00) [GCC 12.2.1 20221121 (Red Hat 12.2.1-4)] (/usr/bin/python)
  jinja version = 3.0.3
  libyaml = True

$ time ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.

real    0m4.616s
user    0m4.126s
sys     0m0.467s

$ time strace -c -e newfstatat ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.233879           1    233466       355 newfstatat
------ ----------- ----------- --------- --------- ----------------
100.00    0.233879           1    233466       355 total

real    0m12.086s
user    0m7.264s
sys     0m4.522s
```

### Ansible Core `devel` branch at commit `3b937123d2` (Tue Sep 20 11:38:26 2022 -0400)

Here is another big one which more than doubled the number of
`newfstatat` system calls:

```
$ ansible --version
[WARNING]: You are running the development version of Ansible. You should only run Ansible from "devel" if you are modifying
the Ansible engine, or trying out features under development. This is a rapidly changing source of code and can become unstable
at any point.
ansible [core 2.14.0.dev0] (detached HEAD 3b937123d2) last updated 2023/01/03 17:49:46 (GMT +200)
  config file = /home/cschug/wrk/ansible_issue/ansible.cfg
  configured module search path = ['/home/cschug/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/cschug/ansible/lib/ansible
  ansible collection location = /home/cschug/wrk/ansible_issue/collections
  executable location = /home/cschug/ansible/bin/ansible
  python version = 3.11.1 (main, Dec  7 2022, 00:00:00) [GCC 12.2.1 20221121 (Red Hat 12.2.1-4)] (/usr/bin/python)
  jinja version = 3.0.3
  libyaml = True

$ time ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.

real    0m5.501s
user    0m4.680s
sys     0m0.799s

$ time strace -c -e newfstatat ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.696195           1    493652       355 newfstatat
------ ----------- ----------- --------- --------- ----------------
100.00    0.696195           1    493652       355 total

real    0m16.770s
user    0m8.558s
sys     0m7.631s
```

### Ansible Core `devel` branch at commit `38cedc7f1a` (Tue Jan 3 16:48:06 2023 +0100)

The `devel` branch as of today:

```
$ ansible --version
[WARNING]: You are running the development version of Ansible. You should only run Ansible from "devel" if you are modifying
the Ansible engine, or trying out features under development. This is a rapidly changing source of code and can become unstable
at any point.
ansible [core 2.15.0.dev0] (detached HEAD 38cedc7f1a) last updated 2023/01/03 17:52:58 (GMT +200)
  config file = /home/cschug/wrk/ansible_issue/ansible.cfg
  configured module search path = ['/home/cschug/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/cschug/ansible/lib/ansible
  ansible collection location = /home/cschug/wrk/ansible_issue/collections
  executable location = /home/cschug/ansible/bin/ansible
  python version = 3.11.1 (main, Dec  7 2022, 00:00:00) [GCC 12.2.1 20221121 (Red Hat 12.2.1-4)] (/usr/bin/python)
  jinja version = 3.0.3
  libyaml = True

$ time ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.

real    0m5.682s
user    0m4.824s
sys     0m0.835s

$ time strace -c -e newfstatat ansible all --list-hosts >/dev/null
[WARNING]: You are running the development version of Ansible. You should only
run Ansible from "devel" if you are modifying the Ansible engine, or trying out
features under development. This is a rapidly changing source of code and can
become unstable at any point.
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.720408           1    525382       355 newfstatat
------ ----------- ----------- --------- --------- ----------------
100.00    0.720408           1    525382       355 total

real    0m15.938s
user    0m8.029s
sys     0m7.262s
```
