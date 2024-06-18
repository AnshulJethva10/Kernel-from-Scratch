# Kernel From Scratch
Hello Everyone, My name is Anshul Jethva, and in this Repository you will be learning how to create a kernel from scratch.

First things first making a kernel from scratch is not that hard but you need to have some basic knowledge of *kernel* and *operating system* and how it works. So if don't know anything about the operating system and kernel, I suggest you to watch some videos form the youtube or if you want to go deep you can also read books. I have create a Resources section [here](https://github.com/AnshulJethva10/Kernel-from-Scratch/blob/main/README.md#resources) where you can get links for that

So now that you have learned about what is OS, kernel and how does it work we can dive little bit into theory. In this tutorial we will write a simple kernel for 32-bit x86 and boot it, So  first we need to know about how does an x86 machine boot:

- Before we think about writing a kernel, let’s see how the machine boots up and transfers control to the kernel:

- Most registers of the x86 CPU have well defined values after power-on. The Instruction Pointer (EIP) register holds the memory address for the instruction being executed by the processor. EIP is hardcoded to the value 0xFFFFFFF0. Thus, the x86 CPU is hardwired to begin execution at the physical address 0xFFFFFFF0. It is in fact, the last 16 bytes of the 32-bit address space. This memory address is called reset vector.

- Now, the chipset’s memory map makes sure that 0xFFFFFFF0 is mapped to a certain part of the BIOS, not to the RAM. Meanwhile, the BIOS copies itself to the RAM for faster access. This is called shadowing. The address 0xFFFFFFF0 will contain just a jump instruction to the address in memory where BIOS has copied itself.

- Thus, the BIOS code starts its execution.  BIOS first searches for a bootable device in the configured boot device order. It checks for a certain magic number to determine if the device is bootable or not. (whether bytes 511 and 512 of first sector are 0xAA55)

- Once the BIOS has found a bootable device, it copies the contents of the device’s first sector into RAM starting from physical address 0x7c00; and then jumps into the address and executes the code just loaded. This code is called the bootloader.

- The bootloader then loads the kernel at the physical address 0x100000. The address 0x100000 is used as the start-address for all big kernels on x86 machines.

- All x86 processors begin in a simplistic 16-bit mode called real mode. The GRUB bootloader makes the switch to 32-bit protected mode by setting the lowest bit of CR0 register to 1. Thus the kernel loads in 32-bit protected mode.

- Do note that in case of linux kernel, GRUB detects linux boot protocol and loads linux kernel in real mode. Linux kernel itself makes the switch to protected mode.

# Prerequisites

Now that we know how the x86 machine boots, our second step will be setting up the enviroment.

