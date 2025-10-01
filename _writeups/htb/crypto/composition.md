---
layout: writeup
show-avatar: false
latex: true
logo: /assets/img/writeups/htb/crypto/composition/composition_logo.png
title: Composition
platform: HTB
category: Crypto
difficulty: Medium
release: 02 Apr 2021
creator: <a href="https://app.hackthebox.eu/users/185587">DaysOfLife</a> 
published: 2021 06 02
---


**Cliffs:** Factor the modulus to get $p$ and $q$, which can then be used to get $e$. Solve $ge = A$ for each curve modulo $p$ and $q$ separately by using the order of the curve and then recombine to get original point on $N$. Use $x$ coordinate of the point to get the key and decrypt the flag.



<h4 align="center">The Challenge</h4>

*"The more the merrier, right? We decided to mash two of the best cryptosystems together for the best product. Our new encryption scheme is up and running and this time it is unbreakable! To prove that, we have also released its source code and a test center where you can test it  out!"*

Bold statement! The source code consists of two files, *ecc.py* and *server.py*. Looking briefly over both it appears the format of RSA is being followed to generate a large number $N$ as the product of two primes $p$ and $q$, along with an encryption exponent $e$. However instead of these parameters being used as they normally would, an elliptic curve is being constructed modulo $N$ and $e$ is being used to multiply a random point $g$ on said curve to produce a point $A$. The $x$ value of the point $g$ is then being used to encrypt the flag we are after. The only values we receive are the point $A$, the modulus $N$ and the IV used during encryption, and optionally a random point on the curve.

The idea here seemingly being that double the protection is provided as we not only need to factor $N$, a task that provides the security in RSA, we also need to solve the elliptic curve discrete logarithm problem, which is how the security in cryptosystems like Diffie-Hellman is achieved. With all that protection surely this must be unbreakable!!?? Let's take a closer look and see what they messed up....
<br>
<br>


**Our path forward**

Working backwards from our ultimate goal of decrypting the flag, which is encrypted as follows in *server.py*

```python
key = md5(str(g.x).encode()).digest()
iv = os.urandom(16)
cipher = AES.new(key,AES.MODE_CBC,iv)
data = cipher.encrypt(pad(flag,16))
```

We see this could be achieved with possesion of the IV and the key. We are given the IV, the key however is reliant on the $x$ co-ordinate of the random point $g$. Which is a point on the elliptic curve $E$ modulo $N$

```python
ec = EllipticCurve(a,b,n)
g = getrandpoint(ec,p,q)
```

This is information we don't have. We are however given the point $A = ge$

```python
A = ec.multiply(g,e)
```

if we could somehow get $e$ and the necessary information to construct $E$ we could then seemingly figure out $g$ and thus the flag. We therefore have two goals, figuring out how to get $e$ and how to construct $E$. Let's start with the first of those.
<br>
<br>


**Too close for comfort**

Let's look at how the RSA half of the system is produced, which takes place in the following function located in *server.py*

```python
def keygen(bits):
    # Returns RSA key in form ((e,n),(p,q))
    p = getPrime(bits // 2)
    while p % 4 == 1:
        p = next_prime(p)
    e = next_prime(p >> (bits // 4))
    q = next_prime(p)
    for i in range(50):
        q = next_prime(q)
    while q % 4 == 1:
        q = next_prime(q)
    n = p * q
    if n.bit_length() != bits:
        return keygen(bits)
    return (e,n),(p,q)
```

we'll see a few important things. The first of which is that $e$ is dependent on $p$ 

```python 
e = next_prime(p >> (bits // 4))
```

$e$ is simply the next prime after dividing $p$ by some number. That's both good and bad news for us. The good is we have a foothold to our first task, all we need to do is find $p$. The bad is that we have to find $p$, a feat that is supposed to be impossible if the system is indeed secure.

There is another important thing going on here though.  We see that once $p$ is chosen, $q$ is then chosen as follows

```python
 q = next_prime(p)
    for i in range(50):
        q = next_prime(q)
    while q % 4 == 1:
        q = next_prime(q)
```

$q$ is the next prime after $p$, 50 times over. So we get $p$, then we go 50 primes and then $q$ is the very next prime that doesn't have a reaminder of one when divided by four (*aside: If you are wondering why primes of the form $p \equiv 3 \pmod{4}$  are being made sure to be used here, this is simply so that square roots are easier to calculate when getting the $y$ value for points on the elliptic curve modulo that prime)* . This means that $p$ and $q$ are very close together, a simple fact that will allow us to factor $N$.
<br>
<br>


