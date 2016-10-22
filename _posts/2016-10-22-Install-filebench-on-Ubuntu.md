---
slayout: post
title:  "Passing Command Line Arguments to a Module"
date:   2016-10-21 17:14:54
categories: File Systems
tags: FS module

---

# Install filebench on Ubuntu 14.04

1. First download tarball from [here](https://github.com/filebench/filebench/releases)

2. unzip it by using tar -xvf 

3. Enter into the folder, (e.g.,/home/dzhou/Downloads/filebench-1.5-alpha1)

4. `sudo apt-get install flex` to pervent any error

5. ```bash
   libtoolize

   aclocal

   autoheader

   automake --add-missing

   autoconf

   ```

6. ```bash
   ./configure
   make
   sudo make install
   ```

7. try `filebench` 