## 1. Linux
I know that most of you are using a Windows OS but to develop a kernel you need a linux OS. There are multiple ways you can set it up but in this repo i am going to use `wsl` which stands for *Windows Server for Linux* and it is pretty easy to set up:
1. Open PowerShell or Windows Command Prompt in administrator mode by right-clicking and selecting "Run as administrator", enter the wsl --install command, then restart your machine.
```
wsl --install
```
![wsl](https://learn.microsoft.com/en-us/windows/wsl/media/wsl-install.png)


2. Once you have installed WSL, you will need to create a user account and password for your newly installed Linux distribution.

![Installation](https://learn.microsoft.com/en-us/windows/wsl/media/ubuntuinstall.png)

- Alternative to this you can also go to the Microsoft store and search for the Ubuntu and install it

![Microsoft Store](https://lh3.googleusercontent.com/u/0/drive-viewer/AKGpihZklpOLeWzpk0JYmYmwZDdGa3Vu0MVaVVo_NOJd8POtQu9qbSBro_qCMCo79eUz1bZ4K-ZYnZG7NiEO7eULiL16Bx6WI_K_sw=w1920-h919-rw-v1)

for any futher guide for the installation regarding `wsl` you can visit this [website](https://learn.microsoft.com/en-us/windows/wsl/install)

## 2. Cross-Compiler
There is a whole article of why do i need cross-compiler, here.

But in short you need cross compiler because the compiler must know the correct target operating system otherwise you will get lots of error to check if you need a cross compiler or not just type
```
gcc -dumpmachine
```
and if you get output as *i686-elf* you don't need the cross-compiler but trust me guys no one is gonna get this output.

So lets start building a cross compiler:

Now building a cross-compiler is very much important and you have to do it very carefully

- First you need to decide where to install your new compiler. It is dangerous and a very bad idea to install it into system directories.installing into `$HOME/opt/cross` is normally a good idea. For that you need to export some variables
```
export PREFIX="$HOME/opt/cross"
export TARGET=i686-elf
export PATH="$PREFIX/bin:$PATH"
```
- Run these into you terminal one by one.

- You need the following in order to build GCC:
  - GCC (existing release you wish to replace), or another system C compiler
  - Make
  - Bison
  - Flex
  - GMP
  - MPFR
  - MPC
  - Texinfo
  - ISL (optional)

- How to install them: 
  - `sudo apt install build-essential` -> for *compiler* and *make*
  - `sudo apt install bison` -> for *Bison* 
  - `sudo apt install flex` -> for *Flex
  - `sudo apt install libgmp3-dev` -> for *GMP*
  - `sudo apt install libmpc-dev` -> for *MPC*
  - `sudo apt install libmpfr-dev` -> for *MPFR*
  - `sudo apt install texinfo` -> for *Texinfo*
  - `sudo apt install libisl-dev` -> for *ISL*

- After installing these you need `gcc` and `binutils`
  - go to home directory `cd ~` and then create a src folder `mkdir src` then go to that folder `cd src`
  - once you are in the `src` folder you need to download gcc and binutils there.
  
- For gcc:
```
curl -O https://ftp.gnu.org/gnu/gcc/gcc-11.4.0/gcc-11.4.0.tar.gz
tar -xzvf gcc-11.4.0.tar.gz
cd $HOME/src
# The $PREFIX/bin dir _must_ be in the PATH. We did that above.
which -- $TARGET-as || echo $TARGET-as is not in the PATH
mkdir build-gcc
cd build-gcc
../gcc-11.4.0/configure --target=$TARGET --prefix="$PREFIX" --disable-nls --enable-languages=c,c++ --without-headers
make all-gcc
make all-target-libgcc
make install-gcc
make install-target-libgcc
```
- For binutils 
```
curl -O http://ftp.gnu.org/gnu/binutils/binutils-2.42.tar.gz
tar -xzvf binutils-2.42.tar.gz
cd $HOME/src
mkdir build-binutils
cd build-binutils
../binutils-2.42/configure --target=$TARGET --prefix="$PREFIX" --with-sysroot --disable-nls --disable-werror
make
make install
```

- Run all of the above commands in your terminal one by one.
- So now you have a cross-compiler. It does not have access to a C library or C runtime yet, so we cannot use most of the standard includes or create runnable binaries. But it is quite sufficient to compile the kernel we will be making shortly.
- You can now run your new compiler by invoking something like:
```
$HOME/opt/cross/bin/$TARGET-gcc --version
```
- To use your new compiler simply by invoking $TARGET-gcc, add $HOME/opt/cross/bin to your $PATH by typing:
```
export PATH="$HOME/opt/cross/bin:$PATH"
```
- This command will add your new compiler to your PATH for this shell session.

# Kernel Building
- Now, to create a kernel we will need 3 files
  - boot.s - kernel entry point that sets up the processor environment
  - kernel.c - your actual kernel routines
  - linker.ld - for linking the above files 

- To start the operating system, an existing piece of software will be needed to load it. This is called the bootloader and in here we will be using GRUB.
- To install GRUB run the following command:
```
sudo apt install grub-pc-bin
sudo apt install grub-efi-amd64-bin
sudo apt install grub-common
sudo apt install mtools
```

## 1. Bootstrap Assembly 
- We will now create a file called `boot.s`
- I have already wrote the file `boot.s` [here](https://github.com/AnshulJethva10/Kernel-from-Scratch/blob/main/boot.s), which has multiple comments, so first read all the comments carefully for better understanding then write is down in you file. This way you won't be just copy-pasting the code, You will be understanding each and every line of the code.
- After writing the code, We can then assemble boot.s using:
```
i686-elf-as boot.s -o boot.o
```

## 2. Implementing the Kernel
- So far we have written the bootstrap assembly stuf that sets up the processor such that high level languages such as C can be used.
- Now we will be creating a file `kernel.c` where we will write the logic of our kernel.
- As above the code is already written with the comments, which you can find [here](https://github.com/AnshulJethva10/Kernel-from-Scratch/blob/main/kernel.c). So first read them carefully and then write it down in you file
- Now compile using:
```
i686-elf-gcc -c kernel.c -o kernel.o -std=gnu99 -ffreestanding -O2 -Wall -Wextra
```

## 3. Linking the Kernel
- After assembling `boot.s` and compiling `kernel.c` 2 object files will be generated. To create our final kernel we need to link both of these files for that we need a linker script.
- Our `linker.ld` script is [here](https://github.com/AnshulJethva10/Kernel-from-Scratch/blob/main/linker.ld), read the comments and write it in your linker script.
- We can then link our kernel using:
```
i686-elf-gcc -T linker.ld -o myos.bin -ffreestanding -O2 -nostdlib boot.o kernel.o -lgcc
```
- This will create a `myos.bin` file which is our kernel ready to boot.
- But before we boot it we need to verify multiboot using GRUB that we installed earlier.
- Simply run the following command and then type `echo $?` if the output is *0* that means that our file has a valid multiboot.
```
grub-file --is-x86-multiboot myos.bin
```

## 4. Booting the Kernel
- Now we have 2 choices here 
  - We run the kernel without bootable medium.
  - We create a .iso file for our kernel

- Both ways work fine for me, but i recommend making an .iso file.

### 1. Without Bootable Medium:
- If you don't want to make a bootable medium then you are done. You just need a virtual machine to boot your kernel.
- To boot the kernel I use the QEMU Virtual Machine / Emulator. Just run the command below to install the QEMU and then boot the kernel using QEMU :
```
sudo apt install qemu
qemu-system-i386 -kernel myos.bin
```

### 2. With Bootable Medium:
- We can create a .iso file containing the GRUB bootloader and our kernel using the program grub-mkrescue.
- But before that we need to install `xorriso` first.
```
sudo apt install xorriso
```
- After installing `xorriso` create a file called `grub.cfg` and write following:
```
menuentry "myos" {
	multiboot /boot/myos.bin
}
```
- We can now create a bootable image of our operating system by typing these commands:
```
mkdir -p isodir/boot/grub
cp myos.bin isodir/boot/myos.bin
cp grub.cfg isodir/boot/grub/grub.cfg
grub-mkrescue -o myos.iso isodir
```
- After these you have successfully created an .iso image of your kernel. Now we need to test it to do so run:
```
qemu-system-i386 -cdrom myos.iso
```
- This should start a new virtual machine containing only your ISO as a cdrom.Simply select myos and if all goes well, you should see the words "Hello, Kernel World!"

**Congratulations! You have successfully created a kernel from scratch.**

# Conclusion 
I want to thank you all for joining me in this journey. If you find this repository helpful, Welcome. This is not the end, This kernel was just a beginning. I will be making more projects like this and share them with you. I have learned this from [kernel101](https://arjunsreedharan.org/post/82710718100/kernels-101-lets-write-a-kernel) and [Bare-Bones](https://wiki.osdev.org/Bare_Bones) websites and combining both created a kernel. Most of the steps that we have done are from the [Bare-Bones](https://wiki.osdev.org/Bare_Bones) website so make sure to check it out for learning in detail. I have also added a section below for all the resources to gain more knowledge about Operating System.

# Resources
Youtube Videos:
- [Operating System - Gate Smashers](https://www.youtube.com/playlist?list=PLxCzCOWd7aiGz9donHRrE9I3Mwn6XdP8p)
- [What is an Operating System as Fast As Possible](https://youtu.be/pVzRTmdd9j0?si=zwXIOBrWs7SxC7T_)
- [Operating Systems: Crash Course Computer Science #18](https://youtu.be/26QPDBe-NB8?si=PBRw09Mj99zsV4aS)
- [What is a Kernel?](https://youtu.be/5S-tTDeFZfY?si=UY-y3oCfPSU_T2xW)
- [What is a Kernel and what does it do? Explore the Kernels of Linux, Windows, and MacOS.](https://youtu.be/IvGdY6luTtU?si=XfclCAR2jMl3aDh0)

Books:
- [Operating System Concepts](https://www.os-book.com/OS10/) 
- [A Journey in Creating an Operating System Kernel](https://539kernel.com/)

Websites:
- [OSDev.org](https://wiki.osdev.org/Expanded_Main_Page)
- [arjunsreedharan.org](https://arjunsreedharan.org/)