**Fermat Factorization**

Let's look at a number that we can write as the difference of two squares
\\[
N = x^2 - y^2
\\]
We could rewrite the right half  of that equation as 
\\[
x^2 - y^2 = (x+y)(x-y)
\\]
giving us
\\[
N = (x+y)(x-y)
\\]
If we continually try adding squares to $N$ until we find one that produces a sum with N that is equal to a square, we would then be able to write $N$ as the difference of two squares and produce a pair of factors for $N$.

Let's say for example we wanted factor $276507$. We would start first by adding $1^2$ to it and checking if the sum is a square
\\[
276507 + 1^2 = 276508
\\]
$276508$ is not a perfect square, so we move on and try $2^2$
\\[
276507 + 2^2 = 276511
\\]
$276511$ is not a square number either. We continue on in this fashion checking if each sum is a square number, until we get to $13^2$
\\[
276507 + 3^2 = 276516
\\\ 276507 + 4^2 = 276523
\\\ \vdots
\\\ 276507 + 13^2 = 276676
\\]
$276676$ is a square number as $276676 = 526^2$. This means we can now write $276507$ as a difference of squares
\\[
276507 = 526^2 - 13^2
\\]
or
\\[
276507 = (526 + 13)(526 - 13)
\\]
giving us $539$ and $513$ as two factors of $276507$

