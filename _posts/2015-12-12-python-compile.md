---
layout: post
title: "Python 编译与反编译"
comments: true
categories:
- python
---

test.py
------

```
def main():
    a = 1
    b = 2
    c = a + b
    print c

if __name__ == '__main__':
    main()
```

py -> pyc
---------

`python -m py_compile test.py`

```
hexdump -C test.pyc
00000000  03 f3 0d 0a 32 d7 6b 56  63 00 00 00 00 00 00 00  |....2.kVc.......|
00000010  00 02 00 00 00 40 00 00  00 73 23 00 00 00 64 00  |.....@...s#...d.|
00000020  00 84 00 00 5a 00 00 65  01 00 64 01 00 6b 02 00  |....Z..e..d..k..|
00000030  72 1f 00 65 00 00 83 00  00 01 6e 00 00 64 02 00  |r..e......n..d..|
00000040  53 28 03 00 00 00 63 00  00 00 00 03 00 00 00 02  |S(....c.........|
00000050  00 00 00 43 00 00 00 73  1f 00 00 00 64 01 00 7d  |...C...s....d..}|
00000060  00 00 64 02 00 7d 01 00  7c 00 00 7c 01 00 17 7d  |..d..}..|..|...}|
00000070  02 00 7c 02 00 47 48 64  00 00 53 28 03 00 00 00  |..|..GHd..S(....|
00000080  4e 69 01 00 00 00 69 02  00 00 00 28 00 00 00 00  |Ni....i....(....|
00000090  28 03 00 00 00 74 01 00  00 00 61 74 01 00 00 00  |(....t....at....|
000000a0  62 74 01 00 00 00 63 28  00 00 00 00 28 00 00 00  |bt....c(....(...|
000000b0  00 73 07 00 00 00 74 65  73 74 2e 70 79 74 04 00  |.s....test.pyt..|
000000c0  00 00 6d 61 69 6e 01 00  00 00 73 08 00 00 00 00  |..main....s.....|
000000d0  01 06 01 06 01 0a 01 74  08 00 00 00 5f 5f 6d 61  |.......t....__ma|
000000e0  69 6e 5f 5f 4e 28 02 00  00 00 52 03 00 00 00 74  |in__N(....R....t|
000000f0  08 00 00 00 5f 5f 6e 61  6d 65 5f 5f 28 00 00 00  |....__name__(...|
00000100  00 28 00 00 00 00 28 00  00 00 00 73 07 00 00 00  |.(....(....s....|
00000110  74 65 73 74 2e 70 79 74  08 00 00 00 3c 6d 6f 64  |test.pyt....<mod|
00000120  75 6c 65 3e 01 00 00 00  73 04 00 00 00 09 06 0c  |ule>....s.......|
00000130  01                                                |.|
00000131
```

pyc -> py
---------

`pip install uncompyle2`

`uncompyle2 test.pyc > 1.py`

```
# 2015.12.12 16:14:33 CST
#Embedded file name: test.py


def main():
    a = 1
    b = 2
    c = a + b
    print c


if __name__ == '__main__':
    main()
+++ okay decompyling test.pyc
# decompiled 1 files: 1 okay, 0 failed, 0 verify failed
# 2015.12.12 16:14:33 CST
```
