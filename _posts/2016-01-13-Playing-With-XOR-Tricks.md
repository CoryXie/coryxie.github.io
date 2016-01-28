---
layout: post
title: "Playing With XOR Tricks"
description: 
headline: 
modified: 2016-01-13
category: ProgrammingTips
tags: [Programming]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

The `XOR` bitwise operation (or `exclusive or`) takes two sets of bits, and for each pair (the two bits at the same index in each bit set) returns 1 only if one but not both of the bits is 1. Otherwise, it returns 0. Think of it like a bag of chips where only one hand can fit in at a time. If no one reaches for chips, no one gets chips, and if both people reach for chips, they can't fit and no one gets chips either! When performing XOR on two integers, only digit columns used by one but not both integers remain. This is simple enough to understand, but there are a lot of tricks around its usage and sometimes these tricks are used in interviews to test the intelligence of the candidates. Although these kind of smart usage of `XOR` property are really not used by most people in their daily work, people are still happy to ask these questions during interview. This blog entry tries to record some of the tricks as I read through a StackOverflow question.

## Why You Should Just Say No to XOR Trick

This [post](http://www.brunton-spall.co.uk/post/2010/09/07/interview-questions-xor-trick-and-why-you-should-j/) by *Michael Brunton Spall* explained how the basic `XOR` trick works and why he suggests candidates should `say no` during the interview for such questions. Basically, I think if you could `say no`, then you should have grasped the idea behind this trick, and the ability to explain why you `say no` should make you stand out a bit. The trick itself looks like this:

```c

	void swap(char *a, char *b) {
		*a ^= *b;
		*b ^= *a;
		*a ^= *b;
	}

```

Or in C++ you may write it like this:

```c

	void swap(int &x, int &y) {
	    x ^= y;
	    y ^= x;
	    x ^= y;
	}

```

> The mathematics are fairly simple and work because XOR has a useful property, when you XOR A and B, you get a value C.  If you XOR C and A you’ll get B back, if you XOR C and B you’ll get A back. So to do the trick, we XOR A and B, and store C in A, destroying the original A.  We then XOR B and C which gives us A back, which we store in B.  We then XOR A and C which gives us B back that we can store in A again.

There are some reasons that the author suggests we should `say no`:

* The compiler has no idea what the hell you are intending.  Using the rather more standard “char t = a; a = b; b = t;” the compiler knows what you are doing and is able to optimize it.

* Secondly, from an interviewing perspective this is a really crappy question. That means you are not testing somebodies ingenuity, you are simply testing whether they prepared for the test by Goggling common interview questions. 

*  Finally from a Meta note, this is a bad question because I really can’t think of any good time that you would actually want to write your own swap function. In almost all libraries there is a swap function already written for you.

That being said, I am still interested what other tricks people would play with `XOR`.

## XOR operations used to "flip bits"

You can use exclusive or to flip bits - the ones that are ON are turned OFF, and vice versa. This can be handy for swapping two numbers without having space for a third number, for example.

```c

	0x0A ^ 0xFF = 0xF5 ( 00001010 ^ 11111111 = 11110101 )

```

You can also toggle single bits: (`n` is the `nth` bit, which gets toggled in `v`).

```c

	#define TOGGLE_BIT(v, n) ((v) ^ (1 << (n)))

```

Regarding toggling bits - in the "olden days", if you wanted to invert an image, you would do `pix = pix & 1` for each pixel - or if you could do it a byte at a time, `byte = byte & 0xFF`. This would turn black text on a white background into white text on a black background. 

On the creative side, the XOR operation is usually employed to render simple square graphics based upon current X and Y coordinates of a given point. For example, the following gives out a chess board look alike pattern with variable cell size.

```c

	#include <stdio.h>
	
	#define CELLSIZE 2   /* cell size is actually 2 power CELLSIZE */
	#define MASK (1<<CELLSIZE)
	
	int main()
	{
	    int i,j;
	
	    for (i=0;i<32;i++)
	    {
	        for (j=0;j<32;j++)
	            putchar (' '+('*'-' ')*( ((i&MASK)^(j&MASK))!=0 ) );
	        putchar ('\n');
	    }
	    return 0;
	}

```

The above code generates something like below:

```console
	
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	    ****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****
	****    ****    ****    ****

```

In videogame coding, when you don't have hardware sprites and/or multiple playfields, the simplest way to draw a sprite on the top of a bitmapped background so you can erase it at your convenience without having to store the part of screen the sprite modifies, is to XOR the sprite pixels with background pixels (one pixel is encoded by one bit). To erase the sprite, simply draw it again at the same coordinates. This method can also be seen to render the mouse pointer on some non accelerated display hardware.

## XOR operations used for "weak encryption"

`XOR` operations are also used for "weak encryption". You can take the XOR of a string with a (repeated) "code word", and the result will be a string of bytes that make no sense. Apply the same operation again, and the original appears. This is quite a weak algorithm, and should never be used in real life.

```c

	void xor_encrypt(char *key, char *string) {
	    int n = strlen(string);

	    for (int i = 0; i < n; i++) {
	        string[i] = string[i] ^ key[i];
	    }
	}

```

One of the cool things about XOR encryption is that when you apply it twice, you get back the original string – see [http://en.wikipedia.org/wiki/XOR_cipher](http://en.wikipedia.org/wiki/XOR_cipher).

In the `xor_decrypt()` function, you take `string[i]` and `key[i]` and return `string[i] ^ key[i]`. If, now, you `XOR` that with the key again, you get `(string[i] ^ key[i]) ^ key[i] = string[i] ^ (key[i] ^ key[i]) = string[i] ^ 0 = string[i]` (by properties of `XOR` operator).

Thus, you can just run the function `xor_encrypt()` a second time on the output of the first `xor_encrypt()` call to `decrypt` the previously encrypted string. So you can just do this:

```c

	#define xor_decrypt(key, str) xor_encrypt(key, str)

```

However, there are two problems with the above `xor_encrypt()` function. 

* **Problem 1: null characters**

One thing to watch out for: if the character in the string matches the corresponding character in the key, your result will be '\0'. This will be interpreted by the above code as the "end of string" and would stop the decryption loop. To circumvent this, you want to pass the length of the "actual" string as a parameter to your function.

* **Problem 2: short keys**

You also want to make sure you don't run past the end of your key - if the plain text is very long you might have to repeat the key. You can do this with the `%` operator - just recycle the key from the beginning.

```c

	void xor_encrypt(char *key, char *string, int n) {
	    int i;
	    int keyLength = strlen(key);

	    for (i = 0 ; i < n; i++ ) {
	        string[i] = string[i] ^ key[i % keyLength];
	    }
	}

```

## XOR operations used for "finding the maximum of two numbers"

Beside of the swapping of two numbers and bits toggling as other answers explains bit-wise XOR is also used in finding the maximum of two numbers without using `if` statement.

```c

	int max(int x, int y)
	{
	      return x ^ ((x ^ y) & -(x < y)); 
	}

```

It's cool. But that raises the question that is this faster than `#define MAX(a,b) ((a)>(b))?(a):(b)`? 

## Number of bits that will need to be changed to convert an integer to another

There is an interview question asked like this:

> Write a function that will return the number of bits that will need to be changed in order to convert an integer, X, into another integer, Y and vice versa. The function should accept two different integers as input. For example, if your method is passed the integers 12 and 16 then your method should return a 3.

Let’s take a closer look at the example given to us. It says that for the input of a 12 and 16, the output of the method should be a 3. Since we are looking for the number of bits that will need to be changed, let’s convert 12 and 16 to binary to further understand this bit manipulation problem. The binary representation of 12 is 01100 and the binary representation of 16 is 10000. Comparing the binary representation of those two numbers, we can see that the first 3 digits are different, which means that to convert 12 to 16 or 16 to 12 we would have to change 3 numbers. And that is why the method that we write should output a 3. Hopefully that’s clear to you now.

The binary operator that is perfect for this exercise is the `XOR` operator. If we just `XOR` the two numbers being passed in, the result of the `XOR` operation will be binary number that will have a binary 1 each and every time there is a different bit between the inputs x and y, and a binary 0 when the bits in the input x and y are the same.

* First, XOR the two numbers being passed in – call them x and y – and then call the result Z.

* Then, just count the number of binary 1’s in Z since that count represents the number of different bits in the two numbers x and y. In order to count the number of binary 1’s in Z, we can just take Z, and perform the binary AND operation with the number 1. When the result of that operation is a 1 then we know that the very last binary digit in Z is a 1. Then, we can shift Z by 1 bit and repeat until there is nothing left to shift – which is when Z is equal to 0. 

* The number of bits set in an integer can also be calculated with some `popcount()` function that is described in [another post](http://www.coderplay.org/programmingtips/Variable-Precision-SWAR-Algorithm.html).

```c

	int findNumberOfBits(int x, int y) {
		int bitCount = 0;
		
		int z = x ^ y;  //XOR x and y
		
		while (z != 0) {
		  //increment count if last binary digit is a 1:
		  bitCount += z & 1; 
		  z = z >> 1;  //shift Z by 1 bit to the right
		}
		
		return bitCount;
	}

```

## Lonely Integer Problem

There is another interview question asked like this:

> There are N integers in an array A. All but one integer occur in pairs. Your task is to find the number that occurs only once.

Constraints to the question are given below:

* 1≤N<100 
* N % 2=1 (N is an odd number) 
* 0≤A[i]≤100,∀i∈[1,N]

The solution is to `XOR` sequentially to the integers.

```c

	#include <stdio.h>
	#include <string.h>
	#include <math.h>
	#include <stdlib.h>
	#include <assert.h>

	int lonelyinteger(int a_size, int* a) {
	    int sum = a[0];
	    int *pVal = a;

	    for (int i = 1; i < a_size; i++) {
	        pVal++;
	        sum ^= *pVal;
	    }
	    
	    return sum;
	}

	int main() {
	    int res;
	    
	    int _a_size, _a_i;
	    scanf("%d", &_a_size);
	    int _a[_a_size];
	    for(_a_i = 0; _a_i < _a_size; _a_i++) { 
	        int _a_item;
	        scanf("%d", &_a_item);
	        
	        _a[_a_i] = _a_item;
	    }
	    
	    res = lonelyinteger(_a_size, _a);
	    printf("%d", res);
	    
	    return 0;
	}

```

## References

This blog entry is mostly an edit based on the following sources, credits should go to these authors!

* http://stackoverflow.com/questions/21034107/xor-operator-in-c
* http://www.brunton-spall.co.uk/post/2010/09/07/interview-questions-xor-trick-and-why-you-should-j


