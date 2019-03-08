---
title: Getting Bitmasks from SSE Vector Comparisons
layout: post
comments: true
date:   2019-03-08 18:49:47 +0900
tags: c asm x86 x86_64 x64 amd64 sse avx simd simd-basics
summary: How to actually use SIMD comparison results
---

These days the single-threaded performance of CPUs just isn't advancing as
quickly as it did in my salad days of the late 90s and early 2000s.
Personally I think it's God's revenge for people putting pineapple on pizza.
Regardless of the causes, I find it to be increasingly the case that if you
aren't using SIMD in the performance critical areas of your software, then you
are leaving a lot of performance on the floor.

Let's say that you have an array of `uint16_t`'s and you want to go through
them all looking for all instances of a given number. In plain-old-C you would
do something like:

```c
void my_pipeline_is_as_empty_as_my_soul(const unsigned count,
					const uint16_t vector[static count])
{
	unsigned int i;
	for(i = 0; i < count; i++) {
		if ( vector[i] == 0x1234 ) {
			/* do the thing */
		}
	}
}
```

But you've been chilling listening to "smooth" by Carlos Santana, and you've
heard of this hip new technology called SSE2. It allows you to do EIGHT of
these comparisons in a single bound!

So you include the relevant header and you start working on an inner-loop which
handles 8 items at a time. We'll worry about any trailing data later. You'll
use `_mm_load_si128` to just slurp an array of 8 elements in to one of these
new wide-boy registers:

```c
#include <emmintrin.h>
uint16_t in16[8] = {
	/* 0 */ 0x1234,
	/* 1 */ 0x4567,
	/* 2 */ 0x1234,
	/* 3 */ 0x1234,
	/* 4 */ 0x1234,
	/* 5 */ 0,
	/* 6 */ 0x1212,
	/* 7 */ 0x3434,

};
__m128i h = _mm_load_si128((__m128i *)in16)
// 1234 4567 1234 1234 1234 0000 1212 3434
```

And, as before, we are looking for all instances of `0x1234` so we do a
comparison using the `_mm_cmpeq_epi16` intrinsic:

```c
__m128i n = _mm_set1_epi16(0x1234);
// 1234 1234 1234 1234 1234 1234 1234 1234
__m128i r = _mm_cmpeq_epi16(n, h);
// ffff 0000 ffff ffff ffff 0000 0000 0000
```

Astute readers will notice that these equality tests can be achieved with
`and`-ing or `xor`-ing, and that's correct. But the topic of this post is the
comparison operators. These become really useful when you want to do range
comparisons using greater-than and/or less-than or things like that.

Anyway, this is all wonderful but what am I going to do with this weird
two-byte boolean-like thing?

I now have a vector of `uint16_t`'s which are set to all 1s for matching items
and all 0s for non-matching items. We could just break these out in a loop but
that would waste all the effort we've put in to avoid branching and looping by
doing this with SIMD in the first place.

It would be nice if we could get the results in a bitmask, so we can do other
set-wise operations on all 8 elements at once. Or use `__builtin_ctzl`
and `__builtin_clz` respectively to find the first/last set bits.

## The solution

How we're going to do this is by using
[\_mm\_packs\_epi16](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=_mm_packs_epi16&expand=4904,4043)
to collapse the 16bit values down to 8bit values and then
[\_mm\_movemask\_epi8](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#expand=4904,4043,3831,3831&text=_mm_movemask_epi8)
to extract the mask.

```c
__m128i p = _mm_packs_epi16(cmp_res, cmp_res);
// ff 00 ff ff ff 00 00 00 (x2)
int mask = _mm_movemask_epi8(p) & 0x7f;
// 0x1d (or 00011101)
```

Now, this might require a bit of unpacking - if you will excuse the pun. At
first glance `_mm_packs_epi16` doesn't seem an obvious choice. Intel describes
it as:

> Convert packed 16-bit integers from `a` and `b` to packed 8-bit integers
> using signed saturation, and store the results in `dst`.

To put this in plain english, what happens is that, for each 16 bit integer in
our input vector we split it down in to two 8-bit integers and add them
together.  But, instead of wrapping back to zero when we overflow, the values
get 'stuck' at the highest possible value of `0xff` (ie. [saturated
addition](https://en.wikipedia.org/wiki/Saturation_arithmetic)). The resulting
8-bit integer is appended to the result vector.  A C implementation would look
something like this:

```c
uint8_t in[32]; // this is the concatenation of the two 128bit operands
uint8_t out[16]; // output operand

for(i = 0; i < 16; i++) {
	unsigned int one, two, res;

	/* Load the input */
	one = in[i * 2 + 0];
	two = in[i * 2 + 1];	

	/* Saturated addition */
	res = one + two;
	if ( res > 0xff ) {
		res = 0xff;
	}

	/* Store the result */
	out[i] = res;
}
```

To avoid adding a dependency on any other registers we use the same input
register for both operands. This leaves us with two identical copies of the
desired output in the upper and lower 8 lanes of the output register
respectively. That's why after converting to a bitmask with
`_mm_movemask_epi8`, we mask out the upper 8 bits to ensure that they're always
zero.

This is, of course, a little wasteful. If we're looping through a huge array
would just do two lots of comparisons so we can fill two registers full of
results and then do a single `_mm_packs_epi16` to generate a 16 bit mask.

## What about with 256 and 512 bit vectors?
The 256 bit version looks much the same. It produces a 16-bit bitmask at the
end and uses `immintrin.h` and the corresponding SSE3 versions of each
operation.

```c
#include <immintrin.h>
__m256i h = _mm256_load_si256((__m256i *)ptr);
__m256i n = _mm256_set1_epi16(0x1234);
__m256i r = _mm256_cmpeq_epi16(n, h);
__m256i p = _mm256_packs_epi16(r, r);
int mask = _mm256_movemask_epi8(p) & 0xffff;
```

For AVX-512 you needn't go through the rigmarole because of the in-built
support for mask registers.

```c
__m512i h = _mm512_load_si512((const __m512i *)ptr);
__m512i n = _mm512_set1_epi16(0x1234);
__mmask32 mask  = _mm512_cmpeq_epi16_mask(n, h);
```

Easy! And it's nice and fast. Looks like you can run comparisons on an entire
cache line, 32 `uint16_t`'s at a time, with only 1-cycle of latency. And you can
run two at a time in parallel on a single core. Pretty neat.
