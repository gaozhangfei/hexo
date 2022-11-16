---
title: cflow
date: 2022-11-16 10:44:45
tags: tools
description: use cflow to check calling sequence
---

We can use cflow to generate calling sequence
```
sudo apt-get install cflow

git clone https://gitee.com/ethan-net/linux-0.11-lab.git
cp linux-0.11-lab/tools/tree2dotx ~/bin/
chmod +x tree2dotx

sudo apt-get install xdot
```

Check one file
```
cd uadk
cflow wd.c | tree2dotx > out.dot ; xdot out.dot
```
Check all file
```
cd uadk
cflow -d 2 *.c  | tree2dotx > out.dot; xdot out.dot
```
-d depth can be used to change the complication, 
For example -d 2 is simple, -d 5 is complicated
