<!-- Beej's guide to C

# vim: ts=4:sw=4:nosi:et:tw=72
-->

# Multithreading

C11 introduced, formally, multithreading to the C language. It's very
eerily similar to [flw[POSIX threads|POSIX_Threads]], if you've ever
used those.

And if you're not, no worries. We'll talk it through.

Do note, however, that I'm not intending this to be a full-blown classic
multithreading how-to^[I'm more a fan of shared-nothing, myself, and my
skills with classic multithreading constructs are rusty, to say the
least.]; you'll have to pick up a different very thick book for that,
specifically. Sorry!

## Background

Threads are a way to have all those shiny CPU cores you paid for do work
for you in the same program.

Normally, a C program just runs on a single CPU core. But if you know
how to split up the work, you can give pieces of it to a number of
threads and have them do the work simultaneously.

Though the spec doesn't say it, on your system it's very likely that C
(or the OS at its behest) will attempt to balance the threads over all
your CPU cores.

And if you have more threads than cores, that's OK. You just won't
realize all those gains if they're all trying to compete for CPU time.

## Things You Can Do

You can create a thread. It will begin running the function you specify.
The parent thread that spawned it will also continue to run.

And you can wait for the thread to complete. This is called _joining_.

Or if you don't care when the thread completes and don't want to wait,
you can _detach it_.

A thread can explicitly _exit_, or it can implicitly call it quits by
returning from it's main function.

A thread can also _sleep_ for a period of time, doing nothing while
other threads run.

The `main()` program is a thread, as well.

Additionally, we have thread local storage, mutexes, and conditional
variables. But more on those later. Let's just look at the basics for
now.

## Creating and Waiting for Threads

Let's hack something up!

We'll make some threads (create) and wait for them to complete (join).

We have a tiny bit to understand first, though.

Every single thread is identified by an opaque variable of type
`thrd_t`. It's a unique identifier per thread in your program. When you
create a thread, it's given a new ID.

