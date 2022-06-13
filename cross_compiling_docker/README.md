# Running arm64v8 docker image on x86_64 machine
 
As described in [multiarch/qemu-user-static](https://github.com/multiarch/qemu-user-static), it is possible running **arm64v8** docker image on top of **x86_64** or **amd64** host by leverage [bitfmt-misc](https://www.kernel.org/doc/html/v4.17/admin-guide/binfmt-misc.html) and [QEMU](https://www.qemu.org/).

## QEMU

QEMU allows us running arm64v8 binary on x86_64 machine. It translates arm64v8 machine language into x86_64 machine language. QEMU user static is a static linked program running in user space.

Taking following code as example.

```
#include<stdio.h>
int main(int argc, char *argv[])
{
    int i;
    printf("Total Number of args: %d\n",argc);
    for(i=0;i<argc;i++)
    {
        printf("%dth argv: %s\n",i, argv[i]);
    }
    return 0;
}
```

Following is the result of compiling and ruuning the code on arm64v8 docker:

```
# gcc --static example.c 
# ./a.out 1 2 3
Total Number of args: 4
0th argv: ./a.out
1th argv: 1
2th argv: 2
3th argv: 3
```

If we ruuning the binary(a.out) on x86_64 machine will receive error since x86_64 doesn't recoginize arm64v8 machine code.

```
$ ./a.out
-bash: ./a.out: cannot execute binary file: Exec format error
```
However, the binary can be executed via quem-static-user program. The program will translate arm64v8 machine code into x86_64 machine code.
```
$ qemu-aarch64-static a.out 1 2 3
Total Number of args: 4
0th argv: ./a.out
1th argv: 1
2th argv: 2
3th argv: 3
```

## binfmt

binfmt is a kernal module to allow us register an interprter, such as Java, Python, quem, .etc, with corresponding binary file. 

> binfmt_misc (Miscellaneous Binary Format) is a capability of the Linux kernel which allows arbitrary executable file formats to be recognized and passed to certain user space applications, such as emulators and virtual machines. It is one of a number of binary format handlers in the kernel that are involved in preparing a user-space program to run. [Wiki](https://en.wikipedia.org/wiki/Binfmt_misc)

> This Kernel feature allows you to invoke almost (for restrictions see below) every program by simply typing its name in the shell. This includes for example compiled Java(TM), Python or Emacs programs.
> To achieve this you must tell binfmt_misc which interpreter has to be invoked with which binary. Binfmt_misc recognises the binary-type by matching some bytes at the beginning of the file with a magic byte sequence (masking out specified bits) you have supplied. Binfmt_misc can also recognise a filename extension aka .com or .exe. [Kernel Doc](https://www.kernel.org/doc/html/latest/admin-guide/binfmt-misc.html)


Following is the result of running the arm64v8 binary on x86_64 machine with binfmt **enabled** and well **configured**.

```
$ ./a.out 1 2 3 # No need to specified qemu-aarch64-static binary
Total Number of args: 4
0th argv: ./a.out
1th argv: 1
2th argv: 2
3th argv: 3
```

### Enable/Disable binfmt

Please run following command as root user
```
$ grep binfmt /proc/mounts                                  # mounted?
$ mount binfmt_misc -t binfmt_misc /proc/sys/fs/binfmt_misc # mount it if not yet.
$ echo 1 > /proc/sys/fs/binfmt_misc/status                  # Enable binfmt
$ echo 0 > /proc/sys/fs/binfmt_misc/status                  # Disable binfmt
```

## How preserve argv[0] register in bitfmt may impact your binary?

However, we have seen an inconsistence behavior for executing a **arm64v8** binary on **x86_64** machine . For instance, running ```ccache ccahe -p``` will not throw error on arm64v8 docker image on AWS Graviton2 and x86_64 docker image on AWS C5 instance. But, it throws error while running **arm64v8** docker image on **x86_64** hosts. The error message is ```ccache: error: Recursive invocation (the name of the ccache binary must be "ccache")``` for ccache 3.7.7 and 3.7.12 vesion.

Following the docker file to to re-produce the error:
```
FROM arm64v8/ubuntu:20.04
RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y ccache
CMD ["ccache", "ccache", "-p"]
```

To dive deep into the issue, we install ccache 3.7.12 from [source code](https://ccache.dev/download.html) and add extra log to check out the __argv__ passed to the binary. Here is the result:

|Host|Docker Image    | argvs       |
|---|------- | ------------|
|x86_64|ubuntu:20.04  | ccache ccache -p|
|x86_64|arm64v8/ubuntu:20.04 | /usr/local/bin/ccache ccache -p|

For arm64v8 docker image, the first argvs has been converted to full path and it will trigger the fatal error.

Check the [bitfmt-misc](https://www.kernel.org/doc/html/v4.17/admin-guide/binfmt-misc.html), it states that **bitfmt** will convert the first argv to full path and it can be turned off via specified register.

> P - preserve-argv[0] Legacy behavior of binfmt_misc is to overwrite the original argv[0] with the full path to the binary. When this flag is included, binfmt_misc will add an argument to the argument vector for this purpose, thus preserving the original argv[0]. e.g. If your interp is set to /bin/foo and you run blah (which is in /usr/local/bin), then the kernel will execute /bin/foo with argv[] set to ["/bin/foo", "/usr/local/bin/blah", "blah"]. The interp has to be aware of this so it can execute /usr/local/bin/blah with argv[] set to ["blah"].


To enable preserved argv[0] in bitfmt-mise, we check the [qemu-binfmt-conf.sh](https://github.com/qemu/qemu/blob/master/scripts/qemu-binfmt-conf.sh) and found out that ```-g yes``` parameter can tell bitfmt-misc to preserve argv[0]. So, we append the paramether to  the command described in [multiarch/qemu-user-static](https://github.com/multiarch/qemu-user-static) and re-run it. Here is the command we run:
```
sudo docker run --rm --privileged multiarch/qemu-user-static --reset -p yes -g yes
```
The following command can check **binfmt** status:
```
cat /proc/sys/fs/binfmt_misc/qemu-aarch64
```

With ```-g yes``` parameter, the ```ccache ccache -p``` command will not throw fatal error.

If your binary will check **argv[0]**, please ensure enable **preserve argv[0]** for **binfmt** to make it work as expected.
