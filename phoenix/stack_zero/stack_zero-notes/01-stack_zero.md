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

- Here we can see the program allocating space for the buffer just before the call to `gets()`:
![[Pasted image 20210330143112.png]]

- We see that 0 is moved into `[rbp-0x10]` which is `local.changeme` then a call to `lea rax, [rbp - 0x50]` which sets `rax` to the the start of our buffer for the `local` struct.

- We can confirm this by looking at the values. The first is `[rbp - 0x10]` which is `local.changme`. It is set to zero just like we expect.
![[Pasted image 20210330143428.png]]

- Next is our buffer:
![[Pasted image 20210330144032.png]]

- Note the address that we are inspected is stored in `rax` from the `lea` instruction before the call to `gets()`. This is where our string is being written to! We can see that our buffer is zeroed out and that our `changeme` variable is right next to our buffer! This means we can overwrite it since there is no length check on data input or validation by the `gets()` function.

- Knowing this we can overflow the 64-byte buffer by one to change the value of `local.changeme`
```python
python -c "print('A' * 65)" # Gives us 65 A's to use as our payload
```

- After our input this is what the buffers look like:
![[Pasted image 20210330144326.png]]

- Notice how the `buffer` is full of `0x41` which is `A` in hex, and that we overflowed into the value of `local.changeme` so that its value is no longer zero! Thus giving us the win on this binary challenge.