Also when you make the thread, you have to give it a pointer to a
function to run, and a pointer to an argument to pass to it (or `NULL`
if you don't have anything to pass).

The thread will begin execution on the function you specify.

When you want to wait for a thread to complete, you have to specify it's
thread ID so C knows which one to wait for.

So the basic idea is:

1. Write a function to act as the thread's "`main`". It's not
   `main()`-proper, but analogous to it. The thread will start running
   there.
2. From the main thread, launch a new thread with `thrd_create()`, and
   pass it a pointer to the function to run.
3. In that function, have the thread do whatever it has to do.
4. Meantimes, the main thread can continue doing whatever _it_ has to
   do.
5. When the main thread decides to, it can wait for the child thread to
   complete by calling `thrd_join()`. Generally you **must**
   `thrd_join()` the thread to clean up after it or else you'll leak
   memory^[Unless you `thrd_detach()`. More on this later.]

`thrd_create()` takes a pointer to the function to run, and it's of type
`thrd_start_t`, which is `int (*)(int *)`. That's Greek for "a pointer
to a function that takes an `int*` as an argument, and returns an `int`."

Let's make a thread!  We'll launch it from the main thread with
`thrd_create()` to run a function, do some other things, then wait for
it to complete with `thrd_join()`.

``` {.c .numberLines}
#include <stdio.h>
#include <threads.h>

// This is the function the thread will run. It can be called anything.
//
// arg is the argument pointer passed to `thrd_create()`.
//
// The parent thread will get the return value back from `thrd_join()`'
// later.

int run(void *arg)
{
    int *a = arg;  // We'll pass in an int* from thrd_create()

    printf("THREAD: Running thread with arg %d\n", *a);

    return 12;  // Value to be picked up by thrd_join() (chose 12 at random)
}

int main(void)
{
    thrd_t t;  // t will hold the thread ID
    int arg = 3490;

    printf("Launching a thread\n");

    // Launch a thread to the run() function, passing a pointer to 3490
    // as an argument. Also stored the thread ID in t:

    thrd_create(&t, run, &arg);

    printf("Doing other things while the thread runs\n");

    printf("Waiting for thread to complete...\n");

    int res;  // Holds return value from the thread exit

    // Wait here for the thread to complete; store the return value
    // in res:

    thrd_join(t, &res);

    printf("Thread exited with return value %d\n", res);
}
```

See how we did the `thrd_create()` there to call the `run()` function?
Then we did other things in `main()` and then stopped and waited for the
thread to complete with `thrd_join()`.

Sample output (yours might vary):

```
Launching a thread
Doing other things while the thread runs
Waiting for thread to complete...
THREAD: Running thread with arg 3490
Thread exited with return value 12
```

The `arg` that you pass to the function has to have a lifetime long
enough so that the thread can pick it up before it goes away. Also, it
needs to not be overwritten by the main thread before the new thread can
use it.

Let's look at an example that launches 5 threads. One thing to note here
is how we use an array of `thrd_t`s to keep track of all the thread IDs.

``` {.c .numberLines}
#include <stdio.h>
#include <threads.h>

int run(void *arg)
{
    int i = *(int*)arg;

    free(arg);

    printf("THREAD %d: running!\n", i);

    return i;
}

#define THREAD_COUNT 5

int main(void)
{
    thrd_t t[THREAD_COUNT];

    int i;

    printf("Launching threads...\n");
    for (i = 0; i < THREAD_COUNT; i++)
        thrd_create(t + i, run, &i);   // <--- NOTE THIS

    printf("Doing other things while the thread runs...\n");
    printf("Waiting for thread to complete...\n");

    for (int i = 0; i < THREAD_COUNT; i++) {
        int res;
        thrd_join(t[i], &res);

        printf("Thread %d complete!\n", res);
    }

    printf("All threads complete!\n");
}
```

When I run the threads, I count `i` up from 0 to 4. And pass a pointer
to it to `thrd_create()`. This pointer ends up in the `run()` routine
where we make a copy of it.

Simple enough? Here's the output:

```
Launching threads...
THREAD 2: running!
THREAD 3: running!
THREAD 4: running!
THREAD 2: running!
Doing other things while the thread runs...
Waiting for thread to complete...
Thread 2 complete!
Thread 2 complete!
THREAD 5: running!
Thread 3 complete!
Thread 4 complete!
Thread 5 complete!
All threads complete!
```

Whaaa---? Where's `THREAD 0`? And why do we have a `THREAD 5` when
clearly `i` is never more than `4` when we call `thrd_create()`? And two
`THREAD 2`s? Madness!

This is getting into the fun land of _race conditions_. The main thread
is modifying `i` before the thread has a chance to copy it. Indeed, `i`
makes it all the way to  `5` and ends the loop before the last thread
gets a chance to copy it.

We've got to have a per-thread variable that we can refer to so we can
pass it in as the `arg`.

We could have a big array of them. Or we could `malloc()` space (and
free it somewhere---maybe in the thread itself.)

Let's give that a shot:

``` {.c .numberLines}
#include <stdio.h>
#include <stdlib.h>
#include <threads.h>

int run(void *arg)
{
    int i = *(int*)arg;  // Copy the arg

    free(arg);  // Done with this

    printf("THREAD %d: running!\n", i);

    return i;
}

#define THREAD_COUNT 5

int main(void)
{
    thrd_t t[THREAD_COUNT];

    int i;

    printf("Launching threads...\n");
    for (i = 0; i < THREAD_COUNT; i++) {

        // Get some space for a per-thread argument:

        int *arg = malloc(sizeof *arg);
        *arg = i;

        thrd_create(t + i, run, arg);
    }

    // ...
```

Notice on lines 27-30 we `malloc()` space for an `int` and copy the
value of `i` into it. Each new thread gets its own freshly-`malloc()`d
variable and we pass a pointer to that into the `run()` function.

Once `run()` makes its own copy of the `arg` on line 7, it `free()`s the
`malloc()`d `int`. And now that it has its own copy, it can do with it
what it pleases.

And a run shows the result:

```
Launching threads...
THREAD 0: running!
THREAD 1: running!
THREAD 2: running!
THREAD 3: running!
Doing other things while the thread runs...
Waiting for thread to complete...
Thread 0 complete!
Thread 1 complete!
Thread 2 complete!
Thread 3 complete!
THREAD 4: running!
Thread 4 complete!
All threads complete!
```

There we go! Threads 0-4 all in effect!

Your run might vary---how the threads get scheduled to run is beyond the
C spec. We see in the above example that thread 4 didn't even begin
until threads 0-1 had completed. Indeed, if I run this again, I likely
get different output. We cannot guarantee a thread execution order.

## Detaching Threads

If you want to fire-and-forget a thread (i.e. so you don't have to
`thrd_join()` it later), you can do that with `thrd_detach()`.

This removes the parent thread's ability to get the return value from
the child thread, but if you don't care about that and just want threads
to clean up nicely on their own, this is the way to go.

Basically we're going to do this:

``` {.c}
thrd_create(&t, run, NULL);
thrd_detach(t);
```

where the `thrd_detach()` call is the parent thread saying, "Hey, I'm
not going to wait for this child thread to complete with `thrd_join()`.
So go ahead and clean it up on your own when it completes."

``` {.c .numberLines}
#include <stdio.h>
#include <threads.h>

int run(void *arg)
{
    (void)arg;

    //printf("Thread running! %lu\n", thrd_current()); // non-portable!
    printf("Thread running!\n");

    return 0;
}

#define THREAD_COUNT 10

int main(void)
{
    thrd_t t;

    for (int i = 0; i < THREAD_COUNT; i++) {
        thrd_create(&t, run, NULL);
        thrd_detach(t);               // <-- DETACH!
    }

    // Sleep for a second to let all the threads finish
    thrd_sleep(&(struct timespec){.tv_sec=1}, NULL);
}
```

Note that in this code, we put the main thread to sleep for 1 second
with `thrd_sleep()`---more on that later.

Also in the `run()` function, I have a commented-out line in there that
prints out the thread ID as an `unsigned long`. This is non-portable,
because the spec doesn't say what type a `thrd_t` is under the hood---it
could be a `struct` for all we know. But that line works on my system.

Something interesting I saw when I ran the code, above, and printed out
the thread IDs was that some threads had duplicate IDs! This seems like
it should be impossible, but C is allowed to _reuse_ thread IDs after
the corresponding thread has exited. So what I was seeing was that some
threads completed their run before other threads were launched.

## Thread Local Data

Threads are interesting because they don't have their own memory beyond
local variables. If you want a `static` variable or file scope variable,
all threads will see that same variable.

This can lead to race conditions, where you get _Weird Things_™
happening.

Check out this example. We have a `static` variable `foo` in block scope
in `run()`. This variable will be visible to all threads that pass
through the `run()` function. And the various threads can effectively
step on each other's toes.

Each thread copies `foo` into a local variable `x` (which is not shared
between threads---all the threads have their own call stacks). So they
_should_ be the same, right?

And the first time we print them, they are^[Though I don't think they
have to be. It's just that the threads don't seem to get rescheduled
until some system call like might happen with a `printf()`... which is
why I have the `printf()` in there.]. But then right after that, we
check to make sure they're still the same.

And they _usually_ are. But not always!

``` {.c .numberLines}
#include <stdio.h>
#include <stdlib.h>
#include <threads.h>

int run(void *arg)
{
    int n = *(int*)arg;  // Thread number for humans to differentiate

    free(arg);

    static int foo = 10;  // Static value shared between threads

    int x = foo;  // Automatic local variable--each thread has its own

    // We just assigned x from foo, so they'd better be equal here.
    // (In all my test runs, they were, but even this isn't guaranteed!)

    printf("Thread %d: x = %d, foo = %d\n", n, x, foo);

    // And they should be equal here, but they're not always!
    // (Sometimes they were, sometimes they weren't!)

    // What happens is another thread gets in and increments foo
    // right now, but this thread's x remains what it was before!

    if (x != foo) {
        printf("Thread %d: Craziness! x != foo! %d != %d\n", n, x, foo);
    }

    foo++;  // Increment shared value

    return 0;
}

#define THREAD_COUNT 5

int main(void)
{
    thrd_t t[THREAD_COUNT];

    for (int i = 0; i < THREAD_COUNT; i++) {
        int *n = malloc(sizeof *n);  // Holds a thread serial number
        *n = i;
        thrd_create(t + i, run, n);
    }

    for (int i = 0; i < THREAD_COUNT; i++) {
        thrd_join(t[i], NULL);
    }
}
```

Here's an example output (though this varies from run to run):

```
Thread 0: x = 10, foo = 10
Thread 1: x = 10, foo = 10
Thread 1: Craziness! x != foo! 10 != 11
Thread 2: x = 12, foo = 12
Thread 4: x = 13, foo = 13
Thread 3: x = 14, foo = 14
```

In thread 1, between the two `printf()`s, the value of `foo` somehow
changed from `10` to `11`, even though clearly there's no increment
between the `printf()`s!

It was another thread that got in there (probably thread 0, from the
look of it) and incremented the value of `foo` behind thread 1's back!

Let's solve this problem two different ways. (If you want all the
threads to share the variable _and_ not step on each other's toes,
you'll have to read on to the [mutex](#mutex) section.)

## `_Thread_local` Storage-Class {#thread-local}

First things first, let's just look at the easy way around this: the
`_Thread_local` storage-class.

Basically we're just going to slap this on the front of our block scope
`static` variable and things will work! It tells C that every thread
should have its own version of this variable, so none of them step on
each other's toes.

The `<threads.h>` header defines `thread_local` as an alias to `_Thread_local`
so your code doesn't have to look so ugly.

Let's take the previous example and make `foo` into a `thread_local`
variable so that we don't share that data.

``` {.c .numberLines startFrom="5"}
int run(void *arg)
{
    int n = *(int*)arg;  // Thread number for humans to differentiate

    free(arg);

    thread_local static int foo = 10;  // <-- No longer shared!!
```

And running we get:

```
Thread 0: x = 10, foo = 10
Thread 1: x = 10, foo = 10
Thread 2: x = 10, foo = 10
Thread 4: x = 10, foo = 10
Thread 3: x = 10, foo = 10
```

No more weird problems!

One thing: if a `thread_local` variable is block scope, it **must** be
`static`. Them's the rules. (But this is OK because non-`static`
variables are per-thread already since each thread has it's own
non-`static` variables.)

