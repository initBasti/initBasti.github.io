# Building the kernel from source

As part of the 20 challenge series, called Eudyptula (that was canceled in 2017),  
I now arrive at the second challenge, which focused around the build process of the kernel.  

If you are interested in the journey so far, checkout:  
[1. Hello world module](https://sebastianfricke.me/eudyptula-challenge-Part1/)  

**If you have not solved the challenge, stop right after the description of the task and try it yourself first!**

## Task  

Now that you have written your first kernel module, it's time to take  
off the training wheels and move on to building a custom kernel.  No  
more distro kernels for you, for this task you must run your own kernel.  
And use git!  Exciting isn't it!  No, oh, ok...  
  
The tasks for this round is:  
  - download Linus's latest git tree from git.kernel.org (you have to  
    figure out which one is his, it's not that hard, just remember what  
    his last name is and you should be fine.)  
  - build it, install it, and boot it.  You can use whatever kernel  
    configuration options you wish to use, but you must enable  
    CONFIG_LOCALVERSION_AUTO=y.  
  - show proof of booting this kernel.  Bonus points for you if you do  
    it on a "real" machine, and not a virtual machine (virtual machines  
    are acceptable, but come on, real kernel developers don't mess  
    around with virtual machines, they are too slow.  Oh yeah, we aren't  
    real kernel developers just yet.  Well, I'm not anyway, I'm just a  
    script...)  Again, proof of running this kernel is up to you, I'm  
    sure you can do well.  
  
Hint, you should look into the 'make localmodconfig' option, and base  
your kernel configuration on a working distro kernel configuration.  
Don't sit there and answer all 1625 different kernel configuration  
options by hand, even I, a foolish script, know better than to do that!  
  
After doing this, don't throw away that kernel and git tree and  
configuration file.  You'll be using it for later tasks, a working  
kernel configuration file is a precious thing, all kernel developers  
have one they have grown and tended to over the years.  This is the  
start of a long journey with yours, don't discard it like was a broken  
umbrella, it deserves better than that.  


## Execution

