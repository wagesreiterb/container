# **chroot** 

*change root*

chroot is a Linux system call which changes the root directory of the calling process and all its children to that specified one. See *"man 2 chroot"*.

There is also a linux command line tool called *"chroot"* which is a wrapper around the chroot system call.

------

### chroot - command line tool

The command line tool "chroot" changes the root directory from "/" to any other directory. If a process is chroot to lets say "/home/que/jail" this is its new root directory.

A process which is started without "chroot" uses shared libraries from /lib, /lib32 or /lib64. If a process is jailed to lets say "/home/YOUR_USERNAME/jail" it also expects the libraries in the appropriate directories /lib, /lib32 and /lib64. But as the root directory has been changed to "/home/YOUR_USERNAME/jail" the libraries must be in "/home/YOUR_USERNAME/jail/lib".

```
# tree /home/que/jail
/home/que/jail
├── bin
│   ├── bash
│   └── ls
├── lib
│   ├── libc.so.6
│   ├── libdl.so.2
│   ├── libpcre.so.3
│   ├── libpthread.so.0
│   ├── libselinux.so.1
│   └── libtinfo.so.5
└── lib64
    └── ld-linux-x86-64.so.2

```



------

### run bash with chroot

- change to the root user (sudo su -)
- create a folder called "jail"
- create three subfolders in "jail" called "bin", "lib", "lib64"

```
mkdir jail
mkdir jail/bin
mkdir jail/lib
mkdir jail/lib64
```

- copy the executable "/bin/bash" into "/home/*YOUR_USER*/jail/bin".
- copy the executable "/bin/ls" into "/home/*YOUR_USER*/jail/bin".

Most executables require shared libraries to run.

To figure out which libraries are required for a binary "ldd" can be used.

```
# ldd /bin/bash
	linux-vdso.so.1 =>  (0x00007fff34672000)
	libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007f541c953000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f541c74f000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f541c385000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f541cb7c000)
```

These libraries are required by /bin/bash and must be copied to "/home/YOUR_USER/jail/lib" or  "/home/YOUR_USER/jail/lib64".

The below scripts copy all libraries listed by "ldd" into ether the "lib" or the "lib64" directory. (the scripts need to be executed from the "jail"-directory)

```
for lib64 in `ldd bin/* | grep \/lib64\/ | cut -d " " -f1 | sort | uniq`;
do
    cp $lib64 lib64;
done

for lib in `ldd bin/* | grep \/lib\/ | cut -d " " -f3 | sort | uniq`;
do
    cp $lib lib;
done
```

If the binaries and the libraries in place "chroot" can be used.

Execute from outside of the directory jail the following command.

```
root@myhost:/home/YOUR_USERNAME/jail# chroot . bin/bash
bash-4.3# ls
bin  lib  lib64
bash-4.3# 
```



### run a c-program with chroot

- copy the program "chroot_001.c" into "/home/YOUR_USERNAME/jail/bin"

  ```
  // chroot_001.c
  //
  #include <stdio.h>
  #include <unistd.h>
  
  int main() {
    char buff[FILENAME_MAX];
    getcwd( buff, FILENAME_MAX );
    printf("Current working dir: %s\n", buff);
    return 1;
  }
  ```

- compile it via "gcc chroot_001.c" - you will get a binary called "a.out"

  ```
  root@myhost:/home/que/jail/bin# gcc chroot_001.c 
  ```

This C-code does nothing except printing the current working directory to standard out.

- Run it without chroot

  ```
  root@myhost:/home/YOUR_USERNAME/jail/bin# ./a.out 
  Current working dir: /home/que/jail/bin
  ```

- copy the libraries as described above.

  ```
  root@myhost:/home/que/jail# ldd bin/a.out 
  	linux-vdso.so.1 =>  (0x00007ffe6e9de000)
  	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f7370ca1000)
  	/lib64/ld-linux-x86-64.so.2 (0x00007f737106b000)
  ```

- run the code within chroot

  ```
  root@myhost:/home/YOUR_USERNAME/jail# chroot . /bin/a.out
  Current working dir: /
  ```



It is possible to compile c-code with statically linked libraries. If you do so, you don't need the shared libraries located in /lib or /lib64 anymore.

- delete everything expect "/home/YOUR_USERNAME/jail/bin"

  The binary must be executed from the "bin"-directory. Otherwise it won't work.

- compile with the following compile flags "-static",  -"static-libgcc" and  "-static-libstdc++"

```
root@myhost:/home/YOUR_USERNAME/jail/bin# gcc -static -static-libgcc -static-libstdc++ chroot.c 
```

- check with "ldd" if shared libraries are not required anymore

```
root@myhost:/home/YOUR_USERNAME/jail/bin# ldd a.out
	not a dynamic executable
```

So as no shared libraries are required anymore by the binary (as they are now statically linked into the binary), the executable can be run with chroot.

```
root@myhost:/home/YOUR_USERNAME/jail# chroot . bin/a.out
Current working dir: /
```