A bit of a lie there: block scope `thread_local` variables can also be
`extern`.

### Another Option: Thread-Specific Storage

Thread-specific storage (TSS) is another way of getting per-thread data.

One additional feature is that these functions allow you to specify a
destructor that will be called on the data when the TSS variable is
deleted. Commonly this destructor is `free()` to automatically clean up
`malloc()`d per-thread data. Or `NULL` if you don't need to destroy
anything.

The destructor is type `tss_dtor_t` which is a pointer to a function
that returns `void` and takes a `void*` as an argument (the `void*`
points to the data stored in the variable). In other words, it's a `void
(*)(void*)`, if that clears it up. Which I admit it probably doesn't.
Check out the example, below.

Generally, `thread_local` is probably your go-to, but if you like the
destructor idea, then you can make use of that.

The usage is a bit weird in that we need a variable of type `tss_t` to
be alive to represent the value on a per thread basis. Then we
initialize it with `tss_create()`. Eventually we get rid of it with
`tss_delete()` (which calls the destructor for all the threads).

In the middle, threads can call `tss_set()` and `tss_get()` to set and
get the value.

In the following code, we set up the TSS variable before creating the
threads, then clean up after the threads.

In the `run()` function, the threads `malloc()` some space for a string
and store that pointer in the TSS variable.

When the main thread finally calls `tss_delete()`, the destructor
function (`free()` in this case) is called for _all_ the threads.

``` {.c .numberLines}
#include <stdio.h>
#include <stdlib.h>
#include <threads.h>

