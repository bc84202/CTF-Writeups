## **crypto/xnor-xnor-xnor**

### **Challenge Information**

```
crypto/xnor-xnor-xnor
348 solves / 107 points
https://osu.ppy.sh/beatmapsets/1236927#osu/2573164

expected difficulty: 1/5

Author: wwm
```
#### **Challenge Files**

script.py:
```
import os
flag = open("flag.txt", "rb").read()

def xnor_gate(a, b):
    if a == 0 and b == 0:
        return 1
    elif a == 0 and b == 1:
        return 0
    elif a == 1 and b == 0:
        return 0
    else:
        return 1

def str_to_bits(s):
    bits = []
    for x in s:
        bits += [(x >> i) & 1 for i in range(8)][::-1]
    return bits

def bits_to_str(bits):
    return bytes([sum(x * 2 ** j for j, x in enumerate(bits[i:i+8][::-1])) for i in range(0, len(bits), 8)])

def xnor(pt_bits, key_bits):
    return [xnor_gate(pt_bit, key_bit) for pt_bit, key_bit in zip(pt_bits, key_bits)]

key = os.urandom(4) * (1 + len(flag) // 4)
key_bits = str_to_bits(key)
flag_bits = str_to_bits(flag)
enc_flag = xnor(xnor(xnor(flag_bits, key_bits), key_bits), key_bits)

print(bits_to_str(enc_flag).hex())
# 7e5fa0f2731fb9b9671fb1d62254b6e5645fe4ff2273b8f04e4ee6e5215ae6ed6c
```

### **Solution**

First, I noticed that there is a hex string on the bottom that is commented out, which is likely what is printed when the script was ran. We can save that as our output.

Next, I noticed that the key is generated in a really odd way. The key is composed of 4 random bytes that are repeated for a certain number of times dictated by `(1 + len(flag) // 4)`. This is actually very interesting because I know what the first 4 bytes of the flag should be. It should be `b"osu{"`!

After that, I noticed what operations was done on the flag. It was converted to bits, then "xnor"ed 3 times with the key. Note that from the definition, we can see that xnor is basically xor but you do an extra bit flip at the end. This means if the 2 input bits are the same, the output is 1, and if the 2 input bits are different, the output is 0 (`XNOR(a,b) = ~(a ^ b)`). Doing xnor 3 times is the same as doing it once.

From here, we are ready to solve. First, we can convert the hex output and our flag format to bits. Then, we can define our key, which would be 1 if the bit in the output and the flag format are the same, and 0 if it is not. We then repeat the pattern 10 times to get a key that is long enough. Note that since the output is 264 bits long and the unlengthened key is 32 bits long, repeating it 10 times is good as zip() will truncate to the shorter iterable and 320 > 264. Then, we can solve for the flag because we know that if the key bit is 1, then no changes are made, and if the key bit is 0, then the bit is flipped. We just need to convert the bits to a string afterward to get the flag.

Thus, our script is:
```
output_hex = "7e5fa0f2731fb9b9671fb1d62254b6e5645fe4ff2273b8f04e4ee6e5215ae6ed6c"

def bits_to_str(bits):
    return bytes([sum(x * 2 ** j for j, x in enumerate(bits[i:i+8][::-1])) for i in range(0, len(bits), 8)])

output_bytes = bytes.fromhex(output_hex)
_ = []
for byte in output_bytes:
    _.append(format(byte, '08b'))
output_bits = "".join(_)

flag_format = b"osu{"
_ = []
for byte in flag_format:
    _.append(format(byte, '08b'))
ff_bits = "".join(_)

key = ""
for ff, out in zip(ff_bits, output_bits):
    if (ff == out):
        key = key + "1"
    else:
        key = key + "0"
key = key * 10

flag_bits = ""
for out, key in zip(output_bits, key):
    if (key == "1"):
        flag_bits = flag_bits + str(out)
    else:
        if (out == "1"):
            flag_bits = flag_bits + "0"
        else:
            flag_bits = flag_bits + "1"

flag = []
for _ in flag_bits:
    flag.append(int(_))

print(bits_to_str(flag))
```

The flag is: `osu{b3l0v3d_3xclus1v3_my_b3l0v3d}`