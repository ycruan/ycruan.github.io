---
title: "[OS] Operating Systems and Systems Programming"
excerpt: "<i>This is a collection of course projects and homework for Berkeley CS162: Operating Systems and Systems Programming (Term: 17 Spring).
</i><br/><br/><img src='/images/projects_162_os.png' height='300' width='500'>"
collection: portfolio
---

Operating Systems and Systems Programming
======

This is a collection of course projects and homework for Berkeley CS162: Operating Systems and Systems Programming (Term: 17 Spring).
* Project 1: Threads ([spec](https://ycruan.github.io/files/162_project1_spec.pdf), [design doc](https://ycruan.github.io/files/162_project1_design.md))
  * Design and implement an alarm clock without using busy waiting;
  * and a priority scheduler that supports priority donation;
  * and a Multi-level Feedback Queue Scheduler (MLFQS) that schedule threads as per their CPU usage history.

* Project 2: User Programs ([spec](https://ycruan.github.io/files/162_project2_spec.pdf), [design doc](https://ycruan.github.io/files/162_project2_design.md))
  * Enable argument passing to user process.
  * Design and implement process control syscalls: `halt` (shutdown process), `exec` (start new process), `wait` (wait for child to exit);
  * and file system syscalls: `create`, `remove`, `open`, `filesize`, `read`, `write`, `seek`, `tell`, `close`.

* Project 3: File Systems ([spec](https://ycruan.github.io/files/162_project3_spec.pdf), [design doc](https://ycruan.github.io/files/162_project3_design.md))
  * Design and implement a buffer cache for most recent used pages;
  * and a extensible file system with subdirectories;
  * and more file system syscalls: `chdir`, `mkdir`, `readdir`, and `isdir` that support both absolute and relative path.

* Homework 1: Shell ([spec](https://ycruan.github.io/files/162_homework1_spec.pdf))
  * Build a shell that supports path resolution, program exec, io redirect, signal handling, and foreground/background switching.

* Homework 2: HTTP Server ([spec](https://ycruan.github.io/files/162_homework2_spec.pdf))
  * Build a server that receives, processes and responds to HTTP GET requests.

* Homework 3: Malloc ([spec](https://ycruan.github.io/files/162_homework3_spec.pdf))
  * Implements `malloc`, `free` and `realloc`.

Technical tools: C, GNU toolchain