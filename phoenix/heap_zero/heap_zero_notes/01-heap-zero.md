# Heap Zero

## Code
```c
/*
 * phoenix/heap-zero, by https://exploit.education
 *
 * Can you hijack flow control, and execute the winner function?
 *
 * Why do C programmers make good Buddhists?
 * Because they're not object orientated.
 */

#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

struct data {
  char name[64];
};

struct fp {
  void (*fp)();
  char __pad[64 - sizeof(unsigned long)];
};

void winner() {
  printf("Congratulations, you have passed this level\n");
}

void nowinner() {
  printf(
      "level has not been passed - function pointer has not been "
      "overwritten\n");
}

int main(int argc, char **argv) {
  struct data *d;
  struct fp *f;

  printf("%s\n", BANNER);

  if (argc < 2) {
    printf("Please specify an argument to copy :-)\n");
    exit(1);
  }

  d = malloc(sizeof(struct data));
  f = malloc(sizeof(struct fp));
  f->fp = nowinner;

  strcpy(d->name, argv[1]);

  printf("data is at %p, fp is at %p, will be calling %p\n", d, f, f->fp);
  fflush(stdout);

  f->fp();

  return 0;
}
```

## Payload

## Solution

- Based on the code above, the path to the goal of calling the function `winner()` is clear. There are two allocations that happen in the `main()` function for `d` and `f`. `d` becomes a pointer to a struct `data` which contains a buffer for a name which is provided by us via `argv[1]`. `f` is becomes a pointer to a struct `fp` which contains a void pointer which is later used as a function pointer. After the allocations, there is our wonderful insecure `strcpy()` function that copies our input from `argv[1]` into the `name` field of the `data` struct which is pointed to by `d`. This means that our string will be stored on the heap right before the `fp` struct which is pointed to by `f`. This means we can overflow the heap chunks allocated for `d->name` into the `f->fp` which should give us control of program execution. Let's try! 

- After the two calls to `malloc()` we can see that we have space given to us on the heap:

![](Pasted%20image%2020210413002114.png)

- This image above shows the space allocated for the `data` and `fp` structs. Halfway through you can see the address of the `nowinner()` function stored on the heap in the `fp` struct which `f` points to. We can now try giving some large input and see what happens.

![](Pasted%20image%2020210413002327.png)

- Here we gave 80 bytes of input in the form of `A`'s. We can clearly see that we have smashed through our chunk of the heap and overwritten the chunk header information that existed previously. We also added an extra byte `\xbd` which should overflow the lowest byte of the function pointer address and execute the `winner()` function. Unfortunately this does not happen for some reason. My suspicion lies with the byte `0x0a` which is a newline character. If you refer to the first screenshot, you can see that this byte exists in the value of `f->fp`.

- I am dumb and forgot that `0x0a` will stop `strcpy()` from continuing to copy bytes from source to the destination. This means that when we try to corrupt the address `strcpy()` will stop copying, and copy in a null byte which messes with any attempts to directly modify the value that `f->fp` contains. Luckily, the heap is executable so we can put shellcode there and return to it! 

![](Pasted%20image%2020210413004506.png)

