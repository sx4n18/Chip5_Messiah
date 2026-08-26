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

## 24 Aug 2026

I have now finished building the compression module with the following logic and how the corner cases are handled:

---

### **3-bit mode**

1. Detecting pattern in 3-bit until it repeats. Unless it exports the raw data: {0, data}
2. When it starts repeating, meaning the previous export of data matches current data, repetition starts: e.g. B, A, A, A -> {0,B}, {0,A}, ...
3. When the repetition starts, the internal 15-bit counter shall also start incrementing. example for above: : 0, 0, 1, 2
4. Export both the new pattern and incremented count when pattern breaks.
5. If during silence, the internal count has reached 7fff, while the pattern continues, we reset the counter back to 1 while exporting special packet: 8000
6. If during silence, the internal counter is 7fff, and the new pattern stopped, we reset the counter back to 0.

Therefore for the following example:

```

clock cycle  		:  		0        1        0        1        0        1        0
data valid 			:       1        0        1        0        1        0        1
pix data            :       A        A        A        A        B        B        C
																Λ
																|


silence cnt         :      7ffd ->  7ffe     7ffe ->  7fff    7fff  ->  0000     0000

enc ready           :       0        0        0        0        0        1        1
enc data            :       0        0        0        0        0       ffff     000B

```

We shall export this sequence as: 0x000A, 0xFFFF, 0x000B


As for this edge case:

```

clock cycle  		:  		0        1        0        1        0        1        0        1
data valid 			:       1        0        1        0        1        0        1        0
pix data            :       A        A        A        A        B        B        C        C

																Λ
																|


silence cnt         :      7ffe  -> 7fff     7fff ->  0001    0001      0001     0000     0000

enc ready           :       0        0        0        1        0        1        1        1
enc data            :       0        0        0       8000      0       8001     000B     000C

```


For decoder, we shall have: 0x000A, 0x8000, 0x8001, 0x000B ...


---

### **1-bit mode**

This mode works similarly, except it is treating all the input pixel values in 1-bit, same rules apply.

Evaluation only happens at the 3rd cycle of the incoming pixel values.

The binarisation follow the rule to simply make a clean cut at: **100**.

Therefore, it only works in this way:

000, 001, 010, 011, 100  -->  0

101, 110, 111            -->  1

The value mentioned above is in binary.



Following the completion of the module, I will proceed with the synthesis, and make a comparison of how much logic we need for this one compared to previous implementation.

The initial synthesis showed that this new compression unit has a cell area of about 7287 with cell count of 328.

```
============================================================
  Generated by:           Genus(TM) Synthesis Solution 21.14-s082_1
  Generated on:           Aug 24 2026  04:17:35 pm
  Module:                 Row_encode_5P
  Operating conditions:   _nominal_ (balanced_tree)
  Wireload mode:          top
  Area mode:              timing library
============================================================

   Instance   Module  Cell Count  Cell Area  Net Area   Total Area  Wireload  
------------------------------------------------------------------------------
Row_encode_5P                328   7287.034     0.000     7287.034   G5K (S)  
  (S) = wireload was automatically selected
  (T) = wireload mode is 'top'
```

By comparison, our old implementation has a cell area of 6203 with cell count of 219, which is about 17% cell area increase together with about 33% cell count increase.

```
============================================================
  Generated by:           Genus(TM) Synthesis Solution 21.14-s082_1
  Generated on:           Aug 24 2026  04:36:16 pm
  Module:                 Row_encoder_5P_plus
  Technology libraries:   fsa0a_c_generic_core_ss1p62v125c 2021Q2v1.0
                          physical_cells 
                          fsa0a_c_generic_core_ss1p62v125c 2021Q2v1.0
                          physical_cells 
  Operating conditions:   _nominal_ 
  Interconnect mode:      global
  Area mode:              physical library
============================================================

      Instance      Module  Cell Count  Cell Area  Net Area   Total Area 
-------------------------------------------------------------------------
Row_encoder_5P_plus                219   6202.728  3191.040     9393.768 

```

But this estimation is based on the physical cells, which may be more accurate.


After updating the lef info during the flow, we have the area report as below:

```
============================================================
  Generated by:           Genus(TM) Synthesis Solution 21.14-s082_1
  Generated on:           Aug 24 2026  04:45:31 pm
  Module:                 Row_encode_5P
  Operating conditions:   _nominal_ 
  Interconnect mode:      global
  Area mode:              physical library
============================================================

   Instance   Module  Cell Count  Cell Area  Net Area   Total Area 
-------------------------------------------------------------------
Row_encode_5P                287   7743.254  4382.456    12125.710 
```

Which showed a total area increase from 9393.768 to 12125.710, which roughly matches my previous estimation of about 30% area increase.

Will have to run a post-syn simulation now to verify the functionality.


