---
layout: post
title:  "Passing Command Line Arguments to a Module"
date:   2016-10-21 17:14:54
categories: File Systems
tags: FS module
---
# Passing Command Line Arguments to a Module

Modules can take command line arguments, but not with the argc/argv you might be used to.

To allow arguments to be passed to your module, declare the variables that will take the values of the command line arguments as global and then use the MODULE_PARM() macro, (defined in linux/module.h) to set the mechanism up. At runtime, insmod will fill the variables with any command line arguments that are given, like **./insmod mymodule.o myvariable=5**. The variable declarations and macros should be placed at the beginning of the module for clarity. The example code should clear up my admittedly lousy explanation.

The MODULE_PARM() macro takes 2 arguments: the name of the variable and its type. The supported variable types are "b": single byte, "h": short int, "i": integer, "l": long int and "s": string, and the integer types can be signed as usual or unsigned. Strings should be declared as "char *" and insmod will allocate memory for them. You should always try to give the variables an initial default value. This is kernel code, and you should program defensively. For example:

```
    int myint = 3;
    char *mystr;

    MODULE_PARM(myint, "i");
    MODULE_PARM(mystr, "s");
	
```

Arrays are supported too. An integer value preceding the type in MODULE_PARM will indicate an array of some maximum length. Two numbers separated by a '-' will give the minimum and maximum number of values. For example, an array of shorts with at least 2 and no more than 4 values could be declared as:

```
    int myshortArray[4];
    MODULE_PARM (myintArray, "3-9i");
	
```

A good use for this is to have the module variable's default values set, like an port or IO address. If the variables contain the default values, then perform autodetection (explained elsewhere). Otherwise, keep the current value. This will be made clear later on.

Lastly, there's a macro function, MODULE_PARM_DESC(), that is used to document arguments that the module can take. It takes two parameters: a variable name and a free form string describing that variable.

**Example 2-7. hello-5.c**

```
/*  hello-5.c - Demonstrates command line argument passing to a module.
 *
 *  Copyright (C) 2001 by Peter Jay Salzman
 *
 *  08/02/2006 - Updated by Rodrigo Rubira Branco <rodrigo@kernelhacking.com>
 */

/* Kernel Programming */
#define MODULE
#define LINUX
#define __KERNEL__

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Peter Jay Salzman");

// These global variables can be set with command line arguments when you insmod
// the module in. 
//
static u8             mybyte = 'A';
static unsigned short myshort = 1;
static int            myint = 20;
static long           mylong = 9999;
static char           *mystring = "blah";
static int            myintArray[2] = { 0, 420 };

/*  Now we're actually setting the mechanism up -- making the variables command
 *  line arguments rather than just a bunch of global variables.
 */
MODULE_PARM(mybyte, "b");
MODULE_PARM(myshort, "h");
MODULE_PARM(myint, "i");
MODULE_PARM(mylong, "l");
MODULE_PARM(mystring, "s");
MODULE_PARM(myintArray, "1-2i");

MODULE_PARM_DESC(mybyte, "This byte really does nothing at all.");
MODULE_PARM_DESC(myshort, "This short is *extremely* important.");
// You get the picture.  Always use a MODULE_PARM_DESC() for each MODULE_PARM().


static int __init hello_5_init(void)
{
   printk(KERN_ALERT "mybyte is an 8 bit integer: %i\n", mybyte);
   printk(KERN_ALERT "myshort is a short integer: %hi\n", myshort);
   printk(KERN_ALERT "myint is an integer: %i\n", myint);
   printk(KERN_ALERT "mylong is a long integer: %li\n", mylong);
   printk(KERN_ALERT "mystring is a string: %s\n", mystring);
   printk(KERN_ALERT "myintArray is %i and %i\n", myintArray[0], myintArray[1]);
   return 0;
}


static void __exit hello_5_exit(void)
{
   printk(KERN_ALERT "Goodbye, world 5\n");
}


module_init(hello_5_init);
module_exit(hello_5_exit);
```

I would recommend playing around with this code:

```
    satan# insmod hello-5.o mystring="bebop" mybyte=255 myintArray=-1
    mybyte is an 8 bit integer: 255
    myshort is a short integer: 1
    myint is an integer: 20
    mylong is a long integer: 9999
    mystring is a string: bebop
    myintArray is -1 and 420
    
    satan# rmmod hello-5
    Goodbye, world 5
    
    satan# insmod hello-5.o mystring="supercalifragilisticexpialidocious" \
    > mybyte=256 myintArray=-1,-1
    mybyte is an 8 bit integer: 0
    myshort is a short integer: 1
    myint is an integer: 20
    mylong is a long integer: 9999
    mystring is a string: supercalifragilisticexpialidocious
    myintArray is -1 and -1
    
    satan# rmmod hello-5
    Goodbye, world 5
    
    satan# insmod hello-5.o mylong=hello
    hello-5.o: invalid argument syntax for mylong: 'h'
```