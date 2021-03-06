
# Notes on Radix Sorting and Implementation

_TODO: These notes (and code) are incomplete. My goal is to eventually provide a step-by-step
guide and introduction to a simple radix sort implementation, and try to cover
the basic issues which are sometimes glossed over in text books and courses.
Furthermore, WHILE THIS NOTE PERSISTS, I MAY FORCE PUSH TO MASTER_

These are my notes on engineering a [radix sort](https://en.wikipedia.org/wiki/Radix_sort).

The ideas behind radix sorting are not new in any way, but seems to have become,
if not forgotten, so at least under-utilized, as over the years _quicksort_ became
the go-to "default" sorting algorithm.

Unless otherwise specified, this code is written foremost to be clear and easy to understand.

All code is provided under the [MIT License](LICENSE).

## Motivation

This code can sort 40 million 32-bit integers in under half a second using a single
core of an [Intel i5-3570T](https://ark.intel.com/products/65521/Intel-Core-i5-3570T-Processor-6M-Cache-up-to-3_30-GHz),
a low-TDP CPU from 2012 using DDR3-1333. `std::sort` requires ~3.5s for the same task (with the
caveat that it's _in-place_).

## From the top; Counting sort

The simplest way to sort an array of integers, without comparing elements, is to simply count
how many there are of each unique integer, and use those counts to write the result.

This is the most basic [counting sort](https://en.wikipedia.org/wiki/Counting_sort).

[Listing 1](counting_sort_8.c):

```c
void counting_sort_8(uint8_t *arr, size_t n)
{
	size_t cnt[256] = { 0 };
	size_t i;

	// Count number of occurences of each octet.
	for (i = 0 ; i < n ; ++i) {
		cnt[arr[i]]++;
	}

	// Write cnt_a a's to the array in order.
	i = 0;
	for (size_t a = 0 ; a < 256 ; ++a) {
		while (cnt[a]--)
			arr[i++] = a;
	}
}
```

You could easily extend this code to work with 16-bit values, but if you try to push much further the drawbacks
of this counting sort become fairly obvious; you need a location to store the count for each unique
integer. For 8- and 16-bit numbers this would amount to `2^8*4`=1KiB and `2^16*4`=256KiB of memory. For
32-bit integers, it'd require `2^32*4`=16GiB of memory. Multiply by two if you need 64- instead of 32-bit counters.

As the wikipedia page explains, it's really the size of the keys involved that matters. Some
implementations can be seen scanning the input data to determine and allocate just enough entries to fit
`max(key) - min(key) + 1` keys. However, if you do this you will most likely have to consider what to do if the
input domain is too wide to handle, which is not a good position to be in. In practice you would _never_ want to
fail on some inputs, which makes that type of implementation not very useful.

As presented, this sort is _in-place_, but since -- in addition to not comparing elements -- it's not moving
any elements either, it doesn't really make sense to think of it as being _stable_ or  _unstable_.

To get us closer to radix sorting, we now need to consider a slightly more general variant where we're, at
least conceptually, rearranging input elements:

[Listing 2](counting_sort_8s.c):

```c
void counting_sort_8s(uint8_t *arr, uint8_t *aux, size_t n)
{
	size_t cnt[256] = { 0 };
	size_t i;

	// Count number of occurences of each octet.
	for (i = 0 ; i < n ; ++i) {
		cnt[arr[i]]++;
	}

	// Calculate prefix sums.
	size_t a = 0;
	for (int j = 0 ; j < 256 ; ++j) {
		size_t b = cnt[j];
		cnt[j] = a;
		a += b;
	}

	// Sort elements
	for (i = 0 ; i < n ; ++i) {
		// Get the key for the current entry.
		uint8_t k = arr[i];
		// Find the location this entry goes into in the output array.
		size_t dst = cnt[k];
		// Copy the current entry into the right place.
		aux[dst] = arr[i];
		// Make it so that the next 'k' will be written after this one.
		// Since we process source entries in increasing order, this makes us a stable sort.
		cnt[k]++;
	}
}
```

We have introduced a separate output array, which means we are no longer _in-place_. This auxillary
array is required; the algorithm would break if we tried to write directly into the input array.

However, the _main_ difference between this and the first variant is that we're no longer directly writing the
output from the counts. Instead the counts are re-processed into a series of prefix sums (sometimes called a _scan_) in the
second loop. This gives us, for each input value, its first location in the sorted output array, i.e
the value of `cnt[j]` tells us the array index at which to write the first _j_ to the output.

For instance, `cnt[0]` will always be zero, because any `0` will always end up in the first
position in the output. `cnt[1]` will contain how many zeroes precede the first `1`, `cnt[2]` will
contain how many zeroes _and_ ones precede the first `2`, and so on.

In the sorting loop, we look up the output location for the key of the current entry, and write the
entry there. We then increase the count of the prefix sum by one, which guarantees that the next same-keyed entry
is written just after this one.

Because we are processing the input entries in order, from the lowest to the highest index, and preserving
this order when we write them out, this sort is in essence _stable_. That said, it's a bit of a pointless distinction
since we're treating each input entry, as a whole, as the key.

With a few basic modifications, we arrive at

[Listing 3](counting_sort_rec_sk.c):

```c
void counting_sort_rec_sk(struct sortrec *arr, struct sortrec *aux, size_t n)
{
	size_t cnt[256] = { 0 };
	size_t i;

	// Count number of occurences of each key.
	for (i = 0 ; i < n ; ++i) {
		uint8_t k = key_of(arr + i);
		cnt[k]++;
	}

	// Calculate prefix sums.
	size_t a = 0;
	for (int j = 0 ; j < 256 ; ++j) {
		size_t b = cnt[j];
		cnt[j] = a;
		a += b;
	}

	// Sort elements
	for (i = 0 ; i < n ; ++i) {
		// Get the key for the current entry.
		uint8_t k = key_of(arr + i);
		size_t dst = cnt[k];
		aux[dst] = arr[i];
		cnt[k]++;
	}
}
```

We are now sorting an array of `struct sortrec`, not an array of octets. The name and the type of the struct
here is arbitary; it's only used to show that we're not restricted to sorting _arrays of integers_.

The primary modification to the sorting function is the small addition of a function `key_of()`, which returns
the key for a given record.

The main insight you should take away from this is that if the things we're sorting aren't themselves the
keys but some composed type, we just need some way to _retrieve_ or _derive_ a key for each entry instead.

We're still restricted to integer keys. We rely on there being some sort of mapping from our records (or _entries_)
to the integers which orders the records the way we require.

Here we still use a single octet inside the `struct sortrec` as the key. Associated with each key
is a short string. This allows us to demonstrate *a)* that sorting keys with associated data is not a problem,
and *b)* that the sort is indeed stable.

Running the full program demonstrates that each _like-key_ is output in the same order it came in the
input array, i.e the sort is _stable_.

```console
$ ./counting_sort_rec_sk
00000000: 1 -> 1
00000001: 2 -> 2
00000002: 3 -> 3
00000003: 45 -> 1st 45
00000004: 45 -> 2nd 45
00000005: 45 -> 3rd 45
00000006: 255 -> 1st 255
00000007: 255 -> 2nd 255
```

Now we are ready to take the step from counting sorts to radix sorts.

## All together now; Radix sort

_NOTE: Very much work in progress._

A radix sort works by looking at some portion of a key, sorting all entries based on
that portion, then taking another pass and look at the next portion, and so on until
the whole of the keys, or as much as is necessary, have been processed.

Some texts describe this as looking at the individual digits of an integer key, which
you can process digit-by-digit via a series of modulo (remainder) and division operations
with the number 10, the base or [radix](https://en.wikipedia.org/wiki/Radix).

In a computer we deal with bits, which means the base is inherently two, not ten.

Because working with individual bits is in some sense "slow", we group multiple of
them together into units that are either faster and/or more convenient to work with.
One such grouping is into strings of eight bits, called octets or [bytes](https://en.wikipedia.org/wiki/Byte).

An octet, `2^8` bits, can represent `256` different values. In other words, just
as processing a base-10 number digit-by-digit is operating in radix-10, processing
a binary number in chunks of eight bits means operating in radix-256.

Since we're not going to do any math with the keys, it may help to conceptually
consider them simply as _bit-strings_ instead of numbers. This gets rid of some
baggage which isn't useful to the task at hand.

Because we're using a computer and operate on bits, instead of division and modulo
operations, we use _bit-shifts_ and _bit-masking_.

Below is a table of random 32-bit keys written out in [hexadecimal](https://en.wikipedia.org/wiki/Hexadecimal), or base-16,
which is a convenient notation for the task at hand. In hexadecimal
a group of four bits is represented with a symbol (digit) from 0-F, and
consequently a group of eight bits is represented by two such symbols.

 | 32-bit key | A  | B  | C  | D  |
 | :--------- | -- | -- | -- | -- |
 | 7A8F97A4   | 7A | 8F | 97 | A4 |
 | F728B2E2   | F7 | 28 | B2 | E2 |
 | 517833CD   | 51 | 78 | 33 | CD |
 | 9332B72F   | 93 | 32 | B7 | 2F |
 | A35138CD   | A3 | 51 | 38 | CD |
 | BBAD9DAF   | BB | AD | 9D | AF |
 | B2667C54   | B2 | 66 | 7C | 54 |
 | 8C8E59A6   | 8C | 8E | 59 | A6 |

If you consider this table, with the four 8-bit wide columns marked *A* through *D*, there's a choice to be made;
if we're going to process these keys top to bottom, one column at a time, _in which order do we process the columns_?

This choice gives rise to the _two main classes_ of radix sorts, those that are _Least Significant Bits_ (LSB)
first and those that are _Most Significant Bits_ (MSB) first. Sometimes 'digit' is substituted for 'bits',
it's the same thing.

If you have prior experience you may already know that, based on the material presented so far,
we're going down the LSB path, meaning we'll process the columns from right to left; D, C, B, A.

In our counting sorts, the key width and the radix (or column) width were the same; 8-bits.
In a radix sort the column width will be less or equal to the key width, and in a LSB radix sort
we'll be forced to make multiple passes over the input to make up the difference. The wider
our radix the more memory (to track counts), but fewer passes we'll need. This is the tradeoff.

The assertion then, and we will demonstrate this to be true, is that if we apply counting sort
by column *D*, and then apply counting sort on that result by column *C*, and so forth, after the last
column (*A*) is processed, our data will be sorted and this sort is stable.

[Listing 3](radix_sort_u32.c):

```c
void radix_sort_u32(struct sortrec *arr, struct sortrec *aux, size_t n)
{
	size_t cnt0[256] = { 0 };
	size_t cnt1[256] = { 0 };
	size_t cnt2[256] = { 0 };
	size_t cnt3[256] = { 0 };
	size_t i;

	// Generate histograms
	for (i = 0 ; i < n ; ++i) {
		uint32_t k = key_of(arr + i);

		uint8_t k0 = (k >> 0) & 0xFF;
		uint8_t k1 = (k >> 8) & 0xFF;
		uint8_t k2 = (k >> 16) & 0xFF;
		uint8_t k3 = (k >> 24) & 0xFF;

		++cnt0[k0];
		++cnt1[k1];
		++cnt2[k2];
		++cnt3[k3];
	}

	// Calculate prefix sums.
	size_t a0 = 0;
	size_t a1 = 0;
	size_t a2 = 0;
	size_t a3 = 0;
	for (int j = 0 ; j < 256 ; ++j) {
		size_t b0 = cnt0[j];
		size_t b1 = cnt1[j];
		size_t b2 = cnt2[j];
		size_t b3 = cnt3[j];
		cnt0[j] = a0;
		cnt1[j] = a1;
		cnt2[j] = a2;
		cnt3[j] = a3;
		a0 += b0;
		a1 += b1;
		a2 += b2;
		a3 += b3;
	}

	// Sort in four passes from LSB to MSB
	for (i = 0 ; i < n ; ++i) {
		uint32_t k = key_of(arr + i);
		uint8_t k0 = (k >> 0) & 0xFF;
		size_t dst = cnt0[k0]++;
		aux[dst] = arr[i];
	}
	SWAP(arr, aux);

	for (i = 0 ; i < n ; ++i) {
		uint32_t k = key_of(arr + i);
		uint8_t k1 = (k >> 8) & 0xFF;
		size_t dst = cnt1[k1]++;
		aux[dst] = arr[i];
	}
	SWAP(arr, aux);

	for (i = 0 ; i < n ; ++i) {
		uint32_t k = key_of(arr + i);
		uint8_t k2 = (k >> 16) & 0xFF;
		size_t dst = cnt2[k2]++;
		aux[dst] = arr[i];
	}
	SWAP(arr, aux);

	for (i = 0 ; i < n ; ++i) {
		uint32_t k = key_of(arr + i);
		uint8_t k3 = (k >> 24) & 0xFF;
		size_t dst = cnt3[k3]++;
		aux[dst] = arr[i];
	}
}
```

The function `radix_sort_u32()` builds on `counting_sort_rec_sk()` in a straight-forward manner
by introducing four counting sort passes. It's _unrolled_ by design, to show off the pattern.

The four histograms are generated in one pass through the input. These are re-processed into prefix sums
in a separate pass. A speed vs memory trade-off can be had by not pre-computing all the histograms.

We then sort columns *D* through *A*, swapping (a.k.a ping-ponging) the
input and output buffer between the passes.

After the 4:th (final) sorting pass, since this is an even number, the final result is available
in the input buffer.

## Key derivation; Sort order and floats

So far we have been using an unsigned integer as the key, which we've sorted
in ascending order. What if key we want to sort on is some other type, or if
we want to sort in descending order?

First off, for a LSB radix sort, the keys really need to be the same width,
and narrower is better to keep the number of passes down. This means that
an LSB implemention is not well suited to sort character strings of any significant
length. For those, a MSB radix sort is better.

For sorting floats, doubles or other fixed-width keys, or changing the sort order,
we can still use the LSB implementation as is. The trick is to map the bit-patterns of the key
we have onto the unsigned integers.

If we want to sort unsigned integers in _descending_ order, have the key-derivation
function return the bitwise inverse of the key:

```c
	return ~key; // unsigned (desc)
```

To treat the key as a signed integer, we need to manipulate the sign-bit,
since by default this is set for negative numbers, meaning they will appear
at the end of the result.

```c
	return key ^ 0x80000000; // signed 32-bit (asc)
```

These can be combined to handle signed integers in descending order:

```c
	return ~(key ^ 0x80000000); // signed 32-bit (desc)
```

To sort IEEE 754 single-precision (32-bit) floats (a.k.a _binary32_) in their natural order we need to invert the key if the sign-bit is set, else we need to flip the sign bit. (via [Radix Tricks](http://stereopsis.com/radix.html)):

```c
	return key ^ (-((uint32_t)key >> 31) | 0x80000000); // 32-bit float
```

Example for sorting `{ 128.0f, 646464.0f, 0.0f, -0.0f, -0.5f, 0.5f, -128.0f, -INFINITY, NAN, INFINITY }`:

```console
00000000: ff800000 -inf
00000001: c3000000 -128.000000
00000002: bf000000 -0.500000
00000003: 80000000 -0.000000
00000004: 00000000 0.000000
00000005: 3f000000 0.500000
00000006: 43000000 128.000000
00000007: 491dd400 646464.000000
00000008: 7f800000 inf
00000009: 7fc00000 nan
```

These of course extends naturally to 64-bit keys also.

Performance-wise it's probably not worth worrying about the extra cost of the
repeated key-derivation, but if you do you could still generalize a sort by using
two functions; Rewrite the input with _into_ once, do the sort on the transformed input,
and then transform it back by applying _outof_.

For very specific sorts, where performance is of the outmost importance, it's certainly
possible the change the underlying code instead of manipulating the key. You can reverse
the sort-order by reversing the prefix sum, or simply reading the final result backwards
if possible.

The 2000-era [Radix Sort Revisited](http://codercorner.com/RadixSortRevisited.htm) presents
a more direct way to change the code to handle floats.


## Optimizations

_TODO: This section very much a work in progress/TBD thing_

In this section I'll talk about some optimizations that I have tried, or
that may be worth investigating.

### Hybrids

I have observed a few different radix sort implementations, and some of them have
a larger _fixed overhead_ than others. This gives rise to the idea of hybrid sorts,
where you redirect small workloads (e.g N < 100) to say an _insertion sort_.
This is often a part of MSB radix sorts.

It's also possible to do one MSB pass and then LSB sort the sub-results. This saves
memory and memory management from doing all MSB passes, and should improve cache
locality for the LSB passes. That said, in the one implementation I've tried, the
fixed overhead was quite high. (_TODO: determine exact reason_)

### Pre-sorted detection

Since we have to scan once through the input to build our histograms, it's
relatively easy to add code there to detect if the input is already sorted/reverse-sorted,
and special case that.

One way to implement it is to initialize a variable to the number of elements
to be sorted, and decrement it every time the current element is _less-than-or-equal_
to the next one, care of bounds. After a pass through the input, you would have
a count of the number of elements that need to be sorted. If this number is less
than two, you're done.

If there's a user-defined key-derivation function, apply it to the values being
compared.

Pushing further, say trying to detect already sorted columns, didn't seem worth the
effort in my experiments. You don't want to add too many conditionals to the
histogram loop.

### Column skipping

If every radix for a column is the same, then the sort loop for that column will
simply be a copy from one buffer to another, which is a waste.

Fortunately detecting this is easy, you don't even have to scan through the
histograms. Simply sample the key of the first element. If any of its radixes has
a histogram count equal to the total number of entries to sort, that column can be
skipped.

```c
	int cols[4];
	int ncols = 0;
	uint32_t key0 = key_of(*src); // sample first entry
	uint8_t k0 = (key0 >> 0);
	uint8_t k1 = (key0 >> 8);
	[...]
	if (cnt0[k0] != num_entries)
		cols[ncols++] = 0;
	if (cnt1[k1] != num_entries)
		cols[ncols++] = 1;
	[...]
	// .. here the cols array contains the index of the ncols columns we must process.
```

Skipping columns has the side-effect of the final sorted result ending up in either the
original input buffer or the auxillary buffer, depending on whether you sorted an even
or odd number of columns. I prefer to have the sort function return the pointer
to the result, rather than add a copy step.

### Histogram memory

Pre-calulating the histograms for multiple radixes at one time is not strictly necessary
since the counts aren't affected by the sorting, Memory can be saved by calculating only
the histogram(s) you need for the next pass or two.

In the simplest case this results in one extra pass through the data per radix, minus
one, but you could try and put the counting inside the sort loop _prior_ to the one
where you need the prefix sums, and ping-pong two histogram buffers.

### Prefetching

Working primarily on IA-32 and AMD64/x86-64 CPUs, I've never had a good experience with
adding manual prefetching (i.e `__builtin_prefetch`).

[Michael Herf](http://stereopsis.com/radix.html) reports a "25% speedup" from adding `_mm_prefetch` calls
to his code that was running on a Pentium 3.

I'll put this in the *TBD* column for now.

### Vectorized histogramming

The vector instructions afforded by x86-64 (SSE, AVX, AVX2) does not seem to provide a path
to a vectorized implementation that is worth it in practice. The issue seems to be poor
gather/scatter support. I'm looking forward to be proven wrong about this in the future.

For other architectures this may be more viable.

### Wider or narrower radix

Using multiples of eight bits for the radix is convenient, but not required. If we do,
we limit ourselves to 8-bit or 16-bit wide radixes in practice. Using an intermediate size such as 11-bits
with three passes saves us one pass for every 32-bits of key, but also makes it less likely
for the column skipping to kick in and may add some masking operations.

Going narrower could allow us to skip more columns in common workloads. There's definitely
room for experimentation in this area, maybe even trying non-uniformly wide radixes (8-16-8) or
dynamic selection of radix widths.

### CPU Bugs

The numbers quoted in the [motivation](#motivation) were taken on intel microcode 06-3a-09 version 0x1c. A later
update to 0x1f (dated 2018-02-07), issued to mitigate [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability))
and [Meltdown](https://en.wikipedia.org/wiki/Meltdown_(security_vulnerability)), decreased performance _significantly_.
The ~460ms radix sort now took ~630ms, and the `std::sort` went from 3.5s to 5s.

_TODO: Determine if huge-pages helps_

### Compiler issues

_TODO_: Very suspectible to compiler optimization (check << 3 vs * 8 again), GCC7.3 vs GCC5...

## C++ Implementation

_TODO_

[radix.cpp](radix.cpp)

## MSB - The other path

_TODO_

## Miscellaneous

* Not going over asymptotics, but mention latency.
* Hybrid sorting.
* Add note on non-parallel histogramming to save memory.
* Asserting on histogram counter overflow.

## Resources

* Pierre Terdiman, "[Radix Sort Revisited](http://codercorner.com/RadixSortRevisited.htm)", 2000.
* Michael Herf, "[Radix Tricks](http://stereopsis.com/radix.html), 2001.

