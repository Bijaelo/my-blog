---
title: debugging compiled R code (Rcpp code) on a windows computer
author: Oliver P. Madsen
date: '2020-12-26'
slug: Rcpp-debugging-2020-12-26
categories: [R, Rcpp, debugging]
tags: ["R", "Rcpp", "debug", "debugging", "compiled code"]
keywords:
  - Rcpp
  - compiled code
  - debugging
---
# Debugging code is essential
This is a phrase that every programmer knows. And those who has reached "advanced" status in their R adventure will know, that debugging R code does by printing our variables is not an efficient method for finding and solving problems. But once we get a little further and start developing compiled code (Rcpp code) we hit a new wall.

<!-- mode -->
Almost every online reference is targeted MacOS or Linux users when it comes to debugging compiled code, and none are user-friendly in their methodology and provide a step by step guide for windows users to reach the infamous debugging environment. Frankly several references just blankly state that one should move to a Linux distribution or use standard printing methods. 

# What I'll be covering in my topic today
In this blog post I'll be covering my journey to find a proper method for debugging compiled code (Rcpp code) that one would generate using R. It is not a finished blog post and will be updated with more information as I find more references and try out different things to see what works and what doesn't. At the moment I have not obtained a working solution but my hopes and dreams are, that by the end of this blog post I'll have described not only 1 but 3 or 4 different methods for debugging C++. 

## A quick look back at how one can debug standard R code and what we'll be looking for
Before we continue, I think it is important just to get an idea what we'd be looking for as our result. So lets just review what we know from base R debugging. In base R we have the following options for debugging:

