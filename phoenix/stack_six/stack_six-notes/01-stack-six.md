# Stack Six

## Code

```c
/*
 * phoenix/stack-six, by https://exploit.education
 *
 * Can you execve("/bin/sh", ...) ?
 *
 * Why do fungi have to pay double bus fares? Because they take up too
 * mushroom.
 */

#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *what = GREET;

char *greet(char *who) {
  char buffer[128];
  int maxSize;

  maxSize = strlen(who);
  if (maxSize > (sizeof(buffer) - /* ensure null termination */ 1)) {
    maxSize = sizeof(buffer) - 1;
  }

  strcpy(buffer, what);
  strncpy(buffer + strlen(buffer), who, maxSize);

  return strdup(buffer);
}

int main(int argc, char **argv) {
  char *ptr;
  printf("%s\n", BANNER);

#ifdef NEWARCH
  if (argv[1]) {
    what = argv[1];
  }
#endif

  ptr = getenv("ExploitEducation");
  if (NULL == ptr) {
    // This style of comparison prevents issues where you may accidentally
    // type if(ptr = NULL) {}..

    errx(1, "Please specify an environment variable called ExploitEducation");
  }

  printf("%s\n", greet(ptr));
  return 0;
}
```

## Payload

## Solution

- These are going to be less _blog-posty_ now because we are starting to get into the area of _what the fuck is going on_.

- Looks like the program checks for the environment variable `ExploitEducation` and continues if it is set otherwise it exits. Then we pass the contents of `ExploitEducation` to a function `greet()` where it does some fishy looking length checking, a `strcpy()` and then a `strncopy()`. Once this is done it calls `strdup()` which returns a pointer to a new string that is a copy of the string that was passed to it. Essentially, this makes a copy of the string residing in `buffer` and returns a pointer to that copy. This new pointer gets sent back to `main()` and the _greeting_ message is printed out.

![](Pasted%20image%2020210331171704.png)

- Values we can control:
	- `ptr` --> This becomes a pointer to what is stored in the environment variable `ExploitEducation`
		- `who` --> This is the same as `ptr` which is passed to the `greet` function

- Fishy code:
	- Gets size of `who` which is a buffer we control from `ExploitEducation`
	- If the size of `who` is greater than the size of `buffer-1` then size is updated to be equal to the size of  `buffer-1`
	- `what` is then copied into `buffer` with `strcpy()` --> UNSAFE!
		- `what` is `Welcome, I am pleased to meet you `
	- Then we use `strncpy()` to copy `who` into `buffer+strlen(buffer)` limiting the amount we copy to the size set prior to the copies.


- Explaining exploit idea: 
	- We control `who`
	- By controlling who we can get `maxSize` to be equal to 127 by giving input larger than 127.
	- `maxSize` is used in `strncopy()` which specifies how many bytes to copy from `who` into `buffer+strlen(buffer)`
	- This is an issue because `buffer` is moved forward by `0x22` bytes (the value returned by `strlen(what)`). This means that if `maxSize` is 127 then we are going to overflow `buffer` by about 34 bytes 
		- `buffer = 127 bytes`
		- `strlen(what) = 34 bytes`
		- `127 - 34 = 93 bytes`
		- There are only 93 bytes leftover in `buffer` but we are able to write up to 127 bytes to `buffer + strlen(what)`. This means we can overflow the buffer by up to 34 bytes (`127 - 93 = 34 bytes`)
	- If we can overflow the buffer by 34 bytes, what are some interesting values that we can control?

- **TESTING REVEALS SEG FAULT WITH 400 CHARS!**



