---
layout: post
title:  "Implicit int and Pointers"
tags: c
---

So here is an interesting problem related to C that can seem completely
innocuous but can blow up in your face.  This is something I see happen a lot
with students who are used to Java but not C.  For this example, I am going to
focus on a x86-64 Linux system.  Let's say you have a function that returns a
pointer and is defined in one file and used in another.  Specifically, you have
the following three files:

**my_struct.h:**
{% highlight c %}
#ifndef _MY_STRUCT_H_
#define _MY_STRUCT_H_

struct my_struct {
  int foo;
  int bar;
};

#endif // _MY_STRUCT_H_

{% endhighlight %}

**main.c:**
{% highlight c %}
#include <stdio.h>
#include "my_struct.h"

int main(void)
{
  struct my_struct *instance = alloc_my_struct(5, 10);

  printf("instance in %s = %p\n", __func__, (void *)instance);

  // rest of code ...

  instance->foo = 10;

  return 0;
}
{% endhighlight %}


**my_struct.c:**
{% highlight c %}
#include <stdio.h>
#include <stdlib.h>
#include "my_struct.h"

struct my_struct*
alloc_my_struct(int foo, int bar)
{
  struct my_struct *instance = malloc(sizeof(*instance));
  if(instance != NULL) {
    instance->foo = foo;
    instance->bar = bar;
  }
  printf("instance in %s = %p\n", __func__, (void *)instance);
  return instance;
}

{% endhighlight %}

I should preface this with I generally perfer a pattern more like:

{% highlight c %}
int init_my_struct(struct my_sturct *instance, int foo, int bar)
{% endhighlight %}

with the return value indicating success or failure, if appropriate.  This way
the memory for the struct can be allocated statically, dynamically or
automatically but lets continue with the previous example.

So when you compile `main.c` you get a warning from gcc (version 4.9.2):

    main.c:7:32: warning: initialization makes pointer from integer without a cast
       struct my_struct *instance = alloc_my_struct(5, 10);

but when you run the executable the output shows the same value for the pointer
so everything must be good (of course it isn't otherwise this would be boring).

    instance in alloc_my_struct = 0x23a5010
    instance in main = 0x23a5010

Now lets say you modify `struct my_struct` which results in its size being
increased:

**my_struct.h:**
{% highlight c %}
#ifndef _MY_STRUCT_H_
#define _MY_STRUCT_H_

struct my_struct {
  int foo;
  int bar;
  char large_buffer[0x1000000];
};

#endif // _MY_STRUCT_H_

{% endhighlight %}

Now because the size of `struct my_struct` has increased when you run the simple
test program you get something like:

    instance in alloc_my_struct = 0x7fdff615f010
    instance in main = 0xfffffffff615f010
    Segmentation fault (core dumped)

or something like:

    instance in alloc_my_struct = 0x7f8d35e55010
    instance in main = 0x35e55010
    Segmentation fault (core dumped)

#What went wrong

I'm changing the struct size so the address returned by `malloc` is higher which
is the key thing that has changed.  So the question is *what happened here?*
Well it probably had to do with that **initialization makes pointer from integer
without a cast** warning and that warning happened because we did not provide a
function prototype.  Specifically, you can get one more warning if you compile
with `-Wall`.

    main.c:7:10: warning: implicit declaration of function ‘alloc_my_struct’ [-Wimplicit-function-declaration]
       struct my_struct *instance = alloc_my_struct(5, 10);

    main.c:7:32: warning: initialization makes pointer from integer without a cast
       struct my_struct *instance = alloc_my_struct(5, 10);

So lets add a function prototype to `my_struct.h`:
{% highlight c %}
#ifndef _MY_STRUCT_H_
#define _MY_STRUCT_H_

struct my_struct {
  int foo;
  int bar;
};

struct my_struct *alloc_my_struct(int foo, int bar);

#endif // _MY_STRUCT_H_

{% endhighlight %}

Now you don't get any compiler warnings and the program output is something like
the following:

    instance in alloc_my_struct = 0x7f83c47a3010
    instance in main = 0x7f83c47a3010


#Looking at some assembly

So all good again.  But what exactly is happening when we include the function
declaration?  Well as the warning states, there is a conversion from an
integer to a pointer.  But where is the integer?  As the name of this post
implies, the integer is **implicit**.  When gcc encounters a function invocation
that it has not seen the declaration for it assumes that the function has a
variable number of arguments and that the return type is an `int`.  So lets look
at the relevant snippet of assembly to see the exact difference when the
function prototype is included or not. To do this we will use the `-S` flag of
gcc that outputs the assembly.

**With the function prototype:**
{% highlight asm %}
subq $16, %rsp
movl $10, %esi
movl $5, %edi
call alloc_my_struct
movq %rax, -8(%rbp)
movq -8(%rbp), %rax
movq %rax, %rdx
{% endhighlight %}

**Without the function prototype:**
{% highlight asm %}
subq $16, %rsp
movl $10, %esi
movl $5, %edi
movl $0, %eax
call alloc_my_struct
cltq
movq %rax, -8(%rbp)
movq -8(%rbp), %rax
movq %rax, %rdx
{% endhighlight %}

So the difference is when the function prototype is not included you get a `movl
$0, %eax` before the `call` instruction and you get a `cltq` after.  The `movl
$0 %eax` isn't relavent to this discussion so you can find the reason behind it
at the end of this post.  The interesting part is the `cltq` instruction.
`ctlq` (which is the AT&T name for the `cdqe` instruction) sign extends the
dword in `%eax` to a qword, storing the result in `%rax`.  Since gcc is assuming
the return value is a 32 bit integer it is sign extending it to fit within the
64 bit pointer variable.  This is in fact the problem.

If you look at the case where the pointer did not change, the value returned by
`malloc` has bits 31 to 63 set to 0.  Therefore the sign extension didn't change
`%rax`.  In the two cases where the pointer changed, bits 32 to 63 are set to
all 0's or all 1's depending on the value of bit 31.

#Final Thoughts

So what have we learned.  Well, the obvious is don't ignore warnings, but also
that something so simple as a C function return value can work sometimes
fail other times when you ignore them.  The code for this post can be found
[here](https://github.com/missimer/implicit-int-and-pointers).

#Setting %eax to 0 before the function call

The `movl $0, %eax` before the `call` is due to the fact that gcc has assumed
`alloc_my_struct` takes a variable length number of arguments.  This is again
because the prototype has not been specified.  The AMD64 calling convention
specifies that the number of vector arguments in the variable length argument
list is passed via register `%rax`.  Vector arguments are arguments that use the
SEE or AVX (Advanced Vector Extensions) registers.  In this case we have just
two integers passed via the general purpose registers so the value is 0.
