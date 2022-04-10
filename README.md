# The MCF gthread library

|CI            |Category                   |Host OS         |Build Status     |Remarks          |
|:-------------|:--------------------------|:---------------|:----------------|:----------------|
|[**AppVeyor**](https://ci.appveyor.com/project/lhmouse/mcfgthread) |:1st_place_medal:Primary |Windows (MSYS2) |[![Build Status](https://ci.appveyor.com/api/projects/status/github/lhmouse/mcfgthread?branch=master&svg=true)](https://ci.appveyor.com/project/lhmouse/mcfgthread) ||

MCF gthread is a gthread implementation for Windows 7 and above that meets the requirements of gthread interfaces, playing a role as a replacement for the (currently working but smelly and inefficient) gthr-win32 implementation when porting GCC.

I decide to recreate everything from scratch. Apologies for the trouble.

# How to build

You need to run these commands in a native MSYS2 shell (**MINGW32** or **MINGW64** is recommended):

```sh
autoreconf -i  # requires autoconf, automake and libtool
./configure
make -j$(nproc)
make -j$(nproc) check
```

Cross-compiling from Linux is also supported:

```sh
autoreconf -i  # requires autoconf, automake and libtool
# Install cross-compilers first.
# On Debian this can be done with `sudo aptitude install gcc-mingw-w64-{i686,x86-64}`.
./configure --host=i686-w64-mingw32  # or `x86_64-w64-mingw32` for 64-bit builds
make -j$(nproc)
```

# Notes

This project is developed and tested on x86 and x64 and hasn't been tested on other CPU architectures.

This project uses some undocumented NT system calls and might be broken in future Windows versions. The author gives no warranty for this project. Use it at your own risk.

# Implementation details

### The condition variable

A condition variable is implemented as an atomic counter of threads that are currently waiting on it. Initially the counter is zero, which means no thread is waiting.

When a thread is about to start waiting on a condition variable, it increments the counter and suspends itself using the global keyed event, passing the address of the condition variable as the key. Another thread may read the counter to tell how many threads that it will have to wake up (note this has to be atomic), and release them from the global keyed event, also passing the address of the condition variable as the key.

### The primitive mutex

A primitive mutex is just a condition variable with a boolean bit, which designates whether the mutex is LOCKED. A mutex is initialized to all-bit zeroes which means it is unlocked and no thread is waiting.

When a thread wishes to lock a mutex, it checks whether the LOCKED bit is clear. If so, it sets the LOCKED bit and returns, having taken ownership of the mutex. If the LOCKED bit has been set by another thread, it goes to wait on the condition variable. If the thread wishes to unlock this mutex, it clears the LOCKED bit and wakes up at most one waiting thread on the condition variable, if any.

### The 'real' mutex

In reality, critical sections are fairly small. If a thread fails to lock a mutex, it might be able to do so soon, and we don't want it to give up its time slice as a syscall is an overkill. Therefore, it is reasonable for a thread to perform some spinning (busy waiting), before it actually decides to sleep.

This could however lead to severe problems in case of heavy contention. When there are hundreds of thread attempting to lock the same mutex, the system scheduler has no idea whether they are spinning or not. As it is likely that a lot of threads will eventually give up spinning and make a syscall to sleep, we are wasting a lot of CPU time and aggravating the situation.

This issue is ultimately solved by mcfgthread by encoding a spin failure counter in each mutex. If a thread gives up spinning because it couldn't lock the mutex within a given number of iterations, the spin failure counter is incremented. If a thread locks a mutex successfully while it is spinning, the spin failure counter is decremented. This counter provides a heuristic way to determine how heavily a mutex is seized. If there have been many spin failures, newcomers will not attempt to spin, but will make a syscall to sleep on the mutex directly.

### The once-initialization flag

A once-initialization flag contains a READY byte (this is the first one according to Itanium ABI) which indicates whether initialization has completed. The other bytes are used as a primitive mutex.

A thread that sees the READY byte set to non-zero knows initialization has been done, so it will return immediately. A thread that sees the READY byte set to zero will lock the bundled primitive mutex, and shall perform initialization thereafter. If initialization fails, it unlocks the primitive mutex without setting the READY byte, so the next thread that locks the primitive mutex will perform initialization. If initialization is successful, it sets the READY byte and unlocks the primitive mutex, releasing all threads that are waiting on it. (Do you remember that a primitive mutex actually contains a condition variable?)
