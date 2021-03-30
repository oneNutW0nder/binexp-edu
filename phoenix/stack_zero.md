# Stack Zero

## Code

```c
/*
 * phoenix/stack-zero, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable.
 *
 * Scientists have recently discovered a previously unknown species of
 * kangaroos, approximately in the middle of Western Australia. These
 * kangaroos are remarkable, as their insanely powerful hind legs give them
 * the ability to jump higher than a one story house (which is approximately
 * 15 feet, or 4.5 metres), simply because houses can't can't jump.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  printf("%s\n", BANNER);

  locals.changeme = 0;
  gets(locals.buffer);

  if (locals.changeme != 0) {
    puts("Well done, the 'changeme' variable has been changed!");
  } else {
    puts(
        "Uh oh, 'changeme' has not yet been changed. Would you like to try "
        "again?");
  }

  exit(0);
}
```

## Solution

- We can see the struct being created the first few lines of `main()`
  - The interesting part is the buffer of 64 chars followed by an integer
  - Due to struct alignment in memory we will likely be able to overflow the buffer and control the var `changeme`

- Sure enough, the program uses `gets()` which is a vulnerable function and will continue to read data from the user until a newline is encountered or EOF
  - The manpage even says _Never use this function_ at the very top of the page 

- Knowing this we can overflow the 64-byte buffer by one to change the value of `local.changeme`
```python
python -c "print('A' * 65)" # Gives us 65 A's to use as our payload
```

