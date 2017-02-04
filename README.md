# NeoJamVM

This is a fork of [JamVM](http://jamvm.sourceforge.net/), a compact Java Virtual Machine that works either as an alternative VM in OpenJDK or a standalone JVM with GNU Classpath.
It might not support all the things HotSpot supports, but it does run Jetty :-)

This fork replaces the insufferable autotools build system with CMake and adds fixes for FreeBSD+OpenJDK support.

Not all operating systems, CPU architectures and feature flags work with the new build system.
It's currently only tested on FreeBSD/amd64 with OpenJDK 7.

## Installation

Something like this:

```bash
$ sudo pkg install openjdk cmake ninja
$ mkdir build
$ cd build
$ cmake .. -DCMAKE_BUILD_TYPE=Release -DJAVA_RUNTIME_LIBRARY=openjdk7 -GNinja
$ ninja
$ sudo mkdir /usr/local/openjdk7/jre/lib/amd64/jamvm
$ sudo cp libjvm.so /usr/local/openjdk7/jre/lib/amd64/jamvm/libjvm.so
```

## Usage

```bash
$ /usr/local/openjdk7/jre/bin/java -XXaltjvm=jamvm -jar some.jar
```
