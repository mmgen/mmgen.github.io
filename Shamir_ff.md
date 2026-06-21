# Shamir’s Secret Sharing Scheme

### An ultra-compact implementation using finite field arithmetic

This code is a modification of the 21-line [toy implementation][t], with the
only significant difference being the replacement of arithmetic operations by
their finite-field counterparts.  Addition, subtraction and multiplication are
now modulo prime *P,* while division is replaced by multiplication by the
modular inverse, computed via Euler’s method.

Three values of *P* are provided, for 128-, 256- and 512-bit secrets.

This implementation is suitable for integration into production code.

For comparison, refer to the reference code on [Wikipedia][w].

```python
#!/usr/bin/env python

# An ultra-compact implementation of Shamir’s Secret Sharing Scheme using finite
# field arithmetic.  Suitable for integration into production code.

# Copyright 2026 The MMGen Project (mmgen.org, github.com/mmgen, mmgen.github.io)
# License: GPLv3

# No sanity checks on input are performed.  It’s up to the implementer to ensure
# that input data is of correct type, secrets and nonces are strongly random and
# within the allowable range, share numbers are unique, and so forth.

# Largest strong safe primes for their respective bit lengths:
p_128 = 2 ** 128 - 94697
p_256 = 2 ** 256 - 315053
p_512 = 2 ** 512 - 16719353

p = p_256

class Coefficient:

	def __init__(self, idx, share, t):
		self.idx = idx
		self.data = [pow(share[0], exp, p) for exp in range(t)] + [share[1]]

	def update(self, others, t):
		for other in others:
			other_mul = self.data[other.idx]
			for n in range(t + 1):
				if other.data[n] != 0:
					self.data[n] = (self.data[n] - other_mul * other.data[n]) % p
		if self.data[self.idx] != 1:
			self_mul_inv = pow(self.data[self.idx], -1, p) # multiplicative inverse by Euler’s method
			for n in range(t + 1):
				self.data[n] = (self.data[n] * self_mul_inv) % p

def create_share(x, a_arr):
	return (x, sum((a * pow(x, exp, p)) % p for exp, a in enumerate(a_arr)) % p)

def recover_secret(shares):
	t = len(shares)
	ccs = [Coefficient(n, s, t) for n, s in enumerate(shares)]
	for i in range(t - 1):
		ccs[i + 1].update((ccs[k] for k in range(i + 1)), t)
		for j in range(i + 1):
			ccs[j].update([ccs[i + 1]], t)
	return [cc.data[-1] for cc in ccs]

if __name__ == '__main__':

	import os
	from pprint import pformat

	max_threshold = 40    # arbitrary sane limit
	max_share_num = 1024  # arbitrary sane limit

	for byte_len, p in {16: p_128, 32: p_256, 64: p_512}.items():

		print(f'\nTesting byte length: {byte_len}')

		test_data = ((
				'worked example data: (s=4, a=3, b=2, c=5, threshold=4, shares 1, 2, 5, 7)',
				[4, 3, 2, 5],
				[1, 2, 5, 7]
			), (
				'random secret and nonces, threshold=5',
				[int.from_bytes(os.urandom(byte_len)) for n in range(5)],
				[137, 16, 1, 235, 6]
			), (
				'zero nonce, max nonce, threshold=20',
				[3, 2, 0, 6, 1, 4, 7, 9, 8, 16, 14, 17, 19, 11, 13, 10, 27, 23, 36, p-1],
				[32, 33, 137, 16, 1, 123, 6, 9, 4, 2, 17, 129, 777, 72, 99, 12, 13, 14, 20, 19]
			), (
				'max secret, max threshold, max share_num',
				[p-(n+1) for n in range(max_threshold)],
				[max_share_num-n for n in range(max_threshold)]
			), (
				'zero secret',
				[0, 3], [1, 9]))

		for desc, a_arr, share_nums in test_data:
			print(f'\nTesting: {desc}')

			assert len(a_arr) <= max_threshold

			assert len(share_nums) == len(a_arr)

			for a in a_arr:
				assert isinstance(a, int) and a < p

			for s in share_nums:
				assert isinstance(s, int) and s and s <= max_share_num

			assert len(set(a_arr)) == len(a_arr) # duplicate nonce check (could indicate bad PRNG)

			assert len(set(share_nums)) == len(share_nums) # duplicate share number check

			print('\nThreshold:', len(a_arr))
			print('Secret:', a_arr[0])

			shares = [create_share(num, a_arr) for num in share_nums]

			for s in shares:
				assert s[1] < p

			print('Shares:', pformat(shares))

			res = recover_secret(shares)

			assert res == a_arr

			print('Recovered secret:', res[0])
```

[p]: images/shamir/parabolas.webp
[e]: images/shamir/example.svg
[w]: https://en.wikipedia.org/wiki/Shamir's_secret_sharing#Python_code
[t]: code/shamir.py
