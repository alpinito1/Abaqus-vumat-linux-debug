# Abaqus-vumat-linux-debug
This explains how you can attach a debugger to your VUMAT.

I am not an expert on debugging nor on abaqus, but as a regular user I had to come up with a way to debug a VUMAT on linux which is the OS I regularly use.

**If you have an improvement,a better explanation or a comment, do not hesitate to push it !**

I use ubuntu 20.04. By the way, I managed to install abaqus on my ubuntu machine following the instructions given in <https://github.com/franaudo/abaqus-ubuntu>

*comments*:<br>
* I am currently using abaqus 2020
* I use gfortran 9.4.0
* I use emacs+gdb to debug
* However I think that you should be able to use the same ideas in order to attach your debugger to other versions of Abaqus, with other compilers or different IDE.

## Basic steps

The VUMAT that I develop, uses some external libraries, therefore for me it is easier to compile the subroutine manually. Thus the first steps (1. and 2.) below are only required if you want to compile your subroutine manually. Otherwise you can skip ahead to step 3 but you must make sure that you subroutine is compiled in *Debug* mode.

1. Get the compilation command in order to compile your VUMAT in debug mode. 
1. Compile your VUMAT in debug mode
1. Attach your debbuger to the running process

## 1. Obtain compilation command for your VUMAT

First I recommend you to create a local *abaqus_v6.env* file in the folder that contains your VUMAT. This file should contain the following lines:

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
As you see, in order to generate the compilation instructions, I replaced the compiler by the *echo* command, that will simply print the compilation command on the terminal. In order to do so, you sould simply ask abaqus to compile your subroutine:

```bash
abaqus make -library VUMAT.f
```
You should get the compilation instructions that look like :

```bash
-c -cpp -fPIC -extend-source -DABQ_LNX86_64 -DABQ_FORTRAN -fautomatic -mno-fma \
-fprotect-parens -fstack-protector-strong -msse3 -mavx -mavx2 -ffixed-line-length-none \
-I/home/user/path/to/your/VUMAT/folder -I/usr/SIMULIA/EstProducts/2020 -ggdb -g3 -I. \
-I/usr/SIMULIA/EstProducts/2020/linux_a64/code/include \
-I/usr/SIMULIA/EstProducts/2020/linux_a64/../SMAUsubs/PublicInterfaces \
-I/usr/SIMULIA/EstProducts/2020/linux_a64/code/bin/SMAExternal/pmpi/include VUMAT.f
```

## 2. Compile your VUMAT

The regular process to attach your debugger to your VUMAT consists in launching the job and then attaching the debugger. If your simulation is small, as it is often the case when debugging, chances are that by the time that you try to attach to the process the simulation is over. Therefore yous might want to pause your VUMAT. To do so, you could add a *sleep(10)* to pause the execution for 10 seconds, but be careful how you do it in order to avoid pausing the execution on every single VUMAT call. The line of code should look something like:

```fortran
call sleep(10)
```
Now you can actually compile your subroutine simply by executing the compilation command previously generated adding *gfortran* at the beginning:


```bash
gfortran -c -cpp -fPIC -extend-source -DABQ_LNX86_64 -DABQ_FORTRAN -fautomatic -mno-fma \
-fprotect-parens -fstack-protector-strong -msse3 -mavx -mavx2 -ffixed-line-length-none \
-I/home/user/path/to/your/VUMAT/folder -I/usr/SIMULIA/EstProducts/2020 -ggdb -g3 -I. \
-I/usr/SIMULIA/EstProducts/2020/linux_a64/code/include \
-I/usr/SIMULIA/EstProducts/2020/linux_a64/../SMAUsubs/PublicInterfaces \
-I/usr/SIMULIA/EstProducts/2020/linux_a64/code/bin/SMAExternal/pmpi/include VUMAT.f
```
This should create an object *VUMAT.o*

If you find a problem regarding the *vaba_param.inc*, it is probably because by default abaqus will try replace this include by the appropriate one *vaba_param_sp.inc* or *vaba_param_dp.inc* depending on whether you are using single or double precision, respectively. So you can manually replace *vaba_param.inc* by *vaba_param_dp.inc* if you use double precision as I do.

If your compilation requires other external libraries, you can adapt the previous compilation command in order to compile your own subroutine.

And now you are ready to launch your simulation and then attach the debugger.


## 3. Attach your debugger to your simulation

When you launch an abaqus job, it actually runs multiples process that are forked from the original one. By default Linux kernel will prevent you from attaching to these forked processes. You can see more details about this here <https://askubuntu.com/questions/146160/what-is-the-ptrace-scope-workaround-for-wine-programs-and-are-there-any-risks>.

A workaround consists in disabling this option manually. There are multiple ways of doing it, all requiring *sudo* rights, so I choose the following:

```bash
sudo su -
echo 0 > /proc/sys/kernel/yama/ptrace_scope
```

In the case of a explicit simulation, abaqus will actually in the end fork in order to run the *explicit_dp* binary. This binary should be located in:

```bash
/usr/SIMULIA/EstProducts/2020/linux_a64/code/bin/explicit_dp
```
Now you can load the binary into your debugger, in my case I use Emacs+GDB.

```bash
emacs
```
```lisp
M-x cd
/usr/SIMULIA/EstProducts/2020/linux_a64/code/bin/
M-x gud-gdb
explicit_dp
M-x cd
/home/user/path/to/your/VUMAT/folder
C-x-f
VUMAT.f
```
You can now add a breakpoint after the `sleep(10)` call. In my case from `gdb` I simply do:

```gdb
b VUMAT.f:68
```

Alternatively, you can simply load the binary into `gdb` and add the breakpoint:

```bash
gdb /usr/SIMULIA/EstProducts/2020/linux_a64/code/bin/explicit_dp
```
```gdb
cd /home/user/path/to/your/VUMAT/folder
b VUMAT.f:68
```
It is important that you prepare your debugger before submitting the job, since you only will have 10 seconds to attach to the process if you want to catch the very first execution. 

Before submitting your job, you should find a way of identifying the process ID of your job. I simply use the system monitor: `gnome-system-monitor` and I look into the `processes` tab.

Now you can submit your job :


```bash
cd /home/user/path/to/your/VUMAT/folder
abaqus debug -j Job_name -user ./VUMAT.o -double both -interactive -ask_delete OFF -input sim.inp -stop vumat
```
Most likely all those options are not required, but those are the ones I use.

By doing so, your process is submitted and normally you should pause as requested (`sleep(10)`) in you VUMAT. Go to your monitor and seek for the `explicit_dp` process and grab its process ID.

Finally you should quickly go to your debugger and attach to it (in this case the process ID will be 46709):

```gdb
attach 46709
```

Alternatively you can interrupt the execution, add another breakpoint and the continue the execution:

```gdb
attach 46709
b VUMAT.f: 70
c
```

That this is it, go find that bug!!
