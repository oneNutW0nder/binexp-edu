# Stack Five

## Code

```c
/*
 * phoenix/stack-five, by https://exploit.education
 *
 * Can you execve("/bin/sh", ...) ?
 *
 * What is green and goes to summer camp? A brussel scout.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

void start_level() {
  char buffer[128];
  gets(buffer);
}

int main(int argc, char **argv) {
  printf("%s\n", BANNER);
  start_level();
}
```

## Payload
```bash
$ shellcraft -f d amd64.linux.sh 

\x6a\x68\x48\xb8\x2f\x62\x69\x6e\x2f\x2f\x2f\x73\x50\x48\x89\xe7\x68\x72\x69\x01\x01\x81\x34\x24\x01\x01\x01\x01\x31\xf6\x56\x6a\x08\x5e\x48\x01\xe6\x56\x48\x89\xe6\x31\xd2\x6a\x3b\x58\x0f\x05
```



## Solution

- Now for the really fun stuff! This time, as the challenge suggests, we will be writing our own shellcode into the buffer and using our control over the return address to execute our shellcode. Note, this is made possible in part by the fact that our buffer is 128 bytes large. This is the amount of space that we have to work with when writing our shell code.

- Taking a look at the manpage for `execve` we see its function definition: `int execve(const char *path, char *const argv[], char *const envp[]);`. Reading the manpage is helpful as well. At the very least give it a glance especially the _Examples_ section.

- First I wanted to take a look and see what this shellcode stuff was all about. Essentially, it is our own custom assembly code that we are going to put on the stack and point the return pointer to the beginning of our shellcode so that it executes when the function returns. We are going to have our shellcode call `/bin/sh` which will drop us into a shell if we are successful!

- To make this shellcode we can write it ourselves or use tools to generate it for us. We are going to leverage the work that others have already done so tools it is! Python's `pwnlib` is an incredible resource for binary exploitation. We will be using the `shellcraft` for this challenge.

- `shellcraft` lets us do cool things like specifying a premade payload then having it output it in a format that is condusive to our needs. In our case, we want to drop into a shell and the _amd64_ platform. We can use `shellcraft` to give us what we want.

![](Pasted%20image%2020210330224318.png)

- `shellcraft` supports several architectures too! We want the `amd64.linux.sh` pyaload for this challenge. You can play around with the format of the shellcode as well. 

![](Pasted%20image%2020210330224550.png)

- The format that we want is the _escaped hex string_ which is the format that we have been using with python for all the challenges.

- Now that we have our shellcode we can attack the program! We know that we need to write our shellcode into the buffer, that's easy, then we need to control the return pointer of the function's stack frame and point it back into the buffer so that our shell code gets executed. First, let's figure out where the return address overwrite happens. We know that our buffer is 128 bytes so let's start with an input larger than that.

![](Pasted%20image%2020210330225446.png)

- Cool! Using an input of 144 bytes we were able to fully overwrite the return address of the `start_level()` function. In this example I used the `cyclic 144` command to generate a cyclic input of 144 bytes. Now I can quickly locate the point at which we begin to overflow the return address by using `cyclic -l 0x6261616a` (the last 4 bytes of the overflown address). 

![](Pasted%20image%2020210330230006.png)

- Let's check to see if we are actually overflowing at this location in the buffer.

![](Pasted%20image%2020210330230133.png)

- Perfect! Anymore input and we will begin overwriting the return address! Let's make sure we have control of the pointer.

![](Pasted%20image%2020210330230353.png)

- We do indeed have full control of the return address. Now we can encorporate our shellcode and point the return pointer to the beginning of our code! But wait... Isn't it kind of hard to point the return address exactly at the beginning of our shellcode? Is there a better way? YES! Enter _NOP Sleds_.

- A `NOP` instruction is a _no operation_. It literally does nothing and the instruction pointer just increments and moves on. This is useful because we can fill the buffer with a bunch of these `NOP` instructions before our shellcode. That way, when the function returns, we don't have to be as precise as hitting the very beginning of our shellcode on the nose. We can afford to miss and be less accurate and land anywhere within the `NOP` instructions which will be ridden all the way until our actual shellcode begins. This is why they are called _NOP Sleds_. 







