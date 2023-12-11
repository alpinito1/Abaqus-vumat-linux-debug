# Abaqus-vumat-linux-debug
This explains how you can attach a debugger to your VUMAT.

I am not an expert on debugging nor on abaqus, but as a regular user I had to come up with a way to debug a VUMAT on linux which is the OS I use. Specifically I use ubunto 20.04. By the way, I managed to install abaqus on my ubuntu machine following the instructions given in <https://github.com/franaudo/abaqus-ubuntu>

*comments*:<br>
* I am currently using abaqus 2020, but I think that you should be able to use the same ideas in order to attach your debugger to other versions of Abaqus.
* I use gfortran
* I use emacs+gdb to debug

## Install prerequisites

Basic steps

* Get the compilation command in order to compile your VUMAT in debug mode
* Compile your VUMAT in debug mode
* attach your debbuger to the running process

## Compile your VUMAT

First I recommend you to creat a local *abaqus_v6.env* file in the folder that contains your VUMAT. This file should contain the following lines:

```python
import os

abaHomeInc = os.path.abspath(os.path.join(os.environ.get('ABA_HOME', ''), os.pardir))

#fortCmd = "gfortran"
fortCmd = "echo"

compile_fortran = [fortCmd,
                   '-c', '-cpp', '-fPIC','-extend-source',
                   '-DABQ_LNX86_64', '-DABQ_FORTRAN',
                   '-fautomatic',
                   '-mno-fma', 
                   '-fprotect-parens',
                   '-fstack-protector-strong',
		   '-msse3',
                   '-mavx', '-mavx2',
                   '-ffixed-line-length-none',
                   '-I%I', '-I'+abaHomeInc, '%P']
		   
# Uncomment for debugging using gdb
compile_fortran=compile_fortran + ['-ggdb', '-g3'] 

link_sl = [fortCmd,
           '-fPIC', '-shared','-Wl,--add-needed', 
           '%E', '-Wl,-soname,%U', '-o', '%U', '%F', '%A', '%L', '%B',
           '-Wl,-Bdynamic']

link_exe = [fortCmd,
            '-fPIC',
            '-Wl,-Bdynamic', '-Wl,--add-needed', '-o', '%J', '%F', '%M', '%L', '%B', '%O']

del fortCmd
del abaHomeInc
```
As you see, in order to generate the compilation instructions, 

