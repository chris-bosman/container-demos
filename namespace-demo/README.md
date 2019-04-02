# Container Functionality Demo #

First, I'll acknowledge that this demo is sourced entirely from [this](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/) blog post by Julia Evans. This demo merely condenses and paraphrases that blog post into a few commands to practically demonstrate the core concepts behind containerization.

## Namespaces ##

In Linux, the concept of namespaces is used to separate processes and functions from one another. There are many different kinds, but let's just touch on a couple:

* PID Namespace: For separating running processes by namespaces.
* Networking Namespace: For separating networking calls.
* Mount Namespace: For separating filesystem mounts.

Docker, essentially, combines the usage of these different namespace types to create individual namespaces for a container for each of these types, so that the container runs essentially separately from the root operating system. If you have a Linux machine accessible, you can run the following to see this in action.

First, for a sanity check, check what's running on your Linux box in the standard PID namespace:

```console
$ ps aux
USER               PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
cbosman            334   6.6  1.3  6481412 221308   ??  S    Tue10AM  75:19.19 /Applications/Vi
cbosman            453   4.0  1.2  5745348 207904   ??  S    Tue10AM  33:20.99 /Applications/Vi
cbosman           1277   3.8 41.8 13701352 7013952   ??  S    Tue10AM  97:08.31 /Applications/Pa
cbosman            640   2.4  0.1  6820620  21836   ??  S    Tue10AM  63:51.30 com.docker.hyper
_windowserver      213   2.2  0.2  6500520  30096   ??  Ss   Tue10AM  43:05.27 /System/Library/
cbosman          27040   2.2  0.2  5480404  28208   ??  S     2:35PM   5:06.26 /Applications/Vi
...
```

Now, we can create a new PID namespace running bash via the `unshare` command:
`sudo unshare --fork --pid --mount-proc bash`

And check what's running now:

```console
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  28372  4148 pts/6    S    23:01   0:00 bash
root         2  0.0  0.0  44432  3836 pts/6    R+   23:01   0:00 ps aux
```

Nice! We've separated out running processes into two different namespaces.

## CGroups ##

If we were just using Namespaces, you might be able to see right away where you'd run into some problems related to resource usage. Without limits on the resource usage of a namespace, you could very easily get a namespace, though separated, chewing up the entire host resource allotment. For this, there's CGroups. CGroups have the option to limit resource usage of any type, but let's just do memory.

`sudo cgcreate -a bork -g memory:testgroup`

And see what's in it:

```console
$ ls -l /sys/fs/cgroup/memory/testgroup`
-rw-r--r-- 1 bork root 0 Okt 10 23:16 memory.kmem.limit_in_bytes
-rw-r--r-- 1 bork root 0 Okt 10 23:14 memory.kmem.max_usage_in_bytes
```

We've got files that dictate the max usage of our cgroup in bytes, so let's give it a 10mb value to test against:

`sudo echo 10000000 >  /sys/fs/cgroup/memory/mycoolgroup/memory.kmem.limit_in_bytes`

And now we can use `cgexec` to jump into my cgroup:

`sudo cgexec -g memory:testgroup bash`

Go ahead and run commands and try things. Anything that requires less than 10mb of memory will succeed. Anything that requires more?

```console
$ sudo dummy command
error: Could not execute process `dummy command` (never executed)

Caused by:
  Cannot allocate memory (os error 12)
```

Features like this, in conjunction with `seccomp-bpf` (for limiting the system calls a process can run), `nsenter` (for entering various namespaces) are the Linux primitives that underpin Docker.