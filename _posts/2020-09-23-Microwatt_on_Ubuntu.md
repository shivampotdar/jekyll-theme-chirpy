---
title: Running Microwatt on Ubuntu 20.04
date: 2020-09-23 18:33 +0530
categories: [cpus]
tags: [cpus, softcores, powerpc]
image: /assets/img/blogs/uwatt/microwatt-title.png
---


### What is Microwatt?

You probably know already :)

IBM recently made their POWER ISA open sourced with a very liberal license.
[OpenPOWER Foundation](https://en.wikipedia.org/wiki/OpenPOWER_Foundation) joined the [Linux Foundation](https://en.wikipedia.org/wiki/Linux_Foundation) in August 2019 and in the same month [Microwatt](https://github.com/antonblanchard/microwatt) was released to the community.

Microwatt is a VHDL2008-based POWER ISA 3.0 core, originally written by Anton Blanchard, supporting Linux, MicroPython and Zephyr RTOS.

It is also supported in [FuseSoC](https://github.com/olofk/fusesoc) for quick simulation and FPGA bringup!

### Installation on Ubuntu 20.04

- The GitHub repo for [Microwatt](https://github.com/antonblanchard/microwatt) does give instructions in short and also a cute little GIF showing 1+2 = 3 ;)

- It does have pointers to what you need to set up on non-POWER (say x86) systems to get it running, you would still need some digging around.

- I was able to successfully get simulations running on Ubuntu 20.04 and decided to list down the steps here:
    1.  Make a new directory wherever you like --  ```bash
        mkdir /home/$USER/uwatt
        cd ~/uwatt
        ```
    3.  Download PowerPC cross toolchain
        - You can select `powerpc64le-power8` and `glibc` on toolchains.bootlin.com or 
        get the stable version as of date directly with
        ```bash 
        wget https://toolchains.bootlin.com/downloads/releases/toolchains/powerpc64le-power8/tarballs/powerpc64le-power8--glibc--stable-2020.02-2.tar.bz2
        tar -xvf powerpc64le-power8--glibc--stable-2020.02-2.tar.bz2
        ```

        - Since this is a prebuilt toolchain, all you need to use it is 
        ```bash
        export PATH=$PATH:/home/$USER/uwatt/powerpc64le-power8--glibc--stable-2020.02-2/bin
        export CROSS_COMPILE=powerpc64le-linux-
        ```
    4. Build Micropython
       ```bash
        git clone https://github.com/micropython/micropython.git
        cd micropython/ports/powerpc
        make -j$(nproc)
        cd ../../../
       ```
    5. Build GHDL from source
       - Check if you have llvm installed with `which llvm-config`. 
       If you see "... not found", install with `bash -c "$(wget -O - https://apt.llvm.org/llvm.sh)"`
       - Install gnat4.9 packages (needed by GHDL): 
       ```bash
        wget http://archive.ubuntu.com/ubuntu/pool/universe/g/gnat-4.9/gnat-4.9-base_4.9.3-3ubuntu5_amd64.deb
        wget http://archive.ubuntu.com/ubuntu/pool/universe/g/gnat-4.9/libgnat-4.9_4.9.3-3ubuntu5_amd64.deb
        sudo dpkg -i ./gnat-4.9-base_4.9.3-3ubuntu5_amd64.deb
        sudo dpkg -i ./libgnat-4.9_4.9.3-3ubuntu5_amd64.deb
       ```
       - Clone GHDL from GitHub
        (ideally you should go for a stable release, but v0.37 does not support llvm10, in which case you will need to specify older version of llvm above. The master branch doesn't seem to have this issue.) 
       ```bash 
       git clone https://github.com/ghdl/ghdl.git
       cd ghdl
       mkdir build && cd build
       ../configure --with-llvm-config --prefix=/home/$USER/uwatt/ghdl_build
       make
       make install
       ```
       - If you face dependency issues, Google would mostly help. In case you realise that a package is not available in focal repositories, you can try the wget and deb method (similar to gnat above)
       - After building, add ghdl to path with 
       ```bash
       export PATH=$PATH:/home/$USER/ghdl_build/bin
       ```

    6. Build Microwatt
       ```bash
        git clone https://github.com/antonblanchard/microwatt
        cd microwatt
        make
       ```

    7. Link Micropython image
    (note that here we are using pre-built image inside microwatt repository. At least on my system, I am not able to provide inputs to the micropython terminal with the image built above.
    This issue has been reported on microwatt GitHub - https://github.com/antonblanchard/microwatt/issues/246)

        ```bash
        ln -s micropython/firmware.bin main_ram.bin
        ```

    1. Run Microwatt!
    (By sending output logs to /dev/null, you can also specify a file name if needed)
        ```bash
        ./core_tb > /dev/null
        ```
        Note that this is similar to running a Python interpreter shell on your local system, but here the execution is through simulation mechanism.

    1. (Optional) Run bare-metal C code on Microwatt!
    By this method, you can write simple C test programs and check their operation. This is achieved by compiling them using the GCC cross-compiler. 
        a. I have made changes to the Makefile in hello_world directory to allow any filename and linking main_ram.bin with compiled binary in single step. First try running the existing hello_world example.
        ```bash
        cd ~/uwatt/microwatt/hello_world
        wget https://pastebin.com/raw/2WdH5Z4d -O Makefile
        make
        ```

        b. Try `cd ..` and `./core_tb > /dev/null` and you would be greeted by Microwatt :)
        ```
               .oOOo.     
             ."      ". 
             ;  .mw.  ;   Microwatt, it works.
              . '  ' .    
               \ || /    
                ;..;      
                ;..;      
                `ww'   
        ```

        c. Now you can go and modify `hello_world.c` or even create a new file say `shivam_test.c` (retain the includes and console_init lines).
        To run this new file 
        ```bash
        make clean
        make FILE=shivam_test
        ```
        
        d. Here is an example of printing sum of 1 to 10 using C:
        ```c
        #include <stdint.h>
        #include <stdbool.h>
        #include "console.h"

        void iprint(int n)
          { 
            if (n/10)
                iprint(n/10);
            putchar(n%10 + '0');
          }

        int main(void)
        {
            console_init();

            int i = 0, sum=0;

            for(i;i<=10;i++)
            {
                sum += i;
                iprint(sum);
                putchar(10);
            }

            puts("Shivam");
        }
        ```

        Output:
        ```
        0
        1
        3
        6
        10
        15
        21
        28
        36
        45
        55
        Shivam
        ```
       
That's all folks! :)
