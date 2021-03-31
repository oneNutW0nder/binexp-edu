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

## Payload (Ran Inside Phoenix Machine)
```bash
$ ./stack_five/exploit.py
```



## Solution

- Now for the really fun stuff! This time, as the challenge suggests, we will be writing our own shellcode into the buffer and using our control over the return address to execute our shellcode. Note, this is made possible in part by the fact that our buffer is 128 bytes large. This is the amount of space that we have to work with when writing our shell code.

- Taking a look at the manpage for `execve` we see its function definition: `int execve(const char *path, char *const argv[], char *const envp[]);`. Reading the manpage is helpful as well. At the very least give it a glance especially the _Examples_ section.

- First I wanted to take a look and see what this shellcode stuff was all about. Essentially, it is our own custom assembly code that we are going to put on the stack and point the return pointer to the beginning of our shellcode so that it executes when the function returns. We are going to have our shellcode call `/bin/sh` which will drop us into a shell if we are successful!

- To make this shellcode we can write it ourselves or use tools to generate it for us. We are going to leverage the work that others have already done so tools it is! Python's `pwnlib` is an incredible resource for binary exploitation. We will be using the `shellcraft` for this challenge.

- `shellcraft` lets us do cool things like specifying a premade payload then having it output it in a format that is condusive to our needs. In our case, we want to drop into a shell and the _amd64_ platform. We can use `shellcraft` to give us what we want.

```python
from pwn import *

# Make shellcode that calls /bin/sh via exeve()
context.update(arch="amd64", os="linux", endian="little")
shellcode = asm(shellcraft.amd64.linux.sh())
```

- The above code will make the shellcode for us and put it in the correct _little-endian_ format so we don't have to worry about it.

- Now that we have our shellcode we can attack the program! We know that we need to write our shellcode into the buffer, that's easy, then we need to control the return pointer of the function's stack frame and point it back into the buffer so that our shell code gets executed. First, let's figure out where the return address overwrite happens. We know that our buffer is 128 bytes so let's start with an input larger than that. In this example I used the `cyclic 144` command to generate a cyclic input of 144 bytes. Now I can quickly locate the point at which we begin to overflow the return address by using `cyclic -l 0x6261616a` (the last 4 bytes of the overflown address). 

![](Pasted%20image%2020210331164420.png)

- Now we know that we have 136 bytes to work with until we need to put our return address that will point back into our buffer. Now we can encorporate our shellcode and point the return pointer to the beginning of our code! But wait... Isn't it kind of hard to point the return address exactly at the beginning of our shellcode? Is there a better way? YES! Enter _NOP Sleds_.

- A `NOP` instruction is a _no operation_. It literally does nothing and the instruction pointer just increments and moves on. This is useful because we can fill the buffer with a bunch of these `NOP` instructions before our shellcode. That way, when the function returns, we don't have to be as precise as hitting the very beginning of our shellcode on the nose. We can afford to miss and be less accurate and land anywhere within the `NOP` instructions which will be ridden all the way until our actual shellcode begins. This is why they are called _NOP Sleds_. 

- It is also time to mention that executing the program within `gdb` versus normally produces different internal addresses for the program. This is important to keep in mind as we move forward because if we get an exploit working within `gdb` it may not work when we run the program normally. We can attach to a running process and view the _real_ addresses by doing: 

```bash
$ sudo gdb -q -p $(ps faux | grep -E "^phoenix.*stack-five" | awk '{
print $2}')
```

- Make sure to execute the program first, it will wait for your input, then run the above command to attach to it with `gdb`. Now we can debug the legitimate process and have a better idea of the addresses being used.

- Our goal is to fill our 136 bytes with the following:

```
|-----NOPS-------|------SHELL CODE-------|---------NOPS-----------|ADDR|
\------------------------------136 bytes--------------------------/
```

- We have `NOP` padding on both sides so that we don't overwrite any of our instructions accidentally. We followup the 136 bytes with the return address that we control. We set this value to be at the beginning of our buffer. In this case it will be the value of `rsp` which is `RSP = 0x7fffffffe5a0`. 

- View the code at `./stack_five/exploit.py` for more details.

![](Pasted%20image%2020210331165332.png)

- Success!