As mentioned in the book [Linux Kernel in a nutshell](http://www.kroah.com/lkn/)
by [Greg Kroah-Hartman](http://www.kroah.com),  
you should **not** use the /usr/src directory as a location for a custom hacking kernel.
[\[1\]](http://files.kroah.com/lkn/lkn\_pdf/ch01.pdf)  
The manual page of the unix hierarchie, explains the general guidelines:  
`man hier | grep "/usr/src" -A 3`  
Additionally it is generally not adviced to do all the building with root permission,  
because this can lead to unexpected behavior (as mentioned in the same chapter).  

I have gathered a few resources, that explain in more detail how the build process works:  
[Freecode camp article about this topic from 2016](https://www.freecodecamp.org/news/building-and-installing-the-latest-linux-kernel-from-source-6d8df5345980/)  
[cyberciti article (last update 2019)](https://www.cyberciti.biz/tips/compiling-linux-kernel-26.html)  

#### You start of by getting the source:  
Go to [Kernel.org](https://www.git.kernel.org/) and search for Linus tree (**torvalds**).
Now you can either *download the tarball* and unzip it or you can clone it using *git*.  

#### Configure to your needs:  
Next, we start to customize the kernel to our needs and likings.  
The Gentoo Wiki presents a very good introduction to the different options available:  
[Gentoo Wiki: section **Configuration Tools**](https://wiki.gentoo.org/wiki/Kernel/Configuration)  
which boils down to the major options:  
* make config  
*(Go through every question, without the possibility to change a option afterwards, meh...)*  
* make menuconfig(Ncurses UI) & make gconfig(GTK-based GUI) & make xconfig(QT-based GUI)  
*(Quite similar to make config, but easier to use with the user interface)*  
* make defconfig  
*(Takes the config of the person that created the tree you want to build e.g. Linus)*  
* make randconfig  
*(Just in case you feel lucky...)*  
* make localmodconfig  
*(Create a config based on current config and loaded modules (lsmod).  
Disables any module option that is not needed for the loadedmodules.
[\[2\]](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/README?id=refs/tags/v4.3.3))*

#### Building process:  
Once that is done, you can build the kernel with the given configuration.  

Go to the linux source directory and type:  
**make**  
options:  
* **-j** (number of cores + 1) *Speed up the build process, **Do not forget the number ;)***   
(lookup the number of cores with `cat /proc/cpuinfo`)  
* **ARCH=**(X86_64,X86,ARM_64....) *Used to build architecture specific code* 
(lookup with `uname -m`)  

If you want to speed up, the compilation of the kernel for repeated builds:  
Use **ccache**, a compile cache for C/C++  

This will probably take a while...  


Afterwards, you have to build the modules as defined in the configuration with:  
`sudo make modules_install` [-j {Number of CPUs + 1}]  
and finally you install the kernel into your system and bootloader with:  
`sudo make install`  

## Proof  

So, how do we proof that we actually have the kernel up and running?  
First of all, we can execute `uname -r`  
*Example output:*  
```
basti@basti:~$ uname -r
5.6.0-10895-g4c205c84e249
```

This contains two parts:  
**5.6.0** (The kernel version)  
**10895-g4c205c84e249** (The local version of the source tree)  

To find the current kernel version, go to the kernel source tree and look at:  
`make kernelversion`  

And to find the local version of your source tree, type:  
```
make prepare
./scripts/setlocalversion
```

If those match with the uname output, then the kernel is correctly set up.  

## Update the source tree

The tree will be updated regulary, so the question is:  
**What should I do to update my source tree to current version?**  
*(This only applies, if you used to git to get source code.)*  

The changes, that made it to the most recent version of linus's tree, went through a lot of stages.  
If you just try to execute `git fetch` and `git pull`, you will encounter a colorful variety of merge conflicts.  
This means that we have to fetch the changes and then checkout to the new upstream HEAD. 
```
git fetch origin
git checkout origin/master
```
But we will encounter a small problem with just doing the checkout: *a detached HEAD*  
**What is a detached HEAD?**  
This is state, where you are no longer within any branch, which is totally fine  
for just looking around without making any changes.  
But as soon as you want to make changes it becomes a bit problematic.  
**How to avoid a detached HEAD?**  
Instead of checking out to the newest commit from the source tree  
use `git reset --hard origin/master`.  
That approach will move the HEAD to newest commit and change all your files to the new state.  
**Careful!** any temporary changes made to the source code are lost, that way!  
**How to merge changes of a detached HEAD into a branch?**  
* Get the commit hash of your change `git log -n 1`  
* Go to the branch you want to use `git checkout master`  
* Create a temporary branch for the change `git branch tmp <commit-hash>`  
* Merge the temporary branch into the branch `git merge tmp`,  
verify that you are on the correct branch before merging with `git branch`  
[\[3\]](https://stackoverflow.com/a/10229202/9918329)  

## A Problem that I had

`No rule to make target 'debian/certs/debian-uefi-certs.pem', needed by 'certs/x509_certificate_list'`  
I created my config though menuconfig and afterwards the debian certificate was added to the config option:  
`CONFIG_SYSTEM_TRUSTED_KEYS`  
You just have to set that option to a empty string \(""\), to fix this.  
[\[4\]](https://www.linuxquestions.org/questions/linux-embedded-and-single-board-computer-78/failed-to-build-linux-kernel-4175667466/)  

## References  
[\[1\]](http://files.kroah.com/lkn/lkn_pdf/ch01.pdf) *Introduction chapter of the Linux kernel in a nutshell book with advice about where to build the Kernel.*  
[\[2\]](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tree/README?id=refs/tags/v4.3.3) *About the make localmodconfig from the Kernel README file*  
[\[3\]](https://stackoverflow.com/a/10229202/9918329) *Stackoverflow answer for working with a detached HEAD*  
[\[4\]](https://www.linuxquestions.org/questions/linux-embedded-and-single-board-computer-78/failed-to-build-linux-kernel-4175667466/) *Answer on linuxquestions* 
