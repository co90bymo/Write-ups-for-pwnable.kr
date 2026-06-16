---
title: collision
---

![challengecol](../images/fd/challengecol.png)


After ssh-ing, we use `ls -la` and see the same type of setup as in the [fd]-challenge (we have no permissions to read flag, but we can execute col, and we have readable code col.c)
Let's inspect the code right away and see what we are supposed to do.

```
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){
		setregid(getegid(), getegid());
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;

````
We need to call `./col` with a parameter that consists of exactly 20 bytes, which is 20 characters using regular ASCII code. That is because in C, a char is 1 byte.
Next, we see that our parameter will be passed as a parameter for the `check_password` function, which performs some operation on our input, whose result should be equal to the hashcode `unsigned long hashcode = 0x21DD09EC;` so we can pass the if-condition.
`int* ip = (int*)p;` tells us that our char-pointer is cast into an int-pointer. This is important because ints are 4 Bytes! The following for-loop iterates through that and adds up the first 5 ints.
This means that our input is separated into chunks of 4 bytes, with a total of 20/4 = 5 chunks. 

What does this mean? Each Byte is reinterpreted as a numerical value, so we get 5 chunks of numerical values. Those values are now added.
Because an int is 32bit (4 bytes), the actual value will wrap around, so our result is actually result = result % 2^32.

As indicated by the author's message, this is basically just simulating a simple hash function. We are trying to find an input that gives us the same hash value that is stored in that Code. A collision is when two different inputs result in the same output (a != b -> hash(a) = hash(b)).



**Attempt #1:**

We can convert the problem into this equation: `res = A + B + C + D + E`. Let `A = B = C = D = X`, we get `res = X*4 + E <=>  res - X*4 = E`.
All we need to do is select some X. And now, by simple arithmetic, we get a value for E. 
But this doesn't work because not all byte values correspond to printable characters. For example, if X = "A", then E = 0x1CD804E8. But 0x1C is "file separator" which is not a valid character.
After trying different X, I found that none of the E are usable characters.

**Attempt #2:**

After research, I found out that it's possible to put hex values directly as a parameter. Another thing is that we need to watch out for endianness, and that we cannot use \x00 because this is interpreted as a terminator. So we can't just pass the hashcode as parameter. 
We can use x01 instead, which means we get x01 * 16, or 0x01010101 in 4 chunks. Meaning we need to subtract 0x04040404 from 0x21DD09EC, which gives us 0x1DD905E8.
Our solution is therefore 0x1DD905E804040404, and with respect to the endianness, our final command is the following:

`./col "$(python3 -c 'import sys;sys.stdout.buffer.write(b"\x01"*16+b"\xe8\x05\xd9\x1d")')"`

Sure enough, this prints the flag and completes the challenge.
Note: As of the time of this write-up, the server uses Python 3, where strings are Unicode rather than raw bytes. When printed, the character represented by \xe8 is typically encoded as UTF-8, which occupies 2 bytes (0xC3 0xA8) instead of 1. As a result, the payload length becomes 21 bytes instead of 20 and the challenge fails. Using sys.stdout.buffer.write() writes the raw bytes directly and avoids this issue. If your payload appears correct but still fails, this may be the reason

**Alternative solution: Let Python handle endianness**
You can also apply endianness using `struct.pack("<I",0x1DD905E8)`, so we can use this:
`./col "$(python3 -c 'import sys,struct;sys.stdout.buffer.write(struct.pack("<I",0x01010101)*4 + struct.pack("<I",0x1DD905E8))')"`

We should also be able to use the method from our first attempt. In this example, we divide our hashcode by 5, and add the remainder of 4 to the last chunk (division is not clean):
`./col "$(python3 -c 'import sys,struct;sys.stdout.buffer.write(struct.pack("<I",0x6c5cec8)*4 + struct.pack("<I",0x6c5cec8+4))')"`

Both attempts print the flag to our console.

---

## What I learned

I didn't really learn anything new here, other than correctly passing raw byte input to a program using Python. 
It's basic stuff I already knew or learned from the previous section - turned into a "riddle" of sorts.
Nonetheless, this challenge did show how the bytes stored in memory are really meaningless and only get assigned meaning by the program that is using them. 
For example, if we didn't cast to `int*`, then we would be calculating 1 byte at a time instead of 4. 
So this challenge, in a way, is a nice little introduction into how memory can be manipulated/ interpreted.
