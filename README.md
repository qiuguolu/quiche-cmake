quiche built with CMake

commit: `71cbf0d`  2025-02-17

**build**

```shell
apt-get install git cmake automake libtool libicu-dev

cd quiche-cmake
git submodule update --init --recursive  
mkdir build && cd build
cmake ../ && make
```