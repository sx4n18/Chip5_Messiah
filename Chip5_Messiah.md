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