1. We have the `debug` function, which can be used to start the debugger upon activation of a function through a direct or indirect call, and `undebug` to remove remove the debugger. We also have the `debugonce` for debugging directly. 
1. Next we have `browser` which we can use to insert a manual browser point into any function, and the debugger will activate once any code reached the call.
1. `trace` is another important function, that can be used to edit and add code to the body of an active function. For example a call to `trace(lm, edit = TRUE)` would allow us to insert a call to `browser` at a point where we believe our error is happening (or after a point where we are certain no errors happened prior). Like `undebug` we have `untrace` to revert the effect.
1. `traceback` is along the lines of `trace` and lets us see the call stack that happened up to a specific error occuring (allowing us to see for example which call to debug).
1. `recover` can be used to change the way errors are handled. If we for example execute `oldOpt <- options(error = recover);  myFit <- lm(y ~ x, data = xy, weights = w); options(oldOpt);` an option menu pops up that lets us choose a specific call to debug.
1. `dump.frames` is an option (that in my experience) is rarely used, but allows one to dump an object (similar to the output from `recover`) into a specific variable or a file, allowing us to continue executing additional code or come back and debug the specific call at a later point (or even send the object over to a help desk and let them do our job for us). `dump.frames` is barely mention by any guides outside the core vignette [Writing R extensions](https://cran.r-project.org/doc/manuals/r-release/R-exts.pdf) but has some useful cases when debugging larger projects.

Once we're in the debugger we have standard movements using `n` for **next** (to move one call forward), `s` for **step** (stepping into the next executed call), `c` for **continue** (continue until next call to `browser` or the end of current call) and `Q` for **quit** which terminates the call stack and destroys the associated environemnt. Additionally we can look at any variable and execute any code. Thereby we can see the value of all variables stored during a call. 

Lastly there is of course a last method for debugging, often used when someone is just starting their adventure in any programming language. Editing calls (maybe one has just discovered `trace(..., edit = TRUE)`) and adding calls to `print` or `cat` to print out the values of our variables. Lets call this the beginners debugging methodology and no shame, honestly we have all been there for the first 2 weeks - 6 months of our programming life.

Using Rstudio (or alternative gui like visual studio code) we further get a nice line-by-line view of the call. 

When we are looking for debugging compiled code, we are looking for the same options so in short:
1. An ability to set a breakpoint within our code (similar to `browser` or `trace`).
1. Options to move through our call stack, move into child function calls and destroy our call stack using something simmilar to `n`, `s`, `c` and `q`.
1. If plausible do our debugging in an interactive view like the Rstudio gui or visual studio code.

# available ressources on debugging compiled code
When it comes to debugging compiled code, all sources reference the [Writing R extensions](https://cran.r-project.org/doc/manuals/r-release/R-exts.pdf) (or reference a source that references [Writing R extensions](https://cran.r-project.org/doc/manuals/r-release/R-exts.pdf)). The main sections about compiled code in the manual is 1.2, 1.5.4, 1.6.4, 3.4, 4.4 and the entire 5'th and 6'th chapter. This is a lot, but the debugging sections are primarily in chapter 4, while the other contain interesting code.

It has been a day since my brain last had a chance to melt when it comes to this problem. Getting back to it, I have now come a little bit closer. 

Below are the current ressources I have and will be using:

* https://r-pkgs.org/src.html#src-debugging
* https://github.com/renkun-ken/vscode-rcpp-demo/blob/master/.vscode/c_cpp_properties.json
* https://gist.github.com/kongdd/28cbc006cba24beb50352d30055e3822
* https://stackoverflow.com/questions/53622354/how-to-debug-line-by-line-rcpp-generated-code-in-windows
* https://blog.davisvaughan.com/2019/04/05/debug-r-package-with-cpp/
* https://github.com/aryoda/R_CppDebugHelper
* https://www.brodieg.com/2018/04/06/adventures-in-r-and-compiled-code/
* https://magic-lantern.github.io/2018/10/05/2018-10-05-how-to-profile-your-r-code-that-calls-c-c-plus-plus/
* http://evolvedmicrobe.com/blogs/?p=359

Really promising references:

* https://code.visualstudio.com/docs/cpp/config-mingw
* http://www.mingw.org/wiki/sampledll

So my current strategy is to find a way to:

1. Debug the code using and R and C++ plugin to Visual Studio Code,
1. Try to get gdb working to get debugging working through the command prompt. First I'll do this with some code I've compiled myself, and then I will try to replicate it with the exact same script, but going through the Rcpp magic filter
1. Then I'll try to see if I can get this working while executing the code from either the R-gui or Rstudio gui. This is what It'd call the "optimal" solution, so we could step through the problem line by line in our most frequently used environemnt
1. And last I want to simply do this by building our library through a linux docker image (rocker/rstudio or similar) and try again to get it working through the commandline and then the Rstudio gui. This should (in theory) be the simplest of them all, as we'd just be using the builtin -d flag in the R executable. But again I am expecting to have some minor hickups along the way. 

I am documenting the procedure as I'm going on and once I've figured which methods work and which I simply cannot get working I'll make a post with with an organized guide for debugging Rcpp on windows.

## debugging with gdb and R.exe
This was the first thing I tried, and it looked promising at first, primarily based on a stackoverflow post about debugging, a blog post about profiling and one about debugging. The only catch was they were all talking about it from the perspective of a mac user, but two of them stating that they believe it should work for windows users. 

My methodology:
1. Install gdb.
   * Download the newest version of gdb (I used gdb 10.1 as of 2020-12-26)
       * Either do this on their website, or from the command prompt type  
       `curl https://ftp.gnu.org/gnu/gdb/gdb-{vers-nr}.tar.gz -o {folder-path}/gdb-{vers-nr}.tar.gz`  
       while replacing `folder-path` and `vers-nr` with the folder you want to download to and the version to download.
   * Unzip the folder. This is simplest from the command prompt by typing `tar -xzvf {folder-path}/gdb-{vers-nr}.tar.gz`. Here `-x` is for extract, `-z` is to indicate it is a zip file, `-v` is for verbose and `-f` is for filename (and can be left out). See `tar --help` for more info
1. If you want to perform this multiple times, I suggest adding `R.home("bin")` and the `{folder-path}/gdb-{vers-nr}/gdb` to your [system environment path variable](https://www.architectryan.com/2018/03/17/add-to-the-path-on-windows-10/). You are likely going to debug multiple times, and this lets you easily execute the things you need. Maybe you should add folder to Rgui.exe and Rstudio.exe as well, in case I get debugging from either working.
1. Create a script **with-a-bug** (example below this list).
1. (maybe set debug flag in Makevars/makeconf/Makevars.win? It didnt seem to have an effect, but I might have to set it in the dll flags. Still testing, will extend if I figure this out.)
1. From the commandline execute 
   1. `gdb R.exe`. This starts up the gdb environment and lets it know that it needs to lookout for errors while running R.exe.
   1. `breakpoint {name of function with known error} -y`. This sets a flag that if an error occurs in the function, the debugger should activate (in the gdb window, not Rgui or R-console!)
   1. `run`, which will start up the R from console (if we had run `gdb Rgui.exe` it would start the Rgui.)
   1. Execute the bugged script. 
   1. Now here we should either be able to debug the code from R itself or maybe using `ctrl+D` to exit to gdb and debug from here. But up till now I haven't been able to replicate the debugger activating.

That should be the process. One we are at the debugger we should be able to use our standard commands and do our thing, but right now it is not working.


## Debugging with Visual Studio Code and Mingw
This is still early in the process. And my C++ experience is till 6 years old (with a break that is of equal length), so I am a bit out of my depths still. 

For this I'll start by creating a commandline script (that is independent of R and Rcpp). Afterwards I'll find the specific flags used by R and try to manually generate a script (old-style) without using R CMD shlib (R CMD shared-library). This **is** possible, and once I've figured it out I will try to access the function from R. This is basically what I expect to be the "old-fashioned" way of generating compiled libraries for R. I might have to rewrite the script slightly here, to accomodate the R `SEXP` class and use **Rcpp.h** for inputs. But I'll figure this out as I go. Last, once I've figured the flags, been able to connect to R, I should have the knowledge to alter makevars/makeconf/makevars.win to set the flags that are used when creating the script from R. And at this point we should be able to breakpoints in visual studio code, execute it from the command line and just step through the function as we would normally do.

## Debugging in a Docker environment
This should in theory be the simplest method of them all. Set up an account for Docker and download Docker Desktop. Log in from the desktop and download the image. Use either the GUI or command prompt to download and start an appropriate docker image. Enter it through the command prompt and execute `R -d gdb -e "script/function"` and go through the debugger as you would expect. 