# Stack Three

## Code

```c
/*
 * phoenix/stack-three, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable to 0x0d0a090a
 *
 * When does a joke become a dad joke?
 *   When it becomes apparent.
 *   When it's fully groan up.
 *
 */

#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

void complete_level() {
  printf("Congratulations, you've finished " LEVELNAME " :-) Well done!\n");
  exit(0);
}

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int (*fp)();
  } locals;

  printf("%s\n", BANNER);

  locals.fp = NULL;
  gets(locals.buffer);

  if (locals.fp) {
    printf("calling function pointer @ %p\n", locals.fp);
    fflush(stdout);
    locals.fp();
  } else {
    printf("function pointer remains unmodified :~( better luck next time!\n");
  }

  exit(0);
}
```

## Solution

- Now these are getting interesting! We now have a new function `complete_level()` which prints our congratualtion message. However, this function is never called in `main()`.... Let's keep digging in.

- Our `local.changeme` value has been replaced by a function pointer. If you do not know what these are look them up then come back!

- The rest of the code follows a similar pattern. An unsafe of `gets()` to read data into `local.buffer` which we can exploit to overwrite the function pointer. The code checks to see if the function pointer is null. If it is not null, the the functoin pointer is called by `locals.fp()`.

- If we can control the function pointer maybe we can call the `complete_level()` function! 

- Let's take a look at the functions and set a breakpoint on main like usual

![](Pasted%20image%2020210330153632.png)

- It is important to note that running `info functions` before program execution will only show the functions contained within the binary. If you start the program and then run `info functions` you will see a list of all function that have been dynamically loaded at the start of execution.

- Let's note the address of `copmlete_level()`: `0x000000000040069d`. Also note that for now things like ASLR are probably not working for these challenges as that would make them much more complicated. 

- The assembly here looks the same as it did on `stack_zero`. `locals.fp` is set to `NULL` which is really zero and our buffer is setup and waiting for our input to be written to it.

![](Pasted%20image%2020210330154507.png)

- Let's see what happens when we try a payload from a previous challenge!

![](Pasted%20image%2020210330154809.png)

- This is expected! We overran the buffer and wrote data into `locals.fp`. Let's keep going and see what happens. 

- First the value that we entered is moved into `rax`. The `rax` register now contains the data that we wrote to `locals.fp`. This is then checked to make sure that it is not `NULL` (zero) by `test eax, eax`. Since we successfully wrote data to this location this check passes and we should continue into where `locals.fp()` is called.

![](Pasted%20image%2020210330155142.png)

- This is the print statement right before the `locals.fp()` call. We can see that it is going to execute the function pointer at the memory location. But wait! What is that address?

![](Pasted%20image%2020210330155311.png)

- A segfault!!! Looking closer, we can see that the last instruction executed before the crash was a `call` to the data that we wrote to `locals.fp` using the overflow! LIGHTBULB! What if we write the address of `complete_level()` to `locals.fp`! Then it would call that function and we would complete the challenge. Let's try!

![](Pasted%20image%2020210330162520.png)

- Almost! The address is almost correct but the last byte is `0xc2` which is not what we want! Let's experiment with a larger input.

![](Pasted%20image%2020210330162921.png)

- Interesting. We continue to overflow into `locals.fp` but that pesky byte `0xc2` is still there.

![](Pasted%20image%2020210330164451.png)

- The pesky byte `0xc2` is gone! After some thought and research I believe this is due to padding of the bytes occuring in order to keep everything aligned. If we back off our buffer input by one `A` we can force the padding to occur in the previous double word instead of where we need to write the address of `complete_level()`

![](Pasted%20image%2020210330164750.png)

- Now the padding is occuring before the target memory and our address is intact!

![](Pasted%20image%2020210330164819.png)

- Success!


