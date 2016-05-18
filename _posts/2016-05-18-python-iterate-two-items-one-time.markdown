---
layout: post
title:  "Python: Iterate over a list 2 items at a time"
date:   2016-05-18
tags:   [python]
---

```python
for (c1, c2) in zip(s[0::2], s[1::2]):
    print c1, c2
```

No-copy version:

```python
from itertools import izip, islice

for (c1, c2) in izip(islice(s, 0, None, 2), islice(s, 1, None, 2)):
    print c1, c2
```

Hex string to bin string:

```python
s = 'bd2866849ae03b59861d40dd8bb0d4'
''.join([chr(int(''.join(two), 16)) for two in zip(s[0::2], s[1::2])])
```

Bin string to hex string:

```python
binstr = '\xbd(f\x84\x9a\xe0;Y\x86\x1d@\xdd\x8b\xb0\xd4'
''.join(('{0:02x}'.format(n) for n in (ord(c) for c in binstr)))
```