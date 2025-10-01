---
layout: writeup
show-avatar: false
latex: true
logo: /assets/img/writeups/htb/crypto/twoforone/twoforone_logo.png
title: TwoForOne
platform: HTB
category: Crypto
difficulty: Easy
release: 04 Dec 2020
creator: <a href="https://app.hackthebox.eu/users/337512">Fukurou</a> 
published: 2021 05 06
---

**Cliffs:** Extract the modulus and exponent from the RSA public keys. Use Bézoult's Identity to  retrieve the message from the ciphertexts.



<h4 align="center">The Challenge</h4>

We are simply told "Alice sent two times the same message to Bob." and then provided with 4 files, *key1.pem*, *key2.pem*, *message1*, *message2*. Taking a look at the files we see

```
└─$ cat key1.pem
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAxqy430huZnHUpVZIA+HD
IUqOJ03grABD7CjIWJ83fH6NMIvD4wKFA4Q0S6eYiIViCkGOatlVV4KE/ATyifEm
s4oBgWJRzvmhT9TCSdlraQh/qRsuGtvcgMuW/wzLYSnY9nN9qFDEUfLtP2y2HDaJ
Hckk0Kso8mrfDtNXzoSNAv/gCRJxTM9jcsH0EIDoZ0egMD61zfbOkS8RRP1PVXQ8
eWh1oU/f+Pi2YhUMVr5YsJI5dx3ETZaQecStj9mTvGMLeFXS4C6L4Wgk3NWrOBMj
HBcxEQqL0CjXod+riS51KUVXuvxxrq9eSNsCZ6bbY9NQ+ZUGjuHK1tMt8RpJvSS6
lwIDAQAB
-----END PUBLIC KEY-----
```

```
└─$ cat message1
RBVdQw7Pllwb42GDYyRa6ByVOfzRrZHmxBkUPD393zxOcrNRZgfub1mqcrAgX4PAsvAOWptJSHbrHctFm6rJLzhBi/rAsKGboWqPAWYIu49Rt7Sc/5+LE2dvy5zriAKclchv9d+uUJ4/kU/vcpg2qlfTnyor6naBsZQvRze0VCMkPvqWPuE6iL6YEAjZmLWmb+bqO+unTLF4YtM1MkKTtiOEy+Bbd4LxlXIO1KSFVOoGjyLW2pVIgKzotB1/9BwJMKJV14/+MUEiP40ehH0U2zr8BeueeXp6NIZwS/9svmvmVi06Np74EbL+aeB4meaXH22fJU0eyL2FppeyvbVaYQ==
```

with *key2.pem* and *message2* being similar. So presumably the messages are the resulting ciphertexts of Alice's encrypting and sending the same message twice to Bob, and the corresponding public keys are what was involved in the encryption.
<br>
<br>

**What the heck is a .pem file?!**

Before we can figure out what to do, we need to understand what we are even working with. <a href="https://www.ssl.com/guide/pem-der-crt-and-cer-x-509-encodings-and-conversions/">PEM</a> is the most common format for <a href="https://www.ssl.com/faqs/what-is-an-x-509-certificate/">X.509</a> certificates and keys. A .pem file is a text file containing one or more such items in Base64 ASCII, each with plain-text headers and footers. (The *-----BEGIN PUBLIC KEY-----* and *-----END PUBLIC KEY-----* seen above). X.509 is a standard for defining the format of public key certificates. This standard uses <a href="https://en.wikipedia.org/wiki/ASN.1">ASN.1</a> to express the structure of these files.  ASN.1 is a language for defining data structures that can be serialized and deserialized across platforms. So long story short, it's a standardized way to express the information necessary to perform various cryptographic functions.

Since it can contain various different elements, how do we know what we are dealing with and what is encoded? There are many ways we can decode a .pem file. One of which is the command line tool *openssl*. Here we use the <a href="https://www.openssl.org/docs/man1.1.1/man1/openssl-asn1parse.html">asn1parse</a> command with the *-in* flag telling it which file to use.

