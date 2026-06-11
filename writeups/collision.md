---
title: collision
---

![challengecol](../images/fd/challengecol.png)

!NOT COMPLETE!

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
We need to call `./col` with a parameter that consists of exactly 20 bytes, which is 20 characters on our terminal. That is because in C, a char is 1 byte.
Next, we see that our parameter will be passed as a parameter for the `check_password` function, which performs some operation on our input, whose result should be equal to the hashcode `unsigned long hashcode = 0x21DD09EC;` so we can pass the if-condition.
`int* ip = (int*)p;` tells us that our char-pointer is cast into an int-pointer. This is important because ints are 4 Bytes! The following for-loop iterates through that and adds up the first 5 ints.
This means that our input is separated into chunks of 4 bytes, with a total of 20/4 = 5 chunks. 

What does this mean? Each Byte is converted into an ASCII code. The ASCII codes of one chunk are concatenated, and the resulting Code is our hex value for that chunk.
So we get a numerical value for each of the 5 chunks. Those chunks are now added.
Because an int is 32bit (8 bytes), the actual value will wrap around, so our result is actually result = result % 2^32.

As indicated by the author's message, this is basically just simulating a simple hash function. We are trying to find an input that gives us the same hash value that is stored in that Code. A collision is when two different inputs result in the same input (a != b -> hash(a) = hash (b)).


Attempt #1:
First, I tried something like this. We can convert the problem into this equation: `res = A + B + C + D + E`. Let `A = B = C = D = A`, we get `res = A*4 + E <=>  res - A*4 = E`.
All we need to do is select some A. And now, by simple arithmetic, we get a value for E. 
But this doesn't work because not all ASCII Codes are characters we can type on our terminal. For example, if A = "A", then E = 0x1CD804E8. But 0x1C is "file separator" which is not a valid character.
After trying different A, I found that none of the E are usable ASCII Codes.

Attempt #2:

.. Working on it ..
