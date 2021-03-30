# Stack One

## Code

```c
/*
 * phoenix/stack-one, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable to 0x496c5962
 *
 * Did you hear about the kid napping at the local school?
 * It's okay, they woke up.
 *
 */

#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  printf("%s\n", BANNER);

  if (argc < 2) {
    errx(1, "specify an argument, to be copied into the \"buffer\"");
  }

  locals.changeme = 0;
  strcpy(locals.buffer, argv[1]);

  if (locals.changeme == 0x496c5962) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Getting closer! changeme is currently 0x%08x, we want 0x496c5962\n",
        locals.changeme);
  }

  exit(0);
}
```

## Solution

- Already this is essentially the same as the previous challenge. The only difference is that we now pass our input in on the command line and we have to overwrite `local.changeme` with a specific value `0x496c5962`

- This can be accomplished because the code uses an usafe `strcpy()` with no length checking. The manpage for `strcpy()` notes that this function will continue writing data to the destination regardless of the size of the destination buffer. Since no other length checking occurs we can abuse this to overrun the buffer just like in `stack_zero`!

- Now we have to give our input when we start the program as a command line arg. It even tells us the number we are setting `local.changeme` to!
![](Pasted%20image%2020210330150631.png)

- So if we were able to get the value `0x41` into `local.changeme` using the techinque from the last exercise, couldn't we do basically the same thing but write the expected value instead of just `A`? Let's try it!

![](Pasted%20image%2020210330151106.png)

- What?! This didn't work!?! And the number isn't even right anymore!!! This is because of a wonderful thing called endianess. The challenge page links to this topic and it is important to understand before moving on. Basically, the processory stores information in little-endian. The simplest explanation I have is to read the value that you want to acheive backwards and that's how it needs to be entered for the CPU to understand it.

- So the value `0x496c5962` needs to be given to the program as `0x62596c49` which if you refer back to our attempt at running the program above is the exact opposite of what the program received. Now when we try executing it:

![](Pasted%20image%2020210330151519.png)

- We successfully set the value :D
- To make this clearer, try different inputs and step through the debugger


