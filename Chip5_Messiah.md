# Chip5 Messiah

To start the next chip development, we will call it project Messiah.

---


## 13 Aug 2026

Since I have come up with the new compression algorithm, I will proceed the hardware implementation now, maybe also with the implementation of one-bit mode included.

But we need to consider the impact our threshold will have to our images, and when we convert the pixel values into binary, we need to know where 0.5 is.

And to add it on top, the pixel values are sent over in gray code.

| Rescombo| 000 |   |001|     | 010 |     | 011 |     | 100 |     | 101 |     | 110 |     | 111 |
|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|-----|
| Gray| 000 |     | 001 |     | 011 |     | 010 |     | 110 |     | 111 |     | 101 |     | 100 |
| 001 |     | 0.13|     | 0.19|     | 0.25|     | 0.31|     | 0.38|     |0.44 |     | 0.50|     |
| 010 |     | 0.14|     | 0.21|     | 0.29|     | 0.36|     | 0.43|     |0.50 |     | 0.57|     |
| 011 |     | 0.17|     | 0.25|     | 0.34|     | 0.42|     | 0.50|     |0.59 |     | 0.67|     |
| 100 |     | 0.15|     | 0.23|     | 0.31|     | 0.38|     | 0.46|     |0.54 |     | 0.62|     |
| 101 |     | 0.17|     | 0.26|     | 0.35|     | 0.43|     | 0.52|     |0.61 |     | 0.69|     |
| 110 |     | 0.18|     | 0.27|     | 0.36|     | 0.45|     | 0.53|     |0.62 |     | 0.71|     |

---

Based the chart above, we should have different binarisation rule for different combo.

Combo 001:

100 -> 1, otherwise, 0.

---

Combo 010:

101, 100 -> 1, otherwise, 0.

---

Combo 011:

111, 101, 100 -> 1, otherwise, 0.

---

Combo 100:

111, 101, 100 -> 1, otherwise, 0.

---

Combo 101:


110, 111, 101, 100 -> 1, otherwise 0.

---

Combo 110:

110, 111, 101, 100 -> 1, otherwise 0.

So Ideally, 1-bit mode with a simple logic can only be applied to combo 101 and 110 (or maybe 111).

But other configurations would require different settings.


### The addition of Gray back to Binary

So it appears that for simple 3-bit gray code, conversion back to binary is relatively simple:

```verilog

assign bin[2] = gray[2];
assign bin[1] = gray[2] ^ gray[1];
assign bin[0] = gray[2] ^ gray[1] ^ gray[0];
```

So from this expression, it seems that we need only 2 xor gate for this conversion, it should be very simple.

I should add this into my compression module...


## 20 Aug 2026

I have drafted the very first version of my compression module, now it is under testing.

So far it seems the basic functions have been tested to be okay.

Will keep on testing for the edge case.

Since I found a case when internal incrementing reached 0x7fff, it might cause problem.

First of all, we have the register update as follow:

```

clock cycle  		:  		0        1        0        1        0        1        0
data valid 			:       1        0        1        0        1        0        1
silence cnt         :      7ffd     7ffe     7ffe     7fff    7fff      0001     0002

enc ready           :       0        0        0        0        1        0        0
enc data            :       0        0        0        0      8000       0        0
```

The clock is double the rate of the data, so we evaluate the equality at cycle 0, and update silence counter at cycle 1.

Therefore when it is repeating and reaches 7fff, we will have one cycle of alarm to output 0x8000 and return to wait state like above

And we noticed that there might be new data coming while we are in the state of alarm.

What if at this very edge case, the pattern broke while we are at alarm state?


```

clock cycle  		:  		0        1        0        1        0        1        0
data valid 			:       1        0        1        0        1        0        1
pix data            :       A        A        A        A        B        B        C
																Λ
																|


silence cnt         :      7ffd     7ffe     7ffe     7fff    7fff      0000     0000

enc ready           :       0        0        0        0        1        1        0
enc data            :       0        0        0        0      8000      000B      0

```

My solution is to still evaluate while in alarm state, if that pattern breaks right at alarm state, we jump straight back in push state. In the meantime, we will reset the silence counter.

So for the decoder end, it may see:   000A, 8000, 000B.

This should stand for the case that value A has repeated 32767 times after the original A and then followed by B.



If this pattern breaks right after the alarm cycle, we shall resume the normal wait state and then jump back to push, but in this case because the silence counter managed to wrap, we will reset it back to 1 instead of 0.


```

clock cycle  		:  		0        1        0        1        0        1        0        1
data valid 			:       1        0        1        0        1        0        1        0
pix data            :       A        A        A        A        B        B        C        C

																Λ
																|


silence cnt         :      7ffe     7fff     7fff     0001    0001      0001     0000     0000

enc ready           :       0        0        1        0        0        1        1        1
enc data            :       0        0       8000      0        0       8001     000B     000C

```

So this can be easily distinguished with a different encoding:  000A, 8000, 8001, 000B, 000C

This would stand for the case that A has repeated 32767 + 1 times after the original A and then followed by B and C.


But now I have a different case where when the module is in WAIT state and experiences disabling, it will jump straight back to push without exporting the time stamp.

