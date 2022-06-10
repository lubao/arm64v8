# How binfmt perserve argv will impact your binary?
 
As described in [multiarch/qemu-user-static](https://github.com/multiarch/qemu-user-static), it is possible running **arm64v8** docker image on top of **x86_64** or **amd64** host by leverage [bitfmt-misc](https://www.kernel.org/doc/html/v4.17/admin-guide/binfmt-misc.html) and [QEMU USER Static](https://github.com/multiarch/qemu-user-static).

However, we have seen an inconsistence behavior for executing a **arm64v8** binary on **x86_64** machine . For instance, running ```ccache ccahe -p``` is working properly on arm64v8 docker image on AWS Graviton2 and x86_64 docker image on AWS C5 instance. But, it drows error while running **arm64v8** docker image on **x86_64** hosts. The error message is ```ccache: error: Recursive invocation (the name of the ccache binary must be "ccache")``` for ccache 3.7.7 and 3.7.12 vesion.

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

To enable preserved argv[0] in bitfmt-mise, we check the [qemu-binfmt-conf.sh](https://github.com/qemu/qemu/blob/master/scripts/qemu-binfmt-conf.sh) and found out that ```-g yes``` parameter can tell bitfmt-misc to preserve argv[0]. So, we append the paramether to  the command described in [multiarch/qemu-user-static](https://github.com/multiarch/qemu-user-static) and re-run it. Here is the command we run:
```
sudo docker run --rm --privileged multiarch/qemu-user-static --reset -p yes -g yes
```
The following command can check **binfmt** status:
```
cat /proc/sys/fs/binfmt_misc/qemu-aarch64
```

With ```-g yes``` parameter, the ```ccache ccache -p``` command can work.

If your binary will check **argv[0]**, please ensure enable **preserve argv[0]** for **binfmt** to make it work as expected.