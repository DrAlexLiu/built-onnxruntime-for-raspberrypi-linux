# built-onnxruntime-for-raspberrypi-linux
This is the whl file repo for x32 onnxruntime on Raspberry Pi

The whole repo is based on this repo:
https://github.com/nknytk/built-onnxruntime-for-raspberrypi-linux

Installaion:

Ref:https://github.com/nknytk/built-onnxruntime-for-raspberrypi-linux/blob/master/BUILD.md

1. prepare a RPi, 8GB recommended. 
2. Flast the 32bit system

Then

Install Kitware APT Repository for installing CMake>=3.26

https://apt.kitware.com/

For Bookworm RPi OS, follow ubuntu 22.04
For Bullesys RPi OS, follow ubuntu 20.04

There is script repo to install kitware on Bookworm RPi OS on RPi5
```bash
sudo apt-get update

sudo apt-get install ca-certificates gpg wget

test -f /usr/share/doc/kitware-archive-keyring/copyright || wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | gpg --dearmor - | sudo tee /usr/share/keyrings/kitware-archive-keyring.gpg >/dev/null

echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy main' | sudo tee /etc/apt/sources.list.d/kitware.list >/dev/null

sudo apt-get update

test -f /usr/share/doc/kitware-archive-keyring/copyright ||
sudo rm /usr/share/keyrings/kitware-archive-keyring.gpg

sudo apt-get install kitware-archive-keyring

echo 'deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ jammy-rc main' | sudo tee -a /etc/apt/sources.list.d/kitware.list >/dev/null

sudo apt-get update

sudo apt-get install cmake
```

Install necessary packages
```bash
sudo apt install git build-essential libcurl4-openssl-dev libssl-dev libatlas-base-dev 
sudo apt install tk-dev libncurses5-dev libncursesw5-dev libreadline6-dev libdb5.3-dev libgdbm-dev libsqlite3-dev libssl-dev libbz2-dev libexpat1-dev liblzma-dev zlib1g-dev libffi-dev libv4l-dev
```

(Optional) (Bookworm RPi OS) If you encounter some issues, lowering the gcc version may help. 
```bash
apt list --installed | grep gcc-12
sudo apt remove gcc-12 g++-12
sudo apt update
sudo apt install gcc-10 g++-10
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-10 100 --slave /usr/bin/g++ g++ /usr/bin/g++-10 --slave /usr/bin/gcov gcov /usr/bin/gcov-10
gcc --version
```
Download the desired Python package (Python 3.11.8 in this example):

```bash
wget https://www.python.org/ftp/python/3.11.8/Python-3.11.8.tgz
tar zxf Python-3.11.8.tgz
cd Python-3.11.8
./configure --enable-optimizations --with-lto --enable-shared --prefix=/opt/python3.8 LDFLAGS=-Wl,-rpath,/opt/python3.8/lib
make
sudo make altinstall
```

Install some python packages for onnxruntime building. Be aware of numpy version, if you choose a version of numpy, you will need at least that numpy version during pip install your onnxruntime whl file.
```bash
sudo /opt/python3.11/bin/pip3.11 install --upgrade pip wheel
sudo apt install libgfortran5 libopenblas0-pthread
sudo /opt/python3.11/bin/pip3.11 install numpy==1.23.5 packaging
```
Download the onnxruntime with a specific version (1.17.1)
```bash
cd
git clone --single-branch --branch v1.17.1 --recursive https://github.com/Microsoft/onnxruntime onnxruntime
cd onnxruntime
```

You have modify several things:
1. cmake/CMakeList.txt

add this in three locations in CMakeList.txt

```
function(onnxruntime_add_executable target_name)
  add_executable(${target_name} ${ARGN})
  onnxruntime_configure_target(${target_name})
  if (MSVC AND onnxruntime_target_platform STREQUAL "x86")
    target_link_options(${target_name} PRIVATE /SAFESEH)
  endif()
  target_link_libraries(${target_name} PRIVATE atomic)
endfunction()
```

```
function(onnxruntime_add_shared_library target_name)
  add_library(${target_name} SHARED ${ARGN})
  onnxruntime_configure_target(${target_name})
  target_link_libraries(${target_name} PRIVATE atomic)
endfunction()
```

```
function(onnxruntime_add_static_library target_name)
  add_library(${target_name} STATIC ${ARGN})
  onnxruntime_configure_target(${target_name})
  target_link_libraries(${target_name} PRIVATE atomic)
endfunction()
```

2. Modify this file: tools/ci_build/build.py

add this in the CMake arguments to disable CPUInfo
```
"-Donnxruntime_ENABLE_CPUINFO=OFF",
```

```
cmake_args += [
        "-Donnxruntime_ENABLE_CPUINFO=OFF",
        "-Donnxruntime_RUN_ONNX_TESTS=" + ("ON" if args.enable_onnx_tests else "OFF"),
```

After finishing above, create the your python build script. Read the build.py and you can add other args for building. 
```
sed "s/python3/\/opt\/python3.11\/bin\/python3.11/" build.sh > build311.sh
chmod +x build311.sh
./build311.sh --use_mpi --config Release --update --build --build_shared_lib --build_wheel
```


The result onnxruntime whl will be:
```
ls build/Linux/Release/dist/
```