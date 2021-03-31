# Stack Four

## Code

```c
/*
 * phoenix/stack-four, by https://exploit.education
 *
 * The aim is to execute the function complete_level by modifying the
 * saved return address, and pointing it to the complete_level() function.
 *
 * Why were the apple and orange all alone? Because the bananna split.
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

void start_level() {
  char buffer[64];
  void *ret;

  gets(buffer);

  ret = __builtin_return_address(0);
  printf("and will be returning to %p\n", ret);
}

int main(int argc, char **argv) {
  printf("%s\n", BANNER);
  start_level();
}
```

## Payload
```bash
$ python3 -c "print('A' * 88 + '\x1d\x06\x40')" | ./stack-four
```

## Solution

- Now this is getting even more interesting! We still have the `complete_level()` function which is probably still our target of our buffer overflows. The program starts off by calling `start_level()` so we will focus our attention there.

- There is nothing special going on that we haven't seen before. Our good old friend `gets()` is still there along with a buffer and a `void *ret`. At the end of the function `ret` is set to the value of `__builtin_return_address()` which gets the return address of the current stack frame AKA our `start_level()` function. This is done for debugging purposes so that we get some feedback about our exploit.

- This gives us a good idea of the goal for this challenge. For this one we need to use the `buffer` to overwrite the return address for the function with the address of `complete_level()`. This will cause the function to return and execute the `complete_level()` function. 

- This time, we can't get away with using a small buffer of 64 characters because we have to overwrite the return address which is the last value on the stack. Let's do some testing.

- We can use the `info frame` command in gdb in order to print out information about the current stack frame. This gives us information such as the saved registers. The important one is where the saved `rip` is. This is the instruction pointer for the caller. If we overwrite this we can change the execution of the program! Here is an example.

![](Pasted%20image%2020210330175432.png)

- Now we can begin testing! Lets try an input of 92 characters.

![](Pasted%20image%2020210330175658.png)

- Here is a screen shot showing the location of the saved `rip`, its current value, and a print out of the stack starting from the base pointer `rbp`. We can see that our input overflowed by 4 bytes into the value of the saved `rip`. Let's start trying to exploit this!

![](Pasted%20image%2020210330180314.png)

- Looks like it worked! We took 4 bytes (4 `A`'s) off our input, and put in the address of the `complete_level()` function! If we continue the program execution we should return and execute the `complete_level()` function!

![](Pasted%20image%2020210330180429.png)

- Success!
