# How to create a patch

The subject of this post is the creation of incremental changes and the workflow  
of submitting the change in the appropriate format.  

I go through the following steps: make the **Modification**, **Proof** that it works and create a **Patch**.  

If you are interested in the journey so far, checkout:  
[1. Hello world module](https://sebastianfricke.me/eudyptula-challenge-Part1/)  
[2. Building the kernel](https://sebastianfricke.me/eudyptula-challenge-Part-2/)  

**If you have not solved the challenge, stop right after the description of the task and try it yourself first!**

## Task  

Now that you have your custom kernel up and running, it's time to modify  
it!  
  
The tasks for this round is:  
  - take the kernel git tree from Task 02 and modify the Makefile to  
    and modify the EXTRAVERSION field.  Do this in a way that the  
    running kernel (after modifying the Makefile, rebuilding, and  
    rebooting) has the characters "-eudyptula" in the version string.  
  - show proof of booting this kernel.  Extra cookies for you by  
    providing creative examples, especially if done in intrepretive  
    dance at your local pub.  
  - Send a patch that shows the Makefile modified.  Do this in a manner  
    that would be acceptable for merging in the kernel source tree.  
    (Hint, read the file Documentation/SubmittingPatches and follow the  
    steps there.)  

## Modification  
This is straightforward, just open the Makefile in the source directory of the kernel.  
And add a name tag of your choice to the `EXTRAVERSION=` option.  
Afterwards, make a commit of the change and recompile the kernel.  
`git add .`  
`git commit -s` (-s to sign the commit for the patch)   

**Why do we have to make a commit?**  
Because otherwise the -dirty or '+' tag is added to the kernel name.  
[\[1\]](https://stackoverflow.com/a/25091076/9918329)  

## Proof  

Executing `make kernelversion` within the kernel source directory, yields in my case:  
`5.6.0eudyptula`  
That proofs that the EXTRAVERSION is added to the kernel name.  

In order to proof, that it was booted:  
`uname -r`  
result: `5.6.0eudyptula-12662-gf788bf746653`  

And finally to verify, that this version runs on the current version of the source tree:  
(Within the kernel source tree)  
`make prepare` && `./scripts/setlocalversion`  

result:  `-12662-gf788bf746653`

#### Grub-bootloader:  

As a little bonus, how do I set the default kernel for the Grub bootloader  
and how do I determine the current default?  

**To find the current default:**  
`grep "set default" /boot/grub/grub.cfg`  
If you see: `set default="0"`, then grub will take the first menu entry in the list.  
In order to see all the menuentries, type: `grep menuentry /boot/grub/grub.cfg`.   

Otherwise if you see something like this:  
set default="gnulinux-advanced-ef962dd6-4592-467e-88d8-a7a283e6c483>gnulinux-5.6.0eudyptula-12662-gf788bf746653-advanced-ef962dd6-4592-467e-88d8-a7a283e6c483"  
Then you find the kernel version right after the ">" sign : **5.6.0eudyptula-12662-gf788bf746653**  

**To set the default option:**  
Backup the current /etc/default/grub file:  
`sudo cp /etc/default/grub /etc/default/grub.bak`  

Find the *$menuentry_id_option* option:  
`grep submenu /boot/grub/grub.cfg`  
submenu 'Advanced options for Debian GNU/Linux' $menuentry_id_option '**gnulinux-advanced-ef962dd6-4592-46**  

Find the kernel ID:  

`grep gnulinux /boot/grub/grub.cfg`  

---

menuentry 'Debian GNU/Linux, with **Linux 5.6.0eudyptula-12662-gf788bf746653**' --class debian ...  
*--class os $menuentry_id_option*  
**'gnulinux-5.6.0eudyptula-12662-gf788bf746653-advanced-ef962dd6-4592-467e-88d8-a7a283e6c483'**  

menuentry 'Debian GNU/Linux, with Linux 4.19.0-8-amd64' --class debian ...  
*--class os $menuentry_id_option*  
'gnulinux-4.19.0-8-amd64-advanced-ef962dd6-4592-467e-88d8-a7a283e6c483'  

---

Enter the submenu id and the menuentry id, separated by a `>`  into  
the GRUB_DEFAULT option, within the /etc/default/grub file:  
`GRUB_DEFAULT="gnulinux-advanced-ef962dd6-4592-46>gnulinux-5.6.0eudyptula-12662-gf788bf746653-advanced-ef962dd6-4592-467e-88d8-a7a283e6c483"`  

And update the grub config with: `sudo update-grub`  
[\[2\]](https://unix.stackexchange.com/a/327686/402744)  

## Patch  
#### Creating the patch out of a commit
* `git commit -s`
* write the commit message
* `git format-patch HEAD~`
* `./scripts/checkpatch.pl {the patch}`
**sending**  
* `git send-email\`   
   `--cc-cmd='./scripts/get_maintainer.pl\`  
   `--norolestats {the patch}\`    
   `--cc persona@kernel.com,personb@kernel.com`\  
   `{the patch}`  
* Optional: change the subject prefix of your email title with: `--subject-prefix="PATCH-for-branch-x"` in order to change: "[PATCH] my title",to: "[PATCH-for-branch-x] my title".

#### Great tutorials:  
[\[3\]](https://nickdesaulniers.github.io/blog/2017/05/16/submitting-your-first-patch-to-the-linux-kernel-and-responding-to-feedback/)
Nick Desaulniers (2017)  
[\[4\]](https://kernelnewbies.org/FirstKernelPatch)
Vaishali Thakkar (last update 2020)  

#### What are important attributes of a patch?
* It has to be signed
* It needs a good commit header and a proper change description
* CC <linux-kernel@vger.kernel.org> and the correct maintainer, as well as people that have recently worked on the same files
* only PLAIN text, nothing else
* Each patch fixes a single problem

#### Checking the patch for errors
./scripts/checkpatch.pl {the patch} (analysis the patch file for obvious errors)  

Using sparse (a static code analyser for the linux kernel):  
`sudo apt-get install sparse`(Debian/Ubuntu)    

`make C=2 path/to/sub-system`  
Example: `make C=2 drivers/staging/octeon`  
[\[5\]](https://kernelnewbies.org/Sparse)  

#### How to add a patch from mutt to git?

You have to use `git am -s` with the mbox file.
To automate that step within mutt, I use two macros  
(within the mutt rc or in account specific configurations):  
`macro index L '|git am -s'\n`  
`macro index <Esc>L '|git am -s -3'\n`  

*How does this work?*  
The first creates a macro for the key 'L' to pipe the message into git am -s, the second is triggered with an additional ESC and includes a `-3`, which is the git am option for a 3 way merge.  

My workflow here is:  
* change directory into the kernel source directory
* open mutt
* go to the PATCH mail
* hit 'L'

I use the first option as my primary choice and the second option only, when the change is small so that I don't end up with big nasty merge conflicts.  

*What is a 3 way merge?*  
Instead of a 2 way merge, which is basically a difference between two files, a 3 way merge compares both files with each other, while additionally comparing both with their common ancestor.   

If you want to go deeper into mutt configuration, look at this fantastic article from Greg KH: [\[6\] Email Workflow from 2019](http://kroah.com/log/blog/2019/08/14/patch-workflow-with-mutt-2019/)  


## Reference  
[\[1\]](https://stackoverflow.com/a/25091076/9918329) *Stackoverflow answer about the dirty tag*   
[\[2\]](https://unix.stackexchange.com/a/327686/402744) *StackExchange answer about setting the default kernel in grub*  
[\[3\]](https://nickdesaulniers.github.io/blog/2017/05/16/submitting-your-first-patch-to-the-linux-kernel-and-responding-to-feedback/)
 *Nick Desaulniers about writing patches to the kernel*  
[\[4\]](https://kernelnewbies.org/FirstKernelPatch) *Vaishali Thakkar introduction to kernel patches on kernelnewbies.org*   
[\[5\]](https://kernelnewbies.org/Sparse) *Bo Yu tutorial sparse static code analysis*   
[\[6\]](http://kroah.com/log/blog/2019/08/14/patch-workflow-with-mutt-2019/) *Greg Kroah Hartman about his Linux Kernel email workflow*  
