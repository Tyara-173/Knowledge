# Is this Twisted?

## 問題

乱数なら全部同じ？
```Python
import random
import secrets
import os

flag = os.environ.get("FLAG", "Alpaca{dummy}")
for r in range(128):
    print(f"#{r+1}/128")
    bit = secrets.randbelow(2)
    a = [secrets.randbits(32) for _ in range(624)]
    if bit:
        random.setstate((3, (*a, 624), None))
        a = [random.getrandbits(32) for _ in range(624)]
    print(a)
    if input("guess: ") != str(bit):
        print(":(")
        exit(1)
print("flag:", flag)
```

[問題リンク](https://alpacahack.com/daily-bside/challenges/is-this-twisted)

## 概要
624個の乱数が与えられる．それが`secrets.randbits(32)`によって生成された乱数か，`random.setstate((3, (*a, 624), None))`をした後に`random.getrandbits(32)`で生成された乱数かを当てたい．

## 解法
Pythonのrandomはメルセンヌ・ツイスタを使用しているため，624個の乱数がメルセンヌ・ツイスタによって生成されたものかを当てたい．

`random.setstate((3, (*a, 624), None))`が気になるので[`random.setstate()`の実装](https://github.com/python/cpython/blob/main/Modules/_randommodule.c#L455)を覗くと，内部状態`mt`を`a`に置き換え，`self->index`を`624`に置き換えている．ここで`self->index`を`624`に置き換えているのが重要で，これにより最初に呼び出された`random.getrandbits(32)`で強制的にTwist演算が発生している．

[Twist演算の実装](https://github.com/python/cpython/blob/main/Modules/_randommodule.c#L135)を覗くと，

```C++
y = (mt[N-1]&UPPER_MASK)|(mt[0]&LOWER_MASK);
mt[N-1] = mt[M-1] ^ (y >> 1) ^ mag01[y & 0x1U];
```

の部分は`mt[N-1]`以外既知の値で構成されており，`mt[N-1]`も`UPPER_MASK=0b10000000000000000000000000000000`との論理積により1bitしか使われないため再現が可能である．内部状態`mt`も与えられた乱数を逆Temperingすることで得られるため，与えられた乱数を逆Temperingした`mt[N-1]`と，`mt[M-1]`と`mt[0]`を用いて再現した値（`(mt[N-1]&UPPER_MASK)=0`か`(mt[N-1]&UPPER_MASK)!=0`かの2通り）が一致するかを判定することによりこの問題を解くことができる．

（ごく低確率で`bit=0`でありながら上記の値が一致することもあるが，十分無視できる確率なので無視）

ソルバは以下の通り
```Python
def untemper(y):
    y ^= (y >> 18)
    y ^= (y << 15) & 0xefc60000
    for _ in range(7):
        y ^= (y << 7) & 0x9d2c5680
    y ^= (y >> 11) ^ (y >> 22)
    return y

LOWER_MASK = 0x7fffffff
UPPER_MASK = 0x80000000
MATRIX_A = 0x9908b0df

def guess_bit(a):
    a_0 = untemper(a[0])
    a_396 = untemper(a[396])
    a_623 = untemper(a[623])

    t1 = (a_396 ^ (MATRIX_A if (a_0 & 1) else 0) ^ ((a_0 & LOWER_MASK) >> 1)) & 0xffffffff
    t2 = (a_396 ^ (MATRIX_A if (a_0 & 1) else 0) ^ (((a_0 & LOWER_MASK) | UPPER_MASK) >> 1)) & 0xffffffff
    if a_623 == t1 or a_623 == t2:
        return "1"
    return "0"

from pwn import remote

io = remote("host",port)

for _ in range(128):
    print(io.recvline())
    a = list(map(int,io.recvline().decode().strip()[1:-1].split(', ')))
    print(a)
    print(guess_bit(a))
    io.sendline(guess_bit(a).encode())

print(io.recvline())
```