# TRON
TRON is a tool designed to improve the effectiveness of fuzzing the Linux kernel network stack. It synthesizes syscall–packet payloads by combining protocol structure with runtime feedback, enabling it to explore deeper and more complex execution paths within the network stack.

we will provide installation steps that how to build and use TRON for kernel fuzzing here.

## Build

### Install Prerequisites

Basic dependencies install (take for example on debain or ubuntu):
``` bash
sudo apt update
sudo apt install make gcc flex bison libncurses-dev libelf-dev libssl-dev
```
### GCC

If your distro's GCC is older, it's preferable to get the latest GCC from [this](https://gcc.gnu.org/) list. Download and unpack into `$GCC`, and you should have GCC binaries in `$GCC/bin/`

>**Ubuntu 20.04 LTS**: You can ignore this section. GCC is up-to-date.

``` bash
ls $GCC/bin/
# Sample output:
# cpp     gcc-ranlib  x86_64-pc-linux-gnu-gcc        x86_64-pc-linux-gnu-gcc-ranlib
# gcc     gcov        x86_64-pc-linux-gnu-gcc-9.0.0
# gcc-ar  gcov-dump   x86_64-pc-linux-gnu-gcc-ar
# gcc-nm  gcov-tool   x86_64-pc-linux-gnu-gcc-nm
```
### Install golang
We use golang in TRON, so make sure golang is installed before build TRON.

``` bash
wget https://dl.google.com/go/go1.22.4.linux-amd64.tar.gz
tar -xf go1.22.4.linux-amd64.tar.gz
mv go goroot
mkdir gopath
export GOPATH=`pwd`/gopath
export GOROOT=`pwd`/goroot
export PATH=$GOPATH/bin:$PATH
export PATH=$GOROOT/bin:$PATH
```
### Prepare Kernel
In here we use Linux Kernel(Enable Real time Config) v6.5 as an example.
First we need to have have a compilable Linux
```bash
# Download linux kernel 
git clone https://github.com/torvalds/linux
cd linux
export Kernel=$pwd
git checkout -f f66d6ac

After we have the Linux Kernel, we need to compile it.
``` bash
# Modified configuration
make defconfig  
make kvmconfig
vim .config
```

``` vim
# modified configuration
CONFIG_KCOV=y 
CONFIG_DEBUG_INFO=y 
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y 
CONFIG_CONFIGFS_FS=y
CONFIG_SECURITYFS=y
```

make it!
``` bash
make olddefconfig
make -j32
```

Now we should have vmlinux (kernel binary) and bzImage (packed kernel image):
```bash
$ ls $KERNEL/vmlinux
$KERNEL/vmlinux
$ ls $KERNEL/arch/x86/boot/bzImage
$KERNEL/arch/x86/boot/bzImage
```

### Prepare Image
```bash 
sudo apt-get install debootstrap 
export IMAGE=$pwd
cd $IMAGE/
wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
chmod +x create-image.sh
./create-image.sh
```
Now we have a image stretch.img and a public key.

### Compile TRON
``` bash 
make 
mkdir workdir
cd workdir
```
As the result compiled binaries should appear in the bin/ dir.

### Ready QEMU
Install QEMU:
``` bash
sudo apt-get install qemu-system-x86
```
Make sure the kernel boots and sshd starts:
``` bash 
qemu-system-x86_64 \
	-m 2G \
	-smp 2 \
	-kernel $KERNEL/arch/x86/boot/bzImage \
	-append "console=ttyS0 root=/dev/sda earlyprintk=serial net.ifnames=0" \
	-drive file=$IMAGE/stretch.img,format=raw \
	-net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
	-net nic,model=e1000 \
	-enable-kvm \
	-nographic \
	-pidfile vm.pid \
	2>&1 | tee vm.log
```
see if ssh works
``` bash 
ssh -i $IMAGE/stretch.id_rsa -p 10021 ``-o "StrictHostKeyChecking no" 
```

To kill the running QEMU instance press Ctrl+A and then X or run:
``` bash
kill $(cat vm.pid)
```
If QEMU works, the kernel boots and ssh succeeds, we can shutdown QEMU and try to run __TRON__.

Now we can start to prepare a __config.json__ file.
``` json 
{
        "target": "linux/amd64",
        "http": "127.0.0.1:56295",
        "workdir": "./workdir",
        "cover": false,
        "kernel_obj": "$(Kernel)/vmlinux",
        "image": "$(image)/stretch.img",
        "sshkey": "$(image)/stretch.id_rsa",
        "syzkaller": "$pwd",
        "procs": 2,
        "type": "qemu",
        "vm": {
                "count": 2,
                "kernel": "$(Kernel)/bzImage",
                "cpu": 2,
                "mem": 4096
        }

```
Now we can run it.

``` bash
./bin/syz-manager -config config.json
```
The `syz-manager` process will wind up VMs and start fuzzing in them.
Found crashes, statistics and other information is exposed on the HTTP address specified in the manager config.

##  Experiment Results

### Host Machine System Configuration
```bash
  CPU: 128 core
  Memory: 32GB
  Ubuntu 22.04.4 LTS jammy 
```

### Virtal Machine Resource Configration
``` bash
  2 core CPU + 2GB Memory
```

### Test targeted Linux Version

We chose Linux kernel v6.12, v6.13, v6.14, and v6.15 as our test kernel targets. In detail, the Linux v6.15 is the latest release version when we were conducting experiments. Each version of the kernel uses the same compilation configuration, while KCOV and KASAN options are enabled in order to collect code coverage and detect memory errors. When setting up the KCSAN configuration, the same configuration is used in the control test. 


### Coverage over time 

10 VM (2vCPU and 2G RAM) average for 24 hours.

![image](https://github.com/rushwow/TRON/blob/main/assets/cov.png)  

### New Bugs Reported:


| Index 	| Version 	| Module 	| Bug Location 	| Bug Type 	| Bug Description 	|
|:---:	|:---:	|---	|---	|---	|---	|
| 1 	| 6.12 	| net/netlabel/netlabelkapi.c 	| netlblconnsetattr() 	| null-ptr-deref 	| uninitialized socket option leads to null pointer dereference 	|
| 2 	| 6.12 	| net/ipv6/calipso.c 	| calipsosocksetattr() 	| null-ptr-deref 	| null socket option dereference triggers IPv6 label handling failure 	|
| 3 	| 6.12 	| net/core/neighbour.c 	| neightimerhandler() 	| logic error 	| improper neighbor timer handling causes logic error 	|
| 4 	| 6.13 	| net/core/rtnetlink.c 	| rtnlnewlink() 	| deadlock 	| unsynchronized network configuration operations lead to deadlock 	|
| 5 	| 6.13 	| net/core/sock.c 	| skfree() 	| logic error 	| socket resource deallocation logic triggers error 	|
| 6 	| 6.14 	| net/netfilter/core.c 	| nfhookslow() 	| logic error 	| improper logic in nfhookslow leads to packet misprocessing 	|
| 7 	| 6.14 	| net/wireless/core.c 	| cfg80211devfree() 	| use-after-free 	| dangling pointer access in cfg80211devfree triggers memory error 	|
| 8 	| 6.14 	| net/mac80211/tx.c 	| ieee80211beaconupdatecntdwn() 	| logic error 	| invalid beacon countdown state triggers logic error 	|
| 9 	| 6.14 	| net/ipv4/ipmr.c 	| ipmrrulesexit() 	| use-after-free 	| improper multicast table cleanup triggers use-after-free error 	|
| 10 	| 6.14 	| net/wireless/nl80211.c 	| nl80211setwiphy() 	| use-after-free 	| invalid pointer access during wiphy netns transition 	|
| 11 	| 6.14 	| /net/ipv4/tcpipv4.c 	| tcpv4rcv() 	| deadlock 	| synchronized TCP processing causes deadlock 	|
| 12 	| 6.14 	| net/ipv4/tcpinput.c 	| tcpconnrequest() 	| null-ptr-deref 	| null socket reference in TCP connection request triggers memory error 	|
| 13 	| 6.14 	| /drivers/net/ethernet/intel/e1000/e1000main.c 	| e1000close() 	| deadlock 	| improper lock ordering causes deadlock 	|
| 14 	| 6.14 	| net/ipv4/ipmr.c 	| ipmrnetexitbatch() 	| logic error 	| improper multicast routing cleanup triggers logic error 	|
| 15 	| 6.14 	| net/ipv4/udp.c 	| udpsendmsg() 	| logic error 	| improper skb allocation during UDP send leads to logic error 	|
| 16 	| 6.15 	| drivers/net/ethernet/intel/e1000/e1000hw.c 	| e1000readphyreg() 	| deadlock 	| synchronized net device operation leads to deadlock 	|
| 17 	| 6.15 	| net/mac80211/scan.c 	| ieee80211informbss() 	| use-after-free 	| invalid reuse of freed BSS entry triggers memory error 	|
| 18 	| 6.15 	| net/unix/afunix.c 	| unixdgramsendmsg() 	| logic error 	| improper socket message routing triggers logic error 	|
| 19 	| 6.15 	| net/ipv4/igmp.c 	| igmpmcinit() 	| logic error 	| protocol initialization inconsistency induces logic error 	|
| 20 	| 6.15 	| net/ipv6/afinet6.c 	| inet6release() 	| logic error 	| improper socket teardown logic causes logic error 	|
| 21 	| 6.15 	| net/ipv6/ip6input.c 	| ip6mcinput() 	| use-after-free 	| invalid buffer access leads to use-after-free in ip6mcinput 	|
| 22 	| 6.15 	| net/mac80211/rx.c 	| ieee80211rxnapi() 	| logic error 	| packet handling in ieee80211rxnapi triggers logic error 	|
| 23 	| 6.15 	| net/unix/afunix.c 	| unixcreate1() 	| double free 	| duplicate deallocation in unixcreate1 triggers memory error 	|
| 24 	| 6.15 	| net/xfrm/xfrmuser.c 	| xfrmsendstatenotify() 	| logic error 	| inconsistent state notification logic triggers logic error 	|
| 25 	| 6.15 	| net/packet/afpacket.c 	| runfilter() 	| logic error 	| faulty packet filtering in net receive operation causes logic error 	|