# It's not just about the 'what', it's also about the 'how'

This challenge is about the little things that make the kernel code base feel less like different parts stuck together and more like a unified project with a clear style agenda.  

I go through the following steps: the **Essence** of the coding style guidelines, **Configuration** of vim to comply with those standards & **Modification** of the example.  

If you are interested in the journey so far, checkout:  
[1. Hello world module](https://sebastianfricke.me/eudyptula-challenge-Part1/)  
[2. Building the kernel](https://sebastianfricke.me/eudyptula-challenge-Part-2/)  
[3. Modify & Patch the kernel](https://sebastianfricke.me/eudyptula-challenge-Part-3/)  

## Task  

Wonderful job in making it this far. I hope you have been having fun. Oh, you're getting bored, just booting and installing kernels? Well, time for some pedantic things to make you feel that those kernel builds are actually fun!

Part of the job of being a kernel developer is recognizing the proper Linux kernel coding style. The full description of this coding style can be found in the kernel itself, in the Documentation/CodingStyle file. I'd recommend going and reading that right now. It's pretty simple stuff, and something that you are going to need to know and understand. There is also a tool in the kernel source tree in the scripts/ directory called checkpatch.pl that can be used to test for adhering to the coding style rules, as kernel programmers are lazy and prefer to let scripts do their work for them...

Why a coding standard at all? Because of your brain (yes, yours, not mine, remember, I'm just some dumb shell scripts). Once your brain learns the patterns, the information contained really starts to sink in better. So it's important that everyone follow the same standard so that the patterns become consistent. In other words, you want to make it really easy for other people to find the bugs in your code, and not be confused and distracted by the fact that you happen to prefer 5 spaces instead of tabs for indentation. Of course you would never prefer such a thing, I'd never accuse you of that, it was just an example, please forgive my impertinence!

Anyway, the tasks for this round all deal with the Linux kernel coding style. Attached to this message are is one kernel module that does not follow the proper Linux kernel coding style rules. Fix this file up, AND fix up the final submission you did for Task 01, and send them back to me as attachments in your response email.

Yes, the logic in attached second module is crazy, and probably wrong, but don't focus on that, just look at the patterns here, and fix up the coding style, do not remove lines of code.

Oh, and before you think "Ah, but I got the coding style right for Task 01, I already know this stuff!", remember that so far only 10 people, out of over 4000, have gotten the coding style exactly right for their Task 01 module. Yes, you could be one of those people, but the odds are not in your favor. You should look at it again just to be sure.

## Essence  

### Visibility & Readability
* 8-char tabs (e.g. 8-char indentations)
* Maximum line length 80-chars (exceed only for readability)
* No more than 3 levels of indentation
* One statement/assignment/data declaration per line
* Functions should do one thing only, with a max. 5-10 local variables
* Use goto with descriptive labels for a centralized exit and clean up but not for simple returns
* Short comments on top of functions that explain **why** and **what** and **not** *how*
* Constant macros and enum labels are all-caps
* Avoid conditional compilation #ifdef ... in *.c files, limit as much as possible to *.h files

### General style-rules
* The labels of a switch statement should be aligned to the same column as the switch itself
* No white-space at the end of a line
* Never break printk statements onto 2 lines (so that it can be found with grep)
* Set white-space after if, else, while, for, do, switch but not after sizeof, typeof, alignof, \_\_attribute\_\_, etc
* No spaces around parenthesized expressions
* Place spaces around [binary](https://en.wikipedia.org/wiki/Binary_operation) & [ternary](https://en.wikipedia.org/wiki/Ternary_operation) operators but not after a [unary](https://en.wikipedia.org/wiki/Unary_operation) operator
* No spaces around structure member operators
* Use enums for multiple constant definitions
* Opening curly braces are placed at the end of a statement with a preceding white-space, except for a function where they are placed on a new line with a subsequent line-break
* Closing curly braces are placed on an empty line except for the 'while' in a do-while or 'else' in an if-else
* Only use curly-braces for conditional statements if necessary
* No mixed case names, global names must be descriptive, no type within name, short local variables
* Do not use typedefs unless for opaque objects, on variables with different types across configurations, new types for type-checking, types safe for user-space

### Rules to avoid Bugs
* Function prototypes have to include the types and should not use the extern keyword
* Data structures that are visible in multi-threaded environments need reference counting
* Prefer inline functions over macros resembling functions
* Macros should not depend on local variables or change the control flow
* Do not use macros for l-values
* Take care of namespace collisions with macros
* Make sure printk messages match to the correct device
* Use sizeof(*p) instead of the spelled-out struct name in mallocs
* In most cases do not use inline on functions with more than 3 lines of code
* If a function name implies an action/command use an error-code-integer (0 for success) if it is a predicate use "boolean" (1 for success)
* Only use inline assembly if C cannot do the job, If there is a lot of inline-assembly put into a *.S file

## Configuration  

**Vim** is my editor and as it is highly configurable, I want to configure it to respect most of these rules on its own. I discovered that this is a solved problem: [Vim Plugin](https://github.com/vivien/vim-linux-coding-stylehttps://github.com/vivien/vim-linux-coding-style) by Vivien Didelot, cares about the tab size, the trailing white-spaces, the maximum line length and even adds additional kernel specific syntax keywords.

## Modification  
The file from the challenge can be found here: [challenge description from Github](https://github.com/sinedoke/eudyptula#this-is-task-04-of-the-eudyptula-challenge).

Just try to apply the rules mentioned in the coding style guidelines. As soon as you feel like the code looks good (which is not that easy, when you are not allowed to remove lines), commit the file (within any repository) and use `git format-patch HEAD~` to create a patch out of the changes. This enables you to run ./scripts/checkpatch.pl (from the kernel root dir), to see if you missed some problems.

I counted a total of 39 single changes until all coding style errors were resolved. Try it out yourself!

## Reference  
[\[1. unary ](https://en.wikipedia.org/wiki/Unary_operation)[binary ](https://en.wikipedia.org/wiki/Binary_operation)[ternary\]](https://en.wikipedia.org/wiki/Ternary_operation) *Wikipedia description of different types operators*  
[\[2. Vim Plugin\]](https://github.com/vivien/vim-linux-coding-stylehttps://github.com/vivien/vim-linux-coding-style) *Vim Plugin for linux kernel coding guidelines by Vivien Didelot*  
[\[3. Example repo\]](https://github.com/sinedoke/eudyptula#this-is-task-04-of-the-eudyptula-challenge) *Repository of eudyptula challenges from Sinedoke*  
