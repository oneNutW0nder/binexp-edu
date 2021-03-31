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

- Now for the really fun stuff! This time, as the challenge suggests, we will be writing our own shellcode into the buffer and using our control over the return address to execute our shellcode. Note, this is made possible in part by the fact that our buffer is 128 bytes large. This is the amount of space that we have to work with when writing our shell code.

