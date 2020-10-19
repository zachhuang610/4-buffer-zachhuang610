# buffer

### Level 1

From analyzing the objdump, I found that the buffer begins 48
bytes below rbp and grows upwards. Then, I went into gdb, setting
a breakpoint at getbuf and searched above rbp to see where
exactly the return address was stored. I found this to be $rbp+0x8
or 8 above rbp. This means that there was a total of 64 bytes
of space between the beginning of the buffer and the return address
location. Thus, I neede 7 lines of 8 byte paddings and 1 line of
the 8 byte address of lights_off, which i got from the objdump.

### Level 2

Same buffer size, so I copied the padding and changed the address
to sandwich_order address which is found in objdump. Then noticed
that func arguments are stored above the func address, so I inputed
the arguments in reverse order (relative to the stack). Realized there
was some padding between the address and the function so I added some
padding until it fit. 

00 00 00 00 00 00 00 00 /* padding */ 
00 00 00 00 00 00 00 00 /* padding */
00 00 00 00 00 00 00 00 /* padding */
00 00 00 00 00 00 00 00 /* padding */
00 00 00 00 00 00 00 00 /* padding */
00 00 00 00 00 00 00 00 /* padding */
00 00 00 00 00 00 00 00 /* padding */
c0 13 40 00 00 00 00 00 /* address to sandwhich order */
00 00 00 00 00 00 00 00 /* padding */
7d 32 bc 2f             /* cookie */
06 00 00 00 07 00 00 00 /* array arguments */
08 00 00 00 09 00 00 00

### Level 3

Same buffer size and address, so i copied the padding and return address
from the previous exploits. The trick is was to write an exploit code which
changed %rax, reset %rbp to the that of the caller (test_exploit), and 
return to the line in test_exploit that would have been called. Changing
%rax was simple, resetting %rbp required me to go into gdb and see what
the old %rbp was before Gets is called. and the line to return to in test_exploit
was found in objdump. The following assembly code was turned into hex (not including nops):

mov $0x000000002fbc327d, %rax
push $0x4014b5
retq

Followed by enough NOPS to fill the buffer. Another 8 bytes were given
to the old %rbp address, which was found in gdb prior to calling Gets.
The last 8 bytes of the exploit code is the return address. Instead of the
address calling a separate function, we are now going into the buffer.
Finding this address is trivial in gdb, as we already know the size and
offset from %rbp from the earlier problems. 


### Level 4

Level 4 required:
1.) getbufn call to return to the buffer, offset by a random amount
2.) change %rax to the cookie.
3.) restore %rbp in a stack where %rbp changes between runs and is not fixed
4.) return to test_exploitn without any noticable stack changes.

Since we only know a range of buffer values, we store our exploit code
near the top of stack and use a nop sled to slide into the code from
all ranges of buffer starting positions. I ran through each of the 5
iterations of test_exploitn and each call to getbufn to find the highest
value of the starting buffer stack and used that as the return address.
I then put enough nops before my exploit code to ensure that my code
was stored above that address.

The following assembly is what is stored in hex (not including nops):

mov $0x000000002fbc327d, %rax
mov %rsp, %rbp
add $0x20, %rbp
push $0x401675
retq

I stored my cookie in %rax to make sure it returns the cookie. To ensure
that I restored the %rbp value, I utilized the fact that before getbufn
returns, it collapses %rsp into what was %rbp before I overwrote it with
nops. From there I found the offset from the getbufn %rbp to where the
old %rbp is stored, which was 0x20. Thus to restore %rbp to old, I 
moved %rsp (which is getbufn unmutated %rbp location) to %rbp and then 
added the offset to return it to the original pointer. Then I returned