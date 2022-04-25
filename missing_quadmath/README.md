[Boosts Library](https://www.boost.org/) is using quadmath from standard C library for **float128** calculation. However, standard C won't build **libquadmath on aarch64** architecture since aarch64 support **long double** data type natively.

[GCC Bug Report](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=96016) contains detail information.

This repo contains a patch to enable libquadmath on GCC 9.4.0 version ONLY.

Sample Build Command
```
sudo docker build -t libquadmath --cpuset-cpus=2 --memory=4g .
```
**NOTE**: Set up **--cpus** or **--cpuset-cpus**to limit max number of CPUs.

**NOTE**: Set up **--memory** to limit memory consummation.

**NOTE**: To prevent OOM error while running docker build, please keep ration of CPU:MEM to **1core:4g**

**NOTE**: Docker build may take a whole...

[Docker Doc](https://docs.docker.com/engine/reference/commandline/build/)


Sample Compile Command
```
gcc -o /opt/sample /opt/sample.cpp -lquadmath
```
**NOTE**: Appending **-lquadmath** at end of GCC command to link libquadmath
