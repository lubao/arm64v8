[Boosts Library](https://www.boost.org/) is using quadmath from standard C library for **float128** calculation. However, standard C won't build **libquadmath on aarch64** architecture since aarch64 support **long double** data type natively.

[GCC Bug Report](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=96016) contains detail information.

This repo contains a patch to enable libquadmath on GCC 9.4.0 version ONLY.

Sample Build Command
```
sudo docker build -t libquadmath --cpuset-cpus 1-9 -o - .  > build.out.tar
```
