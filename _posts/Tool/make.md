---
title: make
layout: post
categories: Tool
---

* `:=`: It is called “simply expanded” because its righthand side is expanded immediately upon reading the line from the makefile.
* `=`: It is called “recursively expanded” because its righthand side is simply slurped up by make and stored as the value of the variable without evaluating or expanding it in any way. Instead, the expansion is performed when the variable is used.
* `?=`: The ?= operator is called the conditional variable assignment operator.
* `define`: remember that macros are just variables where embedded newlines are allowed
* An assignment of a variable on the command line overrides any value from the environment and any assignment in the makefile. 
* `include`: After that, make searches a compiled search path simi- lar to: /usr/local/include, /usr/gnu/include, /usr/include.
* `-include`: If you want make to ignore include files it cannot load, add a leading dash to the include directive.
* `$(function-name arg1[,argn])`: function call
* `$(call macro-name[, param1...])`: call user-defined function
* `commands`: Lines beginning with a tab character are commands that will be executed by a subshell.
* `@`: 开头的 command 不会打印出来
* `-`: 忽略 command 错误