tss_t str;

void some_function(void)
{
    // Retrieve the per-thread value of this string
    char *tss_string = tss_get(str);

    // And print it
    printf("TSS string: %s\n", tss_string);
}

int run(void *arg)
{
    int serial = *(int*)arg;  // Get this thread's serial number
    free(arg);

    // malloc() space to hold the data for this thread
    char *s = malloc(64);
    sprintf(s, "thread %d! :)", serial);  // Happy little string

    // Set this TSS variable to point at the string
    tss_set(str, s);

    // Call a function that will get the variable
    some_function();

    return 0;
}

#define THREAD_COUNT 15

int main(void)
{
    thrd_t t[THREAD_COUNT];

    // Make a new TSS variable, the free() function is the destructor
    tss_create(&str, free);

    for (int i = 0; i < THREAD_COUNT; i++) {
        int *n = malloc(sizeof *n);  // Holds a thread serial number
        *n = i;
        thrd_create(t + i, run, n);
    }

    for (int i = 0; i < THREAD_COUNT; i++) {
        thrd_join(t[i], NULL);
    }

    // And we're done--this calls the destructor (free()) on the string
    tss_delete(str);
}
```

Again, this is kind of a painful way of doing things compared to
`thread_local`, so unless you really need that destructor functionality,
I'd use that instead.

<!--
## Mutexes
## Condition Variables
-->