```
─$ openssl asn1parse -in key1.pem 
    0:d=0  hl=4 l= 290 cons: SEQUENCE          
    4:d=1  hl=2 l=  13 cons: SEQUENCE          
    6:d=2  hl=2 l=   9 prim: OBJECT            :rsaEncryption
   17:d=2  hl=2 l=   0 prim: NULL              
   19:d=1  hl=4 l= 271 prim: BIT STRING
```

This displays information about the structure of the file. We can see that it is telling us this public key is for use with RSA.

We can also use an on-line ASN.1 decoder like the one [here](https://lapo.it/asn1js/) and input our file

![online asn1 decoder](/assets/img/writeups/htb/crypto/twoforone/twoforone_asn1decode.png)
<br>
<br>

**What the heck is a RSA?**

OK, so you've probably heard of <a href="https://en.wikipedia.org/wiki/RSA_(cryptosystem)">RSA</a>, but for this challenge we won't need to understand it deeply, we'll just briefly cover the basics we need. RSA is an asymmetric cryptography scheme. Meaning the key used for encryption is different than the one for decryption. A public key can be distributed and shared and allows anyone to use it to encrypt information, after which it can only be decrypted by a specific private key. The public key needs just two pieces of information, two different numbers. An exponent $e$ and a modulus $n$. These are created in such a way that there is a specific exponent $d$ which is kept secret and used to decrypt messages (this is the private key). The method and math behind the creation of these numbers is not important here. We just need to know that encryption is done by converting a message $m$ to a number and then raising that number to the power of $e$ modulo $n$ which produces the ciphertext $c$. If you aren't familiar with <a href="https://en.wikipedia.org/wiki/Modular_arithmetic">modular arithmetic</a>, you can think of a number "modulo $n$" as taking the number and dividing it by *n* and only keeping the remainder. The equation for encryption is below and would be read as "*m* to the *e* is congruent to *c* mod *n*".
\\[
m^e \equiv c \pmod{n}
\\]
<br>
<br>

**What the heck is $(n,e)$?**

So how do we retrieve the values for $n$ and $e$ that make up the public key from the .pem file? We can use *openssl* again, this time with the <a href="https://www.openssl.org/docs/man1.1.1/man1/openssl-rsa.html">rsa</a> command. We use *-pubin* to tell it we are inputting a public key *-in* to specify the file location *-text* to output the components of the key *-modulus* to print out the value of the modulus and *-noout* to keep it from displaying the encoded version of the key.

```
└─$ openssl rsa -pubin -in key1.pem -text -modulus -noout
RSA Public-Key: (2048 bit)
Modulus:
    00:c6:ac:b8:df:48:6e:66:71:d4:a5:56:48:03:e1:
    c3:21:4a:8e:27:4d:e0:ac:00:43:ec:28:c8:58:9f:
    37:7c:7e:8d:30:8b:c3:e3:02:85:03:84:34:4b:a7:
    98:88:85:62:0a:41:8e:6a:d9:55:57:82:84:fc:04:
    f2:89:f1:26:b3:8a:01:81:62:51:ce:f9:a1:4f:d4:
    c2:49:d9:6b:69:08:7f:a9:1b:2e:1a:db:dc:80:cb:
    96:ff:0c:cb:61:29:d8:f6:73:7d:a8:50:c4:51:f2:
    ed:3f:6c:b6:1c:36:89:1d:c9:24:d0:ab:28:f2:6a:
    df:0e:d3:57:ce:84:8d:02:ff:e0:09:12:71:4c:cf:
    63:72:c1:f4:10:80:e8:67:47:a0:30:3e:b5:cd:f6:
    ce:91:2f:11:44:fd:4f:55:74:3c:79:68:75:a1:4f:
    df:f8:f8:b6:62:15:0c:56:be:58:b0:92:39:77:1d:
    c4:4d:96:90:79:c4:ad:8f:d9:93:bc:63:0b:78:55:
    d2:e0:2e:8b:e1:68:24:dc:d5:ab:38:13:23:1c:17:
    31:11:0a:8b:d0:28:d7:a1:df:ab:89:2e:75:29:45:
    57:ba:fc:71:ae:af:5e:48:db:02:67:a6:db:63:d3:
    50:f9:95:06:8e:e1:ca:d6:d3:2d:f1:1a:49:bd:24:
    ba:97
Exponent: 65537 (0x10001)
Modulus=C6ACB8DF486E6671D4A5564803E1C3214A8E274DE0AC0043EC28C8589F377C7E8D308BC3E302850384344BA7988885620A418E6AD955578284FC04F289F126B38A01816251CEF9A14FD4C249D96B69087FA91B2E1ADBDC80CB96FF0CCB6129D8F6737DA850C451F2ED3F6CB61C36891DC924D0AB28F26ADF0ED357CE848D02FFE00912714CCF6372C1F41080E86747A0303EB5CDF6CE912F1144FD4F55743C796875A14FDFF8F8B662150C56BE58B09239771DC44D969079C4AD8FD993BC630B7855D2E02E8BE16824DCD5AB3813231C1731110A8BD028D7A1DFAB892E75294557BAFC71AEAF5E48DB0267A6DB63D350F995068EE1CAD6D32DF11A49BD24BA97
```

If we do the same for *key2.pem* we will see that the only difference is the exponent.

```
Exponent: 343223 (0x53cb7)
```

The modulus is expressed in <a href="https://learn.sparkfun.com/tutorials/hexadecimal/all">base16</a> (hexadecimal), if it seems weird to you to think of this as a number you can convert it easily with python to alleviate any doubts.

```
└─$ python3 -c "print(int('C6ACB8 //...snip...// BA97', 16))"

25080356853331150673712092961488349508728123694382279186941974911344272809718201683391687288116618021523872262260746884803456249468108932413753368793568123710905490623939699616018064364038794824072468125668702688048418916712950393799664781694224559810656290997284081084848717062228887604668548576609649709572412523306016494962925450783098637867249337121156908328927249731928363360657779226929980928871118145919627109584218577535657544952661333527174942990937484743860494188129607347202336812042045820577108243818426559386634634103676467773122325120858908782192357380855678371737765634819794619802582481594876770433687
```

For obvious reasons I will simply refer to this number as $n$.

So the message was encrypted both times with the same modulus and it was only the exponent that was changed. The resulting numbers $c_1$ and $c_2$ are the [base64](https://base64.guru/learn/what-is-base64) encoded contents of our *message1* and *message2* files.  So we have <a id="equations" style="visibility: hidden">these</a> 
\\[
m^{e_1}\equiv c_1 \pmod{n} \\\ m^{e_2}\equiv c_2 \pmod{n}
\\]
We know $e_1,e_2,c_1,c_2,\text{and }n$, how do we use that to get $m$?
<br>
<br>

**Who the heck is Bézout?!**

There is a theorem that states if you have two non zero whole numbers, $a$ and $b$, then there exists an $x$ and a $y$ that are solutions to the equation
\\[
ax + by = \gcd(a,b)
\\]
where $\gcd(a,b)$ is the <a href="https://en.wikipedia.org/wiki/Greatest_common_divisor">greatest common divisor</a> (the largest number that evenly divides them both) of the numbers $a$ and $b$ . This is known as <a href="https://brilliant.org/wiki/bezouts-identity/">Bézout's identity</a>. Looking at our two exponents $e_1$, $e_2$ we can see that
\\[
\gcd(e_1, e_2) = \gcd(65537, 343223) = 1
\\]
(check <a href="https://www.dcode.fr/gcd">here</a> if you don't believe me). So combining these 2 facts we have
\\[
\begin{align}
65537x + 343223y &= \gcd(65537,343223) \qquad \text{or,}
\\\ e_1x + e_2y &= 1
\end{align}
\\]
How does that help us? Well if we knew $x$ and $y$ and could somehow get ourselves into an equation where we can evaluate $m^{e_1x + e_2y}$ we would be left with the message!
\\[
m^{e_1x + e_2y} = m^1 = m
\\]
Enter <a href="https://www.mathsisfun.com/algebra/exponent-laws.html">The Laws of Exponents</a>. We can use the following facts
\\[
\begin{align}
(x^m)^n &= x^{mn} \qquad &\text{(1)} 
\\\ x^mx^n &= x^{m + n} \qquad &\text{(2)}
\end{align}
\\]
to achieve the following
\\[
\begin{align}
&(m^{e_1})^x (m^{e_2})^y 
\\\ &= m^{e_1x} m^{e_2y} \quad &\text{via (1)}
\\\ &=m^{e_1x + e_2y} \quad &\text{via (2)}
\\\ &=m &\text{via Bézout's Identity}
\end{align}
\\]
We know $m^{e_1}$ and $m^{e_2}$. As we learned <a href="#equations">earlier</a> that's how RSA produces the encrypted messages! So we can just substitute
\\[
\begin{align}
(m^{e_1})^x (m^{e_2})^y = (c_1)^x(c_2)^y 
\end{align}
\\]
and we just showed the left side of that equations reduces to $m$. However we started with a congruence relation, not strict equalities, what we really have is
\\[
\begin{align}
(m^{e_1})^x (m^{e_2})^y \equiv (c_1)^x(c_2)^y \pmod{n}
\\\ m^{e_1x + e_2y} \equiv (c_1)^x(c_2)^y \pmod{n}
\\\ m \equiv (c_1)^x(c_2)^y \pmod{n}
\end{align}
\\]
so all we have to do now is find $x$ and $y$ by solving
\\[
65537x + 343223y = 1
\\]
Enter the <a href="https://brilliant.org/wiki/extended-euclidean-algorithm/">Extended Euclidean Algorithm</a>. The Euclidean algorithm is a famous algorithm for finding the greatest common divisor of two integers. It does this by repeatedly dividing the divisor by the reaminder until nothing is left. The extended algorithm works it's way backwards, by starting with the gcd and eventually expressing it as a linear combination of the original two numbers. We can use this to solve the above formula. Using this <a href="https://www.dcode.fr/extended-gcd">tool</a> we get values of 133,132 and -25,421 for $x$ and $y$ respectively. 
<br>
<br>

**What the heck is a multiplicative inverse?**

Now we just plug those numbers in and let 'er rip right? Well, not quite. We have one small issue. We need to calculate
\\[
c_1^{133132}c_2^{-25421} \mod{n}
\\]
What is a number raised to a negative power? From our Laws of Exponents we know that $c_2^{-25421}$ is the same thing as $(c_2^{-1})^{25421}$, but what is $c_2^{-1}$? You may know that a number to a negative exponent means "take its inverse" or "reciprocal" and are used to treating it like the following
\\[
x^{-1} = \frac{1}{x}
\\]
However here we are working with the [integers](https://www.storyofmathematics.com/integers) modulo $n$, which are whole numbers, not with the typical [real numbers](https://www.mathsisfun.com/numbers/real-numbers.html) (any number you can point to on the number line). Fractions don't exist in this world.

Enter the [multiplicative inverse](https://www.splashlearn.com/math-vocabulary/fractions/multiplicative-inverse). The multiplicative inverse of a number is the number you need to multiply it by to get the number one. *[pedantic math note: strictly speaking this is not true. It is the number you need to multiply by to get the [Identity Element](https://en.wikipedia.org/wiki/Identity_element), but for the sake of brevity we will ignore all that]*. This is why for example $7^{-1} = \frac{1}{7}$ because
\\[
7 \times \frac{1}{7} = 1
\\]
So what we need then is the number that fits in
\\[
c_2 \times ? \equiv 1 \pmod{n}
\\]
We can actually use our new friend the Extended Euclidean Algorithm to find this. We've seen that we can use it to solve Bézout's identity problems, and for reasons we can ignore, we can assume that the $\gcd(c_2, n) = 1$, and so look what we have if we find the solutions for the Bézout's Identity
\\[
\begin{align}
&c_2x + ny = \gcd(c_2, n)
\\\ & c_2x + ny = 1
\\\ &c_2x = -ny + 1
\\\ \Rightarrow \quad &c_2x \equiv 1 \pmod{n}
\end{align}
\\]
this last step might look confusing, but because $c_2x$ equals one more than some multiple of $n$ (specifically we have negative $y$ multiples of it), we can transform the third line into the fourth, because having one more than some multiple of $n$ means we will have a remainder of one when we divide it by $n$, which is how we defined modular arithmetic earlier.

Since $n$ is a rather quite large number, we won't plug it into the same online tool, but now that we know how to find it, we have the last piece to our puzzle and we can now write up some python code to retrieve the message by calculating
\\[
c_1^{x}(c_2^{-1})^{y} \mod{n}
\\]
<br>
<br>
**What the heck is python?**

[Python](https://www.python.org) is a multi-paradigm programming language that uses.....naw I'm just kidding. Here is the code I used for this challenge. You will need the [PyCryptodome](https://pycryptodome.readthedocs.io/en/latest/src/introduction.html) package to run it.

```python
from base64 import b64decode
# pycryptodome
from Crypto.PublicKey import RSA
from Crypto.Util import number

def bezout(a, b):
    """ Calculates ax + by = gcd(a,b) (Bezout's Identity) using the
        Exetended Euclidean Algorithm. Ripped straight off the pseudocode
        at https://en.wikipedia.org/wiki/Extended_Euclidean_algorithm
        
        returns (x, y, gcd(a,b))
    """
    old_r = a
    r = b
    old_s = 1
    s = 0
    old_t = 0
    t = 1

    while (r != 0):
            q = old_r // r
            tmp_r = r
            r = old_r - q * tmp_r
            old_r = tmp_r
            tmp_s = s
            s = old_s - q * tmp_s
            old_s = tmp_s
            tmp_t = t
            t = old_t - q * tmp_t
            old_t = tmp_t

    return (old_s, old_t, old_r)

# import and decode all the provided files
with open('key1.pem', 'r') as key1:
    pub1 = RSA.import_key(key1.read())

with open('key2.pem', 'r') as key2:
    pub2 = RSA.import_key(key2.read())

with open('message1', 'r') as message1:
    cipher1 = b64decode(message1.read())

with open('message2', 'r') as message2:
    cipher2 = b64decode(message2.read())


# fill all the variables for our calculation

# convert the base64 decoded ciphertext into numbers
c_1 = int.from_bytes(cipher1, byteorder='big', signed=False)
c_2 = int.from_bytes(cipher2, byteorder='big', signed=False)

# we could use the bezout function for this but pycrytodome provides us
# with a simple way to get the inverse of a number modulo n
c_2_inv = number.inverse(c_2, pub1.n)

# x and y from our equation e_1x + e_2y = gcd(e_1, e_2)
(x, y, z) = bezout(pub1.e, pub2.e)

# calculating the message
m = pow(c_1, x, pub1.n) * pow(c_2_inv, -y, pub1.n) % pub1.n

# We could actually use pow(c_2, y, pub1.n) as python will automatically calculate the 
# inverse (it will error if it doesnt exist) when we use a negative exponent
# and the modulus parameter, but I thought it would be helpful to explain inversion.
# m = pow(c_1, x, pub1.n) * pow(c_2, y, pub1.n) % pub1.n


# m is a decimal number, to convert it to the text message first we 
# convert it to hexadecimal (which will have the form '0x1234' so we 
# use [2:] to ignore the '0x', and then we convert that to bytes 
# and finally to UTF-8
flag = bytes.fromhex(hex(m)[2:]).decode()

print(flag)
```