This might seem like a great way to factor large numbers, but it is in fact not very efficient. If $N = pq$ it takes $|p - q| / 2$ steps to find the factorization. So if $p$ and $q$ are far apart this is infeasible, but here they are only ~50 primes apart. We can estimate how far apart that is via the [prime number theorem](https://primes.utm.edu/howmany.html#pnt) which tells us the average gap between primes less than $n$ is $\log(n)$. Given $p$ is a random 256 bit prime this works out to an average gap of around 178 numbers between $p$ and the next prime and an estimated average distance of  $50 * 178 = 8900$ numbers until we hit $q$, This then is obviously easily close enough for us to calculate the factors of $N$. So $p$ and $q$ are ours now, and $e$ as well.
<br>
<br>


**Woah we're halfway there, woah-oh...**

We've accomplished our first task, finding $e$. Now we must find the required parameters to construct the elliptic curve $E$ so that we may solve $ge = A$ for the point $g$

I won't be going into depth on the nature of elliptic curves, how points are added or about the group they form and why that is useful cryptographically. If these are things you are unfamiliar with or you would like to learn more about, I highly encourage you to read this ***excellent*** article [Elliptic Curve Cryptography: a gentle introduction](https://andrea.corbellini.name/2015/05/17/elliptic-curve-cryptography-a-gentle-introduction/). For our purposes we will be focusing on a few things. 

First, elliptic curves usually take the form
\\[
E: y^2 = x^3 + ax + b
\\]
If we look at the *lift_x* function in the *EllipticCurve* class in *ecc.py* responsible for turning an *x* value into a point on the curve

```python
class EllipticCurve:
    INF = Point(0,0)
    def __init__(self, a, b, p):
        self.a = a
        self.b = b
        self.p = p
<...>
    def lift_x(self,x,p,q):
        expr = x**3 + self.a*x + self.b
        y = composite_square_root(expr,p,q)
        return Point(x,y)
```

we see from the line

```python
expr = x**3 + self.a*x + self.b
```

That our curve follows the normal form. The coefficients $a$ and $b$ of the curve are generated randomly in *server.py* 

```python
a,b = random.getrandbits(128),random.getrandbits(128)
```

So how are we going to get those? Well thankfully for us we have the option of getting a random 2nd point on the curve from the server. And if you paid attention in algebra you'll know that all we need to solve for two unknowns is two equations. We can plug in each of the $x$ and $y$ values of the two points into the equation for the curve, solve one for $b$ and then substitute that into the other and solve for $a$.
\\[
\begin{align}
&y_1^2 = x_1^3 + ax_1 + b
\\\ \Rightarrow \qquad &b = y_1^2 - x_1^3 -ax_1
\\\ &y_2^2 = x_2^3 + ax_2 + b
\\\ \Rightarrow \qquad &y_2^2 = x_2^3 + ax_2 + (y_1^2 - x_1^3 - ax_1)
\\\ \Rightarrow \qquad &y_2^2 - y_1^2 - x_2^3 + x_1^3 = a(x_2 - x_1)
\\\ \Rightarrow \qquad &a = \frac{y_2^2 - y_1^2 - x_2^3 + x_1^3}{x_2 - x_1}
\end{align}
\\]
 

After the coefficients are generated our curve is then created modulo $N$

```python
ec = EllipticCurve(a,b,n)
```
<br>


**If you build it they will come**

This brings us to the second thing. For an elliptic curve to form a group, it needs to be defined over a [field](https://en.wikipedia.org/wiki/Field_(mathematics)), if you aren't sure what this is, just know it is a certain kind of algebraic structure. Our composite $N$ will not form a field, it forms what is called a [ring](https://en.wikipedia.org/wiki/Ring_(mathematics)), which again is just a type of algebraic structure. What this means is that our elliptic curve over $N$ does not form a group as not all points can be added. It is not "closed under addition", if we add two points on the curve we can not be guaranteed to get a 3rd point on the curve, and closure is a required property of a group.

However all is not lost, we can simply look at the curves
\\[
y^2 = x^3 + ax + b \pmod{p}
\\\ y^2 = x^3 + ax + b \pmod{q}
\\]
and perform our calculations on each as the set of integers modulo $p$ (denoted as $\mathbb{Z}/p\mathbb{Z}$)  and $q$ $ (\mathbb{Z}/q\mathbb{Z}$) can both form finite fields. We can then use the chineese remainder theorem to combine each point into a point on $N$.
<br>
<br>


**Order in the court**

How do we actually go about solving $ge = A$ on each of these curves? This brings us to the third thing we need to know, the order of a curve. An elliptic curve defined over a finite field $K$ has a finite number of points. That number of points is the order of the curve (which I will be denoting as $\\#E(K)$). The order of each point on the curve is defined as the smallest integer by which that point can be multiplied by to get the identity element, which in an elliptic curve is the point of infinity (denoted $\mathcal{O}$). So for example the order of the point $g$ would be the smallest value of $x$ such that $gx = \mathcal O$.  We can then make use of [Lagrange's theorem](https://en.wikipedia.org/wiki/Lagrange%27s_theorem_(group_theory)), which states that the order of a subgroup is a divisor of the order of the parent group. In short, no matter what the order of $g$ is, we know that it will divide the order of the curve $E$. So if we multiply $g$ by the order of the curve we will get the identity.  

For example say the order of $g$ is 7, then
\\[
7g = \mathcal{O}
\\]
By Lagrange's theorem the order of $E$ must then be some multiple of 7. Let's say it's 21. Then
\\[
21g = 3 * 7g = 3 * \mathcal{O} = \mathcal{O} + \mathcal{O} + \mathcal{O} = \mathcal{O}
\\]
since the definining property of an identity element is that anything added to it is equal to itself, and multiplication over an elliptic curve is a series of point additions.

How does this help us? Well let's look at what happens if we find the inverse of $e$ modulo the order of the curve.  Let's define $x$ to be that inverse
\\[
x \equiv e^{-1} \pmod{\\#E(K)}
\\]
this means that $xe \equiv 1 \pmod{\\#E(K)}$ since this is how the [modular inverse](https://www.khanacademy.org/computing/computer-science/cryptography/modarithmetic/a/modular-inverses) is defined. It is one more than some multiple of $\\#E(K)$ or
\\[
xe = i(\\#E(K)) + 1
\\]
for some $i$. So if we multiply our point $A$ by $x$ we get
\\[
\begin{align}
Ax &= gex 
\\\ gex &= g(i(\\#E(K) + 1) \qquad &\text{via definition of inverse}
\\\ g(i(\\#E(K) + 1) &= gi\\#E(K) + g \qquad &\text{via distribution}
\\\ gi\\#E(K) + g &= \mathcal{O} + g \qquad &\text{via definition of order}
\\\ \mathcal{O} + g &= g \qquad &\text{via property of identity}
\end{align}
\\]
In case I lost you along the way, the headline is that if we have the order of the curve, we can find the value for the point $g$, which somewhere above we said was a very good thing.

Thankfully for us there is an efficient algorithm to compute the number of points on an elliptic curve. [Schoof's algorithm](https://en.wikipedia.org/wiki/Schoof%27s_algorithm). I won't go over it here, but we can use it to get the order of our curves. 
<br>
<br>


**Let's recap**
- Use Fermat factorization to factor $N$ into $p$ and $q$. 
- The aquisition of $p$ allows us to derive the value for $e$. 
- Use the 2nd point given to us on the curve to solve a system of equations and derive $a$ and $b$
- Construct two seperate curves modulo $p$ and modulo $q$
- Use Schoof's algorithm to get the order of each curve. 
- Calculate the inverse of $e$ modulo that order
- Multiply the point $A$ by that inverse to get the point $g$, 
- Take the $g$ from each curve and use the chineese remainder theorem to get the point on $N$. 
- Use the $x$ coordinate of that point and the IV to decrypt the flag.
- Bask in our glory
<br>
<br>


**Putting it all together**

Let's write some python to implement all our steps. Like any true programmer we'll be copy/pasting as much of the code as possible from the included source code. We'll also have to use a separate program to calculate the order of the curves as there doesn't seem to be a good python implementation.

First up is our fermat factorization, we'll be importing the *is_square* and *sqrt* functions from [SymPy](https://www.sympy.org/en/index.html)

```python
def fermat_factorize(n, bound = 10**6):
    i = 1
    p, q = None, None
    for i in range(bound):
        num = n + i**2
        if is_square(num):
            root = sqrt(num)
            p = root - i
            q = root + i
            break
    return (p,q)
```

We'll copy the *next_prime* prime function directly from *server.py* making sure to import *isPrime* from the [pycryptodome](https://pypi.org/project/pycryptodome/) library as they did. This way we'll be sure to get the same value for $e$ as they did.

To get our our coefficients *a* and *b* well make use of the *moddiv* function from *ecc.py* to calculate the division that takes place

```python
def get_curve_constants(p1, p2, n):
    a = moddiv((p2.y**2 - p1.y**2 - p2.x**3 + p1.x**3), (p2.x - p1.x), n)
    b = p1.y**2 - p1.x**3 - a*p1.x

    return (a % n, b % n)
```

Construct the curves over $p$ and $q$ using the imported *EllipticCurve* class from $ecc.py$. We don't need to worry about $a$ or $b$ being larger than $p$ or $q$ since they are random 128 bit numbers and both primes are at least 256 bits. Otherwise we would need to make sure to calculate each coefficient modulo the prime.

```python
Ep = EllipticCurve(a, b, p)
Eq = EllipticCurve(a, b, q)
```

To get the order of the curves we will be using an online implementation of [Sage](https://www.sagemath.org/), found [here](https://sagecell.sagemath.org/). We'll have python print out the values of the variables we need, and then we'll input the results from our manual execution of Sage back in. Here is how the input it into sagecell will look, obviously changing the values to match what the particular variable is in each case.

```python
p = 103185539025321143671338999706522072700199299392023728394174386975405659182187
q = 103185539025321143671338999706522072700199299392023728394174386975405659190899
a = 223305774678616518868787010099897981024
b = 100931363354017193559535104815100748475

Fp = FiniteField(p)
Fq = FiniteField(q)
Ep = EllipticCurve(Fp, [a, b])
Eq = EllipticCurve(Fq, [a, b])

print(Ep.order())
print(Eq.order())
```

After calculating the inverse of $e$ modulo each prime

```python
e_inv_oder_Ep = pow(e, -1, order_Ep)
e_inv_order_Eq = pow(e, -1, order_Eq)
```

and putting the point $A$ onto each curve

```
A_Ep = Point(A.x % p, A.y % p)
A_Eq = Point(A.x % q, A.y % q)
```

We'll then multiply them together to get the original $g$ on each curve. However there is an error in the implementation of *EllipticCurve* in *ecc.py* we need to fix. In the *multiply* function on line 50, we need to remove (or comment out) this line

```python
n %= self.p
```

Because as we now know the order of a curve is the number of points on the curve and that number is not always the value of the prime it is over (It rarely is). It is possible we that will need to multiply a point by a value larger than the modulus. With that fixed we can get $g$

```python
g_Ep = Ep.multiply(ge_Ep, e_inv_order_Ep)
g_Eq = Eq.multiply(ge_Eq, e_inv_order_Eq)
```

and then use the *crt* function from *ecc.py* to recombine them to a point in $N$

```python
g_Epq = Point(crt([g_Ep.x, p], [g_Eq.x, q]), crt([g_Ep.y, p], [g_Eq.y, q])
```

and then run that through our decryption function to decode the flag

```python
def decryption(g, iv, ct):
    key = md5(str(g.x).encode()).digest()
    iv = bytes.fromhex(iv)
    cipher = AES.new(key, AES.MODE_CBC, iv)

    return cipher.decrypt(bytes.fromhex(ct))
```

The full code is below

```python
from pwn import *
from ecc import *
from sympy.ntheory.primetest import is_square
from sympy import sqrt
from Crypto.Util.Padding import unpad
from Crypto.Util.number import isPrime
from Crypto.Cipher import AES
from hashlib import md5

# attempt to factorize n using fermat factorization
def fermat_factorize(n, bound = 10**6):
    i = 1
    p, q = None, None
    for i in range(bound):
        num = n + i**2
        if is_square(num):
            root = sqrt(num)
            p = root - i
            q = root + i
            break
    return (p,q)

# get the values for a and b given two points on the curve
def get_curve_constants(p1, p2, n):
    a = moddiv((p2.y**2 - p1.y**2 - p2.x**3 + p1.x**3), (p2.x - p1.x), n)
    b = p1.y**2 - p1.x**3 - a*p1.x

    return (a % n, b % n)

def decryption(g, iv, ct):
    key = md5(str(g.x).encode()).digest()
    iv = bytes.fromhex(iv)
    cipher = AES.new(key, AES.MODE_CBC, iv)

    return cipher.decrypt(bytes.fromhex(ct))

# Copied from server.py
def next_prime(num):
    if num % 2 == 0:
        num += 1
    else:
        num += 2
    while not isPrime(num):
        num += 2
    return num

# handles getting the output from the remote server
def get_variables(host, port):
    r = remote(host, port)
    p = log.progress('Recieving data')

    r.recvuntil('Encrypted flag: ')
    flag = r.recvline().decode().strip()
    iv = r.recvline().decode().split(' ')[1].strip()
    n = r.recvline().decode().split(' ')[1].strip()
    pointA = r.recvline().decode().split(' ')
    A = Point(int(pointA[2][8:-1]), int(pointA[3][2:-2]))

    r.recvuntil('[y/n]> ')
    r.writeline('y')
    r.recvuntil('point...')
    r.recvline()
    pointG = r.recvline().decode().split(' ')
    G = Point(int(pointG[0][8:-1]), int(pointG[1][2:-2]))
    p.success('recieved')

    return (flag, iv, int(n), A, G)

HOST = '46.101.33.243'
PORT = 31203

# get values from server
encrypted_flag, iv, N, A, randG = get_variables(HOST, PORT)

#calculate variables
a, b = get_curve_constants(A, randG, N)
p, q = fermat_factorize(N)
e = next_prime(int(p) >> 128) # p is a sympy.core.numbers.Integer

# output variables so we can input them in sagemath
print(f"p: {p}")
print(f"q: {q}")
print(f"a: {a}")
print(F"b: {b}")

# get these values from the output of https://sagecell.sagemath.org/
order_Ep = int(input('Enter the order of the curve Ep: '))
order_Eq = int(input('Enter the order of the curve Eq: '))

# calculate our inverses
e_inv_order_Ep = pow(e, -1, order_Ep)
e_inv_order_Eq = pow(e, -1, order_Eq)

# construct the seperate curves mod p and q
Ep = EllipticCurve(a, b, p)
Eq = EllipticCurve(a, b, q)

# create the point on each curve
A_Ep = Point(A.x % p, A.y % p)
A_Eq = Point(A.x % q, A.y % q)

# Multiply by inverse on each curve
g_Ep = Ep.multiply(A_Ep, e_inv_order_Ep)
g_Eq = Eq.multiply(A_Eq, e_inv_order_Eq)

# combine to a point over N
g_Epq = Point(crt([g_Ep.x, p], [g_Eq.x, q]), crt([g_Ep.y, p], [g_Eq.y, q]))

# decrypt
flag = unpad(decryption(g_Epq, iv, encrypted_flag))

print(flag.decode())
```

**El Fin**
