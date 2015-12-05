---
layout: post
title: "Variable Precision SWAR Algorithm"
description: 
headline: 
modified: 2015-12-05
category: ProgrammingTips
tags: [Programming]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
--- 

The population count of a binary integer value x is the number of one bits in the value. Although many machines have single instructions for this, the single instructions are usually microcoded loops that test a bit per cycle; a log-time algorithm coded in C is often faster. An interesting way to get the `popcount`, or the number of bits set in an integer is via a **SWAR (SIMD Within A Register)** algorithm. While its performance is very good (16 instructions when compiled with `gcc -O3`), why the algorithm works is somewhat opaque. This blog post tries to understand this algorithm.

## Population Count (Ones Count) SWAR Algorithm

[The Aggregate Magic Algorithms](http://aggregate.ee.engr.uky.edu/MAGIC) site maintains "Population Count (Ones Count)" algorithm like below using a `variable-precision SWAR algorithm` to perform a `tree reduction` adding the bits in a 32-bit value:

```c

	unsigned int
	ones32(register unsigned int x)
	{
	        /* 32-bit recursive reduction using SWAR...
		   but first step is mapping 2-bit values
		   into sum of 2 1-bit values in sneaky way
		*/
	        x -= ((x >> 1) & 0x55555555);
	        x = (((x >> 2) & 0x33333333) + (x & 0x33333333));
	        x = (((x >> 4) + x) & 0x0f0f0f0f);
	        x += (x >> 8);
	        x += (x >> 16);
	        return(x & 0x0000003f);
	}

```

## Code Walk Through for Population Count SWAR Algorithm

There is another variation for the "Population Count (Ones Count) Algorithm".

```c

	unsigned int
	ones32(register unsigned int x)
	{
		x = (x & 0x55555555) + ((x >> 1) & 0x55555555);
		x = (x & 0x33333333) + ((x >> 2) & 0x33333333);
		x = (x & 0x0f0f0f0f) + ((x >> 4) & 0x0f0f0f0f);
		x = (x & 0x00ff00ff) + ((x >> 8) & 0x00ff00ff);
		x = (x & 0x0000ffff) + ((x >> 16) &0x0000ffff);

		return x;
	}

```

*Zhi-Jun* made a table to describe the above algorithm (covering the 16 bit version).

<img src="{{ site.baseurl }}/images/2015-12-05-1/popcount-steps-table.png" alt="Population Count SWAR Algorithm Steps">

In explanation of the `ones32()` function, Professor *Hank Dietz* also stated:

> One additional note: the AMD Athlon optimization guidelines suggest a very similar algorithm that replaces the last three lines with `return((x * 0x01010101) >> 24);`. For the Athlon (which has a very fast integer multiply), I would have expected AMD's code to be faster... but it is actually 6% slower according to my benchmarks using a 1.2GHz Athlon (a Thunderbird). Why? Well, it so happens that GCC doesn't use a multiply instruction - it writes out the equivalent shift and add sequence!

So this function then can be adapted like this:

```c

	int SWAR(register unsigned int i)
	{
	    i = i - ((i >> 1) & 0x55555555);
	    i = (i & 0x33333333) + ((i >> 2) & 0x33333333);
	    return (((i + (i >> 4)) & 0x0F0F0F0F) * 0x01010101) >> 24;
	}

```

Someone on [StackOverflow](http://stackoverflow.com/questions/22081738/how-variable-precision-swar-algorithm-works) has noticed that its performance is better than `__builtin_popcount()` on `Intel Pentium III 733 MHz` but can't understand the way it works.

[*Ilmari Karonen*](http://stackoverflow.com/users/411022/ilmari-karonen) made a detailed explanation about how this code works by going through the code line by line:

### Line 1:

    i = i - ((i >> 1) & 0x55555555);

First of all, the significance of the constant `0x55555555` is that, written using the Java / GCC style [binary literal notation](http://gcc.gnu.org/onlinedocs/gcc/Binary-constants.html)),

    0x55555555 = 0b01010101010101010101010101010101

That is, all its odd-numbered bits (counting the lowest bit as bit 1 = odd) are `1`, and all the even-numbered bits are `0`.

The expression `((i >> 1) & 0x55555555)` thus shifts the bits of `i` right by one, and then sets all the even-numbered bits to zero.  (Equivalently, we could've first set all the odd-numbered bits of `i` to zero with `& 0xAAAAAAAA` and *then* shifted the result right by one bit.)  For convenience, let's call this intermediate value `j`.

What happens when we subtract this `j` from the original `i`?  Well, let's see what would happen if `i` had only *two* bits:

        i           j         i - j
    ----------------------------------
    0 = 0b00    0 = 0b00    0 = 0b00
    1 = 0b01    0 = 0b00    1 = 0b01
    2 = 0b10    1 = 0b01    1 = 0b01
    3 = 0b11    1 = 0b01    2 = 0b10

Hey!  We've managed to count the bits of our two-bit number!

OK, but what if `i` has more than two bits set?  In fact, it's pretty easy to check that the lowest two bits of `i - j` will still be given by the table above, *and so will the third and fourth bits*, and the fifth and sixth bits, and so and.  In particular:

* despite the `>> 1`, the lowest two bits of `i - j` are not affected by the third or higher bits of `i`, since they'll be masked out of `j` by the `& 0x55555555`; and

* since the lowest two bits of `j` can never have a greater numerical value than those of `i`, the subtraction will never borrow from the third bit of `i`: thus, the lowest two bits of `i` also cannot affect the third or higher bits of `i - j`.

In fact, by repeating the same argument, we can see that the calculation on this line, in effect, applies the table above to *each* of the 16 two-bit blocks in `i` *in parallel*.  That is, after executing this line, the lowest two bits of the new value of `i` will now contain the *number* of bits set among the corresponding bits in the original value of `i`, and so will the next two bits, and so on.

### Line 2:

    i = (i & 0x33333333) + ((i >> 2) & 0x33333333);

Compared to the first line, this one's quite simple.  First, note that

    0x33333333 = 0b00110011001100110011001100110011

Thus, `i & 0x33333333` takes the two-bit counts calculated above and throws away every second one of them, while `(i >> 2) & 0x33333333` does the same *after* shifting `i` right by two bits.  Then we add the results together.

Thus, in effect, what this line does is take the bitcounts of the lowest two and the second-lowest two bits of the original input, computed on the previous line, and add them together to give the bitcount of the lowest *four* bits of the input.  And, again, it does this in parallel for *all* the 8 four-bit blocks (= hex digits) of the input.

### Line 3:

    return (((i + (i >> 4)) & 0x0F0F0F0F) * 0x01010101) >> 24;

OK, what's going on here?

Well, first of all, `(i + (i >> 4)) & 0x0F0F0F0F` does exactly the same as the previous line, except it adds the adjacent *four-bit* bitcounts together to give the bitcounts of each *eight-bit* block (i.e. byte) of the input.  (Here, unlike on the previous line, we can get away with moving the `&` outside the addition, since we know that the eight-bit bitcount can never exceed 8, and therefore will fit inside four bits without overflowing.)

Now we have a 32-bit number consisting of four 8-bit bytes, each byte holding the number of 1-bit in that byte of the original input.  (Let's call these bytes `A`, `B`, `C` and `D`.)  So what happens when we multiply this value (let's call it `k`) by `0x01010101`?

Well, since `0x01010101 = (1 << 24) + (1 << 16) + (1 << 8) + 1`, we have:

    k * 0x01010101 = (k << 24) + (k << 16) + (k << 8) + k

Thus, the *highest* byte of the result ends up being the sum of:

* its original value, due to the `k` term, plus
* the value of the next lower byte, due to the `k << 8` term, plus
* the value of the second lower byte, due to the `k << 16` term, plus
* the value of the fourth and lowest byte, due to the `k << 24` term.

(In general, there could also be carries from lower bytes, but since we know the value of each byte is at most 8, we know the addition will never overflow and create a carry.)

That is, the highest byte of `k * 0x01010101` ends up being the sum of the bitcounts of all the bytes of the input, i.e. the total bitcount of the 32-bit input number.  The final `>> 24` then simply shifts this value down from the highest byte to the lowest.

**Ps.** This code could easily be extended to 64-bit integers, simply by changing the `0x01010101` to `0x0101010101010101` and the `>> 24` to `>> 56`.  Indeed, the same method would even work for 128-bit integers; 256 bits would require adding one extra shift / add / mask step, however, since the number 256 no longer quite fits into an 8-bit byte.

## References

This blog entry is mostly an edit based on the following sources, credits should go to these authors!

* http://aggregate.ee.engr.uky.edu/MAGIC
* http://www.playingwithpointers.com/swar.html
* https://zhjxue.wordpress.com/tag/%E7%AE%97%E6%B3%95-%E7%BB%9F%E8%AE%A1-hamming-weight/
* http://stackoverflow.com/questions/22081738/how-variable-precision-swar-algorithm-works