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

## FIX ME!!! -> This section is very much incomplete

Other references:
* https://r-pkgs.org/src.html#src-debugging
* https://github.com/renkun-ken/vscode-rcpp-demo/blob/master/.vscode/c_cpp_properties.json
* https://gist.github.com/kongdd/28cbc006cba24beb50352d30055e3822
* https://stackoverflow.com/questions/53622354/how-to-debug-line-by-line-rcpp-generated-code-in-windows
* https://blog.davisvaughan.com/2019/04/05/debug-r-package-with-cpp/
* https://github.com/aryoda/R_CppDebugHelper
* https://www.brodieg.com/2018/04/06/adventures-in-r-and-compiled-code/
