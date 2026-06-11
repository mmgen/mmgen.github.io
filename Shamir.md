![points on parabolas][p]

# Shamir’s Secret Sharing Scheme

### A worked example and ultra-compact implementation in 21 lines of Python code

![Shamir’s Secret Sharing Scheme: a worked example][e]

*Typesetting by LibreOffice Math*

The code started out by following the worked example and ended up as a generalized solution.

Thus the implementation is self-contained and relies on nothing but basic arithmetic: note the absence of Lagrange interpolations and Python module imports.

```python
#!/usr/bin/env python

# A toy implementation of Shamir’s Secret Sharing Scheme
# For demonstration purposes only, not for production use

# Copyright 2026 The MMGen Project (mmgen.org, github.com/mmgen, mmgen.github.io)
# License: GPLv3

class Coefficient:

	def __init__(self, idx, share, t):
		self.idx = idx
		self.data = [share[0] ** exp for exp in range(t)] + [share[1]]

	def update(self, others, t):
		for other in others:
			mul = self.data[other.idx]
			for n in range(t + 1):
				self.data[n] -= mul * other.data[n]
		div = self.data[self.idx]
		for n in range(t + 1):
			self.data[n] //= div

def create_share(x, a_arr):
	return (x, sum(a * x ** exp for exp, a in enumerate(a_arr)))

def recover(shares, t):
	ccs = [Coefficient(n, s, t) for n, s in enumerate(shares)]
	for i in range(t - 1):
		ccs[i + 1].update((ccs[k] for k in range(i + 1)), t)
		for j in range(i + 1):
			ccs[j].update([ccs[i + 1]], t)
	return [cc.data[-1] for cc in ccs]

if __name__ == '__main__':

	a_arr = [4, 3, 2, 5]      # 4 arbitrary integers (secret + 3 nonces, threshold=4)
	share_nums = [1, 2, 5, 7] # 4 arbitrary integers

	a_arr = [1432059, 3319, 2207, 15634, 557, 987425, 83500225] # threshold=7
	share_nums = [13371, 4, 1, 2, 6, 7, 889]

	assert len(share_nums) == len(a_arr)

	print('Threshold:', len(a_arr))
	print('Secret:', a_arr[0])

	shares = [create_share(num, a_arr) for num in share_nums]

	from pprint import pformat
	print('Shares:', pformat(shares))

	res = recover(shares, len(a_arr))

	assert res == a_arr

	print('Recovered secret:', res[0])
```

Output with worked example data:

```text
Threshold: 4
Secret: 4
Shares: [(1, 14), (2, 58), (5, 694), (7, 1838)]
Recovered secret: 4
```

Output with alternate data:

```text
Threshold: 7
Secret: 1432059
Shares: [(13371, 477168056575161181455940409240886),
 (4, 343030668615),
 (1, 85941426),
 (2, 5377193509),
 (6, 3903470344641),
 (7, 9840321886254),
 (889, 41219620449806768439288930)]
Recovered secret: 1432059
```

[p]: images/shamir/parabolas.webp
[e]: images/shamir/example.png
