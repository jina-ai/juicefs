---
sidebar_label: POSIX Compatibility
sidebar_position: 6
slug: /posix_compatibility
---
# POSIX Compatibility

JuiceFS ensures POSIX compatibility with the help of pjdfstest and LTP.

## Pjdfstest

 [Pjdfstest](https://github.com/pjd/pjdfstest) is a test suite that helps exercise POSIX system calls. JuiceFS passed all of its latest 8813 tests:

```
All tests successful.

Test Summary Report
-------------------
/root/soft/pjdfstest/tests/chown/00.t          (Wstat: 0 Tests: 1323 Failed: 0)
  TODO passed:   693, 697, 708-709, 714-715, 729, 733
Files=235, Tests=8813, 233 wallclock secs ( 2.77 usr  0.38 sys +  2.57 cusr  3.93 csys =  9.65 CPU)
Result: PASS
```

Besides the things covered by pjdfstest, JuiceFS provides:

- Close-to-open consistency. Once a file is closed, the following open and read are guaranteed see the data written before close. Within same mount point, read can see all data written before it immediately.
- Rename and all other metadata operations are atomic guaranteed by transaction of metadata engines.
- Open files remain accessible after unlink from same mount point.
- Mmap is supported (tested with FSx).
- Fallocate with punch hole support.
- Extended attributes (xattr).
- BSD locks (flock).
- POSIX record locks (fcntl).

## LTP

[LTP](https://github.com/linux-test-project/ltp) (Linux Test Project) is a joint project developed and maintained by IBM, Cisco, Fujitsu and others.

> The project goal is to deliver tests to the open source community that validate the reliability, robustness, and stability of Linux.
>
> The LTP testsuite contains a collection of tools for testing the Linux kernel and related features. Our goal is to improve the Linux kernel and system libraries by bringing test automation to the testing effort.

JuiceFS passed most of its file system related tests.

### Test Environment

- Host: Amazon EC2: c5d.xlarge (4C 8G)
- OS: Ubuntu 20.04.1 LTS (Kernel 5.4.0-1029-aws)
- Object storage: Amazon S3
- JuiceFS version: 0.17-dev (2021-09-16 292f2b65)

### Test Steps

1. Download LTP [release](https://github.com/linux-test-project/ltp/releases/download/20210524/ltp-full-20210524.tar.bz2) from GitHub
2. Unarchive, compile and install:

```bash
$ tar -jvxf ltp-full-20210524.tar.bz2
$ cd ltp-full-20210524
$ ./configure
$ make all
$ make install
```

3. Change directory to `/opt/ltp` since test tools are installed here:

```bash
$ cd /opt/ltp
```

The test definition files are located under `runtest`. To speed up testing, we delete some pressure cases and unrelated cases in `fs` and `syscalls` (refer to [Appendix](#Appendix), modified files are saved as `fs-jfs` and `syscalls-jfs`), then execute:

```bash
$ ./runltp -d /mnt/jfs -f fs_bind,fs_perms_simple,fsx,io,smoketest,fs-jfs,syscalls-jfs
```

### Test Result

```bash
Testcase                                           Result     Exit Value
--------                                           ------     ----------
fcntl17                                            FAIL       7
fcntl17_64                                         FAIL       7
getxattr05                                         CONF       32
ioctl_loop05                                       FAIL       4
ioctl_ns07                                         FAIL       1
lseek11                                            CONF       32
open14                                             CONF       32
openat03                                           CONF       32
setxattr03                                         FAIL       6

-----------------------------------------------
Total Tests: 1270
Total Skipped Tests: 4
Total Failures: 5
Kernel Version: 5.4.0-1029-aws
Machine Architecture: x86_64
```

Reasons for the skipped and failed tests:

- fcntl17, fcntl17_64: automatically detect deadlock when trying to add POSIX locks. JuiceFS doesn't support it yet
- getxattr05: need ACL, which is not supported yet
- ioctl_loop05, ioctl_ns07, setxattr03: need `ioctl`, which is not supported yet
- lseek11: handle SEEK_DATA and SEEK_HOLE flags properly in `lseek`. JuiceFS uses kernel general function, which doesn't support these two flags
- open14, openat03: handle O_TMPFILE flag in `open`. JuiceFS can do nothing with it since it's not supported by FUSE

### Appendix

Deleted cases in `fs` and `syscalls`:

```bash
# fs --> fs-jfs
gf01 growfiles -W gf01 -b -e 1 -u -i 0 -L 20 -w -C 1 -l -I r -T 10 -f glseek20 -S 2 -d $TMPDIR
gf02 growfiles -W gf02 -b -e 1 -L 10 -i 100 -I p -S 2 -u -f gf03_ -d $TMPDIR
gf03 growfiles -W gf03 -b -e 1 -g 1 -i 1 -S 150 -u -f gf05_ -d $TMPDIR
gf04 growfiles -W gf04 -b -e 1 -g 4090 -i 500 -t 39000 -u -f gf06_ -d $TMPDIR
gf05 growfiles -W gf05 -b -e 1 -g 5000 -i 500 -t 49900 -T10 -c9 -I p -u -f gf07_ -d $TMPDIR
gf06 growfiles -W gf06 -b -e 1 -u -r 1-5000 -R 0--1 -i 0 -L 30 -C 1 -f g_rand10 -S 2 -d $TMPDIR
gf07 growfiles -W gf07 -b -e 1 -u -r 1-5000 -R 0--2 -i 0 -L 30 -C 1 -I p -f g_rand13 -S 2 -d $TMPDIR
gf08 growfiles -W gf08 -b -e 1 -u -r 1-5000 -R 0--2 -i 0 -L 30 -C 1 -f g_rand11 -S 2 -d $TMPDIR
gf09 growfiles -W gf09 -b -e 1 -u -r 1-5000 -R 0--1 -i 0 -L 30 -C 1 -I p -f g_rand12 -S 2 -d $TMPDIR
gf10 growfiles -W gf10 -b -e 1 -u -r 1-5000 -i 0 -L 30 -C 1 -I l -f g_lio14 -S 2 -d $TMPDIR
gf11 growfiles -W gf11 -b -e 1 -u -r 1-5000 -i 0 -L 30 -C 1 -I L -f g_lio15 -S 2 -d $TMPDIR
gf12 mkfifo $TMPDIR/gffifo17; growfiles -b -W gf12 -e 1 -u -i 0 -L 30 $TMPDIR/gffifo17
gf13 mkfifo $TMPDIR/gffifo18; growfiles -b -W gf13 -e 1 -u -i 0 -L 30 -I r -r 1-4096 $TMPDIR/gffifo18
gf14 growfiles -W gf14 -b -e 1 -u -i 0 -L 20 -w -l -C 1 -T 10 -f glseek19 -S 2 -d $TMPDIR
gf15 growfiles -W gf15 -b -e 1 -u -r 1-49600 -I r -u -i 0 -L 120 -f Lgfile1 -d $TMPDIR
gf16 growfiles -W gf16 -b -e 1 -i 0 -L 120 -u -g 4090 -T 101 -t 408990 -l -C 10 -c 1000 -S 10 -f Lgf02_ -d $TMPDIR
gf17 growfiles -W gf17 -b -e 1 -i 0 -L 120 -u -g 5000 -T 101 -t 499990 -l -C 10 -c 1000 -S 10 -f Lgf03_ -d $TMPDIR
gf18 growfiles -W gf18 -b -e 1 -i 0 -L 120 -w -u -r 10-5000 -I r -l -S 2 -f Lgf04_ -d $TMPDIR
gf19 growfiles -W gf19 -b -e 1 -g 5000 -i 500 -t 49900 -T10 -c9 -I p -o O_RDWR,O_CREAT,O_TRUNC -u -f gf08i_ -d $TMPDIR
gf20 growfiles -W gf20 -D 0 -b -i 0 -L 60 -u -B 1000b -e 1 -r 1-256000:512 -R 512-256000 -T 4 -f gfbigio-$$ -d $TMPDIR
gf21 growfiles -W gf21 -D 0 -b -i 0 -L 60 -u -B 1000b -e 1 -g 20480 -T 10 -t 20480 -f gf-bld-$$ -d $TMPDIR
gf22 growfiles -W gf22 -D 0 -b -i 0 -L 60 -u -B 1000b -e 1 -g 20480 -T 10 -t 20480 -f gf-bldf-$$ -d $TMPDIR
gf23 growfiles -W gf23 -D 0 -b -i 0 -L 60 -u -B 1000b -e 1 -r 512-64000:1024 -R 1-384000 -T 4 -f gf-inf-$$ -d $TMPDIR
gf24 growfiles -W gf24 -D 0 -b -i 0 -L 60 -u -B 1000b -e 1 -g 20480 -f gf-jbld-$$ -d $TMPDIR
gf25 growfiles -W gf25 -D 0 -b -i 0 -L 60 -u -B 1000b -e 1 -r 1024000-2048000:2048 -R 4095-2048000 -T 1 -f gf-large-gs-$$ -d $TMPDIR
gf26 growfiles -W gf26 -D 0 -b -i 0 -L 60 -u -B 1000b -e 1 -r 128-32768:128 -R 512-64000 -T 4 -f gfsmallio-$$ -d $TMPDIR
gf27 growfiles -W gf27 -b -D 0 -w -g 8b -C 1 -b -i 1000 -u -f gfsparse-1-$$ -d $TMPDIR
gf28 growfiles -W gf28 -b -D 0 -w -g 16b -C 1 -b -i 1000 -u -f gfsparse-2-$$ -d $TMPDIR
gf29 growfiles -W gf29 -b -D 0 -r 1-4096 -R 0-33554432 -i 0 -L 60 -C 1 -u -f gfsparse-3-$$ -d $TMPDIR
gf30 growfiles -W gf30 -D 0 -b -i 0 -L 60 -u -B 1000b -e 1 -o O_RDWR,O_CREAT,O_SYNC -g 20480 -T 10 -t 20480 -f gf-sync-$$ -d $TMPDIR
rwtest01 export LTPROOT; rwtest -N rwtest01 -c -q -i 60s  -f sync 10%25000:$TMPDIR/rw-sync-$$
rwtest02 export LTPROOT; rwtest -N rwtest02 -c -q -i 60s  -f buffered 10%25000:$TMPDIR/rw-buffered-$$
rwtest03 export LTPROOT; rwtest -N rwtest03 -c -q -i 60s -n 2  -f buffered -s mmread,mmwrite -m random -Dv 10%25000:$TMPDIR/mm-buff-$$
rwtest04 export LTPROOT; rwtest -N rwtest04 -c -q -i 60s -n 2  -f sync -s mmread,mmwrite -m random -Dv 10%25000:$TMPDIR/mm-sync-$$
rwtest05 export LTPROOT; rwtest -N rwtest05 -c -q -i 50 -T 64b 500b:$TMPDIR/rwtest01%f
iogen01 export LTPROOT; rwtest -N iogen01 -i 120s -s read,write -Da -Dv -n 2 500b:$TMPDIR/doio.f1.$$ 1000b:$TMPDIR/doio.f2.$$
quota_remount_test01 quota_remount_test01.sh
isofs isofs.sh

# syscalls --> syscalls-jfs
bpf_prog05 bpf_prog05
cacheflush01 cacheflush01
chown01_16 chown01_16
chown02_16 chown02_16
chown03_16 chown03_16
chown04_16 chown04_16
chown05_16 chown05_16
clock_nanosleep03 clock_nanosleep03
clock_gettime03 clock_gettime03
leapsec01 leapsec01
close_range01 close_range01
close_range02 close_range02
fallocate06 fallocate06
fchown01_16 fchown01_16
fchown02_16 fchown02_16
fchown03_16 fchown03_16
fchown04_16 fchown04_16
fchown05_16 fchown05_16
fcntl06 fcntl06
fcntl06_64 fcntl06_64
getegid01_16 getegid01_16
getegid02_16 getegid02_16
geteuid01_16 geteuid01_16
geteuid02_16 geteuid02_16
getgid01_16 getgid01_16
getgid03_16 getgid03_16
getgroups01_16 getgroups01_16
getgroups03_16 getgroups03_16
getresgid01_16 getresgid01_16
getresgid02_16 getresgid02_16
getresgid03_16 getresgid03_16
getresuid01_16 getresuid01_16
getresuid02_16 getresuid02_16
getresuid03_16 getresuid03_16
getrusage04 getrusage04
getuid01_16 getuid01_16
getuid03_16 getuid03_16
ioctl_sg01 ioctl_sg01
fanotify16 fanotify16
fanotify18 fanotify18
fanotify19 fanotify19
lchown01_16 lchown01_16
lchown02_16 lchown02_16
lchown03_16 lchown03_16
mbind02 mbind02
mbind03 mbind03
mbind04 mbind04
migrate_pages02 migrate_pages02
migrate_pages03 migrate_pages03
modify_ldt01 modify_ldt01
modify_ldt02 modify_ldt02
modify_ldt03 modify_ldt03
move_pages01 move_pages01
move_pages02 move_pages02
move_pages03 move_pages03
move_pages04 move_pages04
move_pages05 move_pages05
move_pages06 move_pages06
move_pages07 move_pages07
move_pages09 move_pages09
move_pages10 move_pages10
move_pages11 move_pages11
move_pages12 move_pages12
msgctl05 msgctl05
msgstress04 msgstress04
openat201 openat201
openat202 openat202
openat203 openat203
madvise06 madvise06
madvise09 madvise09
ptrace04 ptrace04
quotactl01 quotactl01
quotactl04 quotactl04
quotactl06 quotactl06
readdir21 readdir21
recvmsg03 recvmsg03
sbrk03 sbrk03
semctl08 semctl08
semctl09 semctl09
set_mempolicy01 set_mempolicy01
set_mempolicy02 set_mempolicy02
set_mempolicy03 set_mempolicy03
set_mempolicy04 set_mempolicy04
set_thread_area01 set_thread_area01
setfsgid01_16 setfsgid01_16
setfsgid02_16 setfsgid02_16
setfsgid03_16 setfsgid03_16
setfsuid01_16 setfsuid01_16
setfsuid02_16 setfsuid02_16
setfsuid03_16 setfsuid03_16
setfsuid04_16 setfsuid04_16
setgid01_16 setgid01_16
setgid02_16 setgid02_16
setgid03_16 setgid03_16
sgetmask01 sgetmask01
setgroups01_16 setgroups01_16
setgroups02_16 setgroups02_16
setgroups03_16 setgroups03_16
setgroups04_16 setgroups04_16
setregid01_16 setregid01_16
setregid02_16 setregid02_16
setregid03_16 setregid03_16
setregid04_16 setregid04_16
setresgid01_16 setresgid01_16
setresgid02_16 setresgid02_16
setresgid03_16 setresgid03_16
setresgid04_16 setresgid04_16
setresuid01_16 setresuid01_16
setresuid02_16 setresuid02_16
setresuid03_16 setresuid03_16
setresuid04_16 setresuid04_16
setresuid05_16 setresuid05_16
setreuid01_16 setreuid01_16
setreuid02_16 setreuid02_16
setreuid03_16 setreuid03_16
setreuid04_16 setreuid04_16
setreuid05_16 setreuid05_16
setreuid06_16 setreuid06_16
setreuid07_16 setreuid07_16
setuid01_16 setuid01_16
setuid03_16 setuid03_16
setuid04_16 setuid04_16
shmctl06 shmctl06
socketcall01 socketcall01
socketcall02 socketcall02
socketcall03 socketcall03
ssetmask01 ssetmask01
swapoff01 swapoff01
swapoff02 swapoff02
swapon01 swapon01
swapon02 swapon02
swapon03 swapon03
switch01 endian_switch01
sysinfo03 sysinfo03
timerfd04 timerfd04
perf_event_open02 perf_event_open02
statx07 statx07
io_uring02 io_uring02
```
