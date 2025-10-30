## **crypto/linear-feedback**

### **Challenge Information**

```
crypto/linear-feedback
142 solves / 119 points
this owc map is so fire btw :steamhappy: https://osu.ppy.sh/beatmapsets/2451798#osu/5355997

expected difficulty: 2/5

Author: wwm
```

#### **Challenge Files**

script.py:
```
from secrets import randbits
from math import floor
from hashlib import sha256

class LFSR:
    def __init__(self, key, taps, format):
        self.key = key
        self.taps = taps
        self.state = list(map(int, list(format.format(key))))
    
    def _clock(self):
        ob = self.state[0]
        self.state = self.state[1:] + [sum([self.state[t] for t in self.taps]) % 2]
        return ob

def xnor_gate(a, b):
    if a == 0 and b == 0:
        return 1
    elif a == 0 and b == 1:
        return 0
    elif a == 1 and b == 0:
        return 0
    else:
        return 1

key1 = randbits(21)
key2 = randbits(29)
L1 = LFSR(key1, [2, 4, 5, 1, 7, 9, 8], "{:021b}")
L2 = LFSR(key2, [5, 3, 5, 5, 9, 9, 7], "{:029b}")

bits = [xnor_gate(L1._clock(), L2._clock()) for _ in range(floor(72.7))]
print(bits)

FLAG = open("flag.txt", "rb").read()
keystream = sha256((str(key1) + str(key2)).encode()).digest() * 2
print(bytes([b1 ^ b2 for b1, b2 in zip(FLAG, keystream)]).hex())
```

output.txt:
```
[0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 1, 0, 1, 0, 1, 0, 0, 1, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 0, 1, 0, 0]
9f7f799ec2fb64e743d8ed06ca6be98e24724c9ca48e21013c8baefe83b5a304af3f7ad6c4cc64fa4380e854e8
```

### **Solution**

The first thing I did was that I noticed was that the flag is xored with a keystream, which is a sha256 encrypted combination of key1 and key2. That tells me there is probably no way to find keystream directly and I have to find key1 and key2, then xor that with the unhexed output to get the flag.

Then I noticed that I am also given a list of 0's and 1's that are the results of xnoring bits of L1 and L2 when they are called from the _clock() function. Looking more closely, I saw that I basically have the results of xnoring results of a few operations done on key1 and key2. More specifically, taking key1 as an example, what the function does is that it adds a new digit (0 or 1) to the end of key1 based on xoring numbers in indices 2, 4, 5, 1, 7, 9, and 8, and then removes the most significant bit, returning that which then gets xnored with whatever gets returned from the same function done on key2(but using different indices). This is done 72 times to give me the list in output.txt.

However, from here, I realized that I could just brute force this as there are only $2^{21}$ possible values of key1. So I wrote a script that loops from $0$ to $2^{21} - 1$ to account for all possible values of key1, which then performs the function on key1 and then xnors with the list in output.txt to get key2. After that, I perform the same hashing function to get the keystring, and xor that with the unhexed ciphertext. I then run a check to see if I end up with bytes that start with "osu" as that is the flag format for this ctf.

Note that I reconstruct key1 slightly differently than how the challenge script does it as I decided to not remove the most significant bit each time, but instead increase all the indices used to find the next digit by 1. I did this in order to keep the first 21 bits of key1 so I can use it more easily later. I also didn't reconstruct the whole 72 bit that the output gave me because I realized I only need to go up to 29 bits to allow me to recover key2, I didn't care about the rest. I also slightly shortened their definition of xnor_gate.

Thus, this is my script:
```
from hashlib import sha256

bits = [0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 0, 1, 1, 0, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 1, 0, 1, 0, 1, 0, 0, 1, 1, 1, 0, 1, 1, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 0, 1, 0, 0]
output_hex = "9f7f799ec2fb64e743d8ed06ca6be98e24724c9ca48e21013c8baefe83b5a304af3f7ad6c4cc64fa4380e854e8"

list1 = [2, 4, 5, 1, 7, 9, 8]

def xnor_gate(a, b):
    if a == b:
        return 1
    else:
        return 0

output_bytes = bytes.fromhex(output_hex)

for i in range(2**21):
    key1 = format(i, '021b')
    key1l = list(map(int, list(key1)))
    key2l = []
    fake_list = list1.copy()
    for _ in range(8):
        key1l.append(sum([key1l[t] for t in fake_list]) % 2)
        fake_list = [x + 1 for x in fake_list]
    for j in range(29):
        key2l.append(xnor_gate(key1l[j], bits[j]))
    key1 = ''
    for k in range(21):
        key1 = key1 + str(key1l[k])
    key1 = str(int(key1, 2))
    key2 = ''
    for k in range(29):
        key2 = key2 + str(key2l[k])
    key2 = str(int(key2, 2))
    keystream = sha256((key1 + key2).encode()).digest() * 2
    flag = bytes([b1 ^ b2 for b1, b2 in zip(output_bytes, keystream)])
    if flag.startswith(b"osu"):
        print(flag)
        break
```

Running the script gives me the flag.

Interestingly, after running with timers, the amount of time it took to brute force on my computer is only 12.70s. For worst case scenario (where key1 = $2^{21} - 1$, compared to $776071$ which is what it actually is), it would have taken 33.13s. Overall, the numbers are small enough where it is fast and simple to brute force.

The flag is: `osu{th1s_hr1_i5_th3_m0st_fun_m4p_3v3r_1n_0wc}`