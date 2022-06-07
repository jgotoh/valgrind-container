## Changes in this fork

- added instructions on how to integrate this in a CMake project, see [Using Valgrind with a CMake project](#using-valgrind-with-a-cmake-project)
- for instructions on how to use docker-sync for faster compilations, see [Docker-Sync](#docker-sync)

## Run Valgrind in a container
[Valgrind](http://valgrind.org/) may be difficult if not impossible to install on macOS (X86/Darwin (> 10.10, 10.11), AMD64/Darwin (> 10.10, 10.11)), 
and if succesfully installed it may not run properly or may not run at all.

A workaround for this is to run Valgrind in a Linux container. Some extra work is required, such as compilation for the container's platform,
but what you do get is a properly running Valgrind.

### Usage
First, start the container so that your source code is bind mounted in the container,
and also an interactive session is started:
```sh
docker run -tiv $PWD/path/to/my/files:/valgrind karek/valgrind:latest
```
After starting the container, your terminal should look something like this:
```sh
root@e5215faf4f73:/valgrind# 
```
Now you may compile your projact as usual, but for the container's platform (x86_64).
The container has basic build tools installed:
- dpkg-dev (>= 1.17.11)
- g++ (>= 4:5.2)
- GNU C++ compiler
- gcc (>= 4:5.2)
- GNU C compiler
- libc6-dev
- GNU C Library: Development Libraries and Header Files or libc-dev
- make
- cmake

Run valgrind as usual:
```sh
root@e5215faf4f73:/valgrind# valgrind ./your_executable
```

[Google Test](https://github.com/google/googletest) is cloned in */usr/local/src/googletest/googletest*
and pre-compiled to two static libraries *libgtest*.a and *libgtest_main.a* in */usr/local/lib*

There is a sample [Makefile](https://github.com/google/googletest/blob/master/googletest/make/Makefile) demonstrating how to compile unit tests with Google Test.

#### A Memory debugging example

Start the container with:
```
docker run -tiv $PWD/examples:/valgrind karek/valgrind:latest
```
Once in the container, compile [leak.cpp](https://github.com/karekoho/docker-valgrind/blob/master/examples/leak.cpp) and run it in Valgrind:
```
root@d3e8ebc051b8:/valgrind# make && valgrind ./leak
g++    -c -o leak.o leak.cpp
g++ -o leak leak.o
==19== Memcheck, a memory error detector
==19== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==19== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==19== Command: ./leak
==19== 
0
1
2
==19== 
==19== HEAP SUMMARY:
==19==     in use at exit: 24 bytes in 1 blocks
==19==   total heap usage: 3 allocs, 2 frees, 73,752 bytes allocated
==19== 
==19== LEAK SUMMARY:
==19==    definitely lost: 24 bytes in 1 blocks
==19==    indirectly lost: 0 bytes in 0 blocks
==19==      possibly lost: 0 bytes in 0 blocks
==19==    still reachable: 0 bytes in 0 blocks
==19==         suppressed: 0 bytes in 0 blocks
==19== Rerun with --leak-check=full to see details of leaked memory
==19== 
==19== For counts of detected and suppressed errors, rerun with: -v
==19== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

### Using Valgrind with a CMake project

To build the Dockerfile yourself and set the image's tag to valgrind, run this:
```sh
docker build -t valgrind .
```

Cd into your CMake project.
Start an interactive shell session in a container using valgrind image, mounting your sources:

```sh
docker run -tiv $PWD:/valgrind valgrind
```

We want to compile the project in docker using an out-of-source build (use an out-of-source build normally on MacOS, too!).
This avoids conflating generated files in the build process required for Valgrind with the files we generate in MacOS.
Take care to also use different binary output directories for MacOS/valgrind by setting `CMAKE_LIBRARY_OUTPUT_DIRECTORY` and `CMAKE_RUNTIME_OUTPUT_DIRECTORY` in your `CMakeLists.txt`.

``` 
# this will create the BIN_DIR flag controlling your output directory with the default directory being `bin`
set(BIN_DIR "bin" CACHE STRING "Binaries directory")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/${BIN_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/${BIN_DIR})
```

```sh
mkdir cmake-valgrind
cmake .. -DBIN_DIR=bin-valgrind
```

Afterwards just run `make` to build the project.
To run memory leak detection on your binary, you can run:

``` sh
valgrind --leak-check=full --track-origins=yes bin-valgrind/test
```

### Docker-Sync

This project should be used with [docker-sync](https://docker-sync.readthedocs.io/en/latest/) because using Docker on MacOS is incredibly slow.
Install it, create a `docker-sync.yml` in your CMake project.
A simple `docker-sync.yml` might look like this:

``` yaml
version: "2"
syncs:
  your-app-osx-sync:
    src: './'
    sync_excludes: ['cmake-valgrind', 'cmake']
    sync_strategy: 'unison'
```

For explanation of the contents consult the official [docs](https://docker-sync.readthedocs.io/en/latest/getting-started/configuration.html).
By using `sync_excludes: ['cmake-valgrind', 'cmake']`, we exclude intermediate cmake build files from the sync.
This will lead to a huge speedup during compilation.
Add every directory that will contain build files, e.g. CLion will put its files in `cmake-build-debug` by default.

Start docker-sync via `docker-sync start` in your project.
This creates a volume called `your-app-osx-sync`.
You can now run a shell in your valgrind image using your synchronized volume:

```sh
docker run --rm -tiv valgrind-osx-sync:/valgrind:nocopy valgrind
```

### Docker-Sync in Docker Compose

You can also integrate your docker-sync volume in a Docker Compose project.
Have a look at this `docker-compose.yml`:

```
version: '3.8'
services:
  your-app:
    image: valgrind
    volumes:
      - valgrind-osx-sync:/valgrind:nocopy

volumes:
  valgrind-osx-sync:
    external: true # is provided by docker-sync
```

You can now run an interactive shell in the valgrind container with the following command:
`docker-compose run --rm your-app`
