# Pierre


In this challenge the flag is hidden in an RSA-encrypted number. The ciphertext `c` is stored as hex in the file called `flag.enc`. We are also given the public key in the form of a modulus `n` and exponent `e` stored in the file called `pierre.public_key`.

**TLDR:** Use Fermat's factorization to find `p` and `q`, reconstruct key and decipher the plaintext to get the flag.

## RSA crash course


To encrypt a message `m` with your public key, you do: `E(m) = m^e mod n`. You can then use your secret key `d` to decrypt: `D(c) = c^d mod n = (m^e)^d mod n = m`. This works because of the fact that `d` is the inverse of `e` and therefore `d = e^-1`. The modulus `n` is generated by choosing two primes `p` and `q` and computing `n = p * q`. These are called factors and **should** be chosen at random and always kept secret.
This is a very vague overview of RSA, for more info and references do checkout the Wikipedia article on the [RSA cryptosystem](https://en.wikipedia.org/wiki/RSA_(cryptosystem)).

Now that we are familiar with RSA, what can we do to decrypt the message? The wiki page states a couple of attacks against plain RSA (plain means: uses no padding mechanism; see wiki for details):

* Low exponent attack
* Coppersmith's attack (shared exponent)
* Chosen plaintext attack (CPA) on semantic security
* Exploiting the multiplicative property of RSA

Unfortunately none of the above apply for the given case. The exponent in our case is `0x10001 = 65537` which is by no means low. For Coppersmith's attack you need more than one ciphertext encrypted under distinct keys that share the exponent, but not `p` and `q`. To mount a CPA we would need to know enough about the plaintext to encrypt a similar plaintext. For exploiting the multiplicative property of RSA we would need access to an oracle that decrypts all ciphertexts except for our `c`. We have neither.

There is one attack that will always work against RSA, padded or not. If we can somehow reconstruct `p` and `q` from `n` we can reconstruct the private key `d` and decrypt any message encrypted with the given public key. This is called integer factorization and the security of RSA depends on this problem being hard to solve. This is indeed a hard problem to solve if `p` and `q` are chosen randomly...

## Reconstructing the private key

We need to somehow factor the `n` we are given. There are a bunch of factoring algorithms to choose from, but which one should we go for? Well the name of the challenge is Pierre which happens to be the first name of Fermat, who happened to invent a [method for factoring integers](https://en.wikipedia.org/wiki/Fermat's_factorization_method). After a quick DuckDuckGo search I found an implementation on [StackOverflow](https://stackoverflow.com/questions/20464561/fermat-factorisation-with-python) and reworked it to use `gmpy2`:

````python
import gmpy2

def fermat_factorise(n):
    a = gmpy2.isqrt(n)  # int(ceil(n**0.5))
    b2 = a*a - n
    b = gmpy2.isqrt(n)  # int(b2**0.5)
    count = 0
    while b*b != b2:
        a = a + 1
        b2 = a*a - n
        b = gmpy2.isqrt(b2)  # int(b2**0.5)
        count += 1
    p = a+b
    q = a-b
    return p, q
````

Since we are dealing with very large numbers we import the `gmpy2` library. This can be installed with `pip3 install gmpy2`.

Now let's take a look at the ciphertext. It is encoded as hex and represents a very large number: 

````
0x270194c61dc35055e24e5666d0e3d55a1a56101
e5671c8ba64b9bb1b134a20307180f1408e8d9451
96b1fbeaf058f6f25401e77b244d053ec9130538a
81c45a589a5919b154c7ada1f34e8aa6804218234
18950f0d66574d6dac2ea0d852b6b1e6093e28268
2019f36c20957497bab0f9f0e9fa44cdb34384aac
29654d916ede051f6d77eb29af166a4e631fbd8f1
2236980e23c34f3213f961a5c6c8bb22d1591c253
58762c9be1cd8f0332c60d095a79b417205dfbafc
80125c8cba867c3705b0daf0f3f7c38db42013203
87f27dc0e52c13b39e587d643aa9090ee7e36ac4b
95c8c0c4b7fb820e3d151c4a068aeebd29130ecdb
bdf597e99a11032770dd77
````

Running the algorithm on our ciphertext yields the two prime factors `p` and `q`:

````
p: 
13927373879178315972347413545070607646768
83132178783032667258696796439410897825195
99883739126996219650621388485987414991867
54101110094330930239372047000776721370428
26381542395279014422750729854646121680004
25163572105027092370553317488531002704964
44677055923965247554280161527759346296210
2824810226264704834429

q:
13927373879178315972347413545070607646768
83132178783032667258696796439410897825195
99883739126996219650621388485987414991867
54101110094330930239372047000776721370428
26381542395279014422750729854646121680004
25163572105027092370553317488531002704964
44677055923965247554280161527759346296210
2824810226264704766731
````

Now that we have `p` and `q` we can reconstruct the private key by referring to the wiki page for RSA describing the [key generation](https://en.wikipedia.org/wiki/RSA_(cryptosystem)#Key_generation):

1. Choose distinct primes `p`, `q`
2. Compute `n = pq`
3. Compute `phi = (p-1)(q-1)`.
4. Choose an exponent `e`
5. Compute `d = e^-1 mod phi`, that is the multiplicative inverse of `e` modulo `phi` using the [Extended Euclidean Algorithm](https://en.wikipedia.org/wiki/Extended_Euclidean_algorithm).


We now have 1, 2 and 4 and need to compute the last two. Computing `phi` is straightforward. The last point requires an implementation of the extended euclidean algorithm. This is fortunately very easy to find for basically any language you choose to program in. I borrowed an implementation from [WikiBooks](https://en.wikibooks.org/wiki/Algorithm_Implementation/Mathematics/Extended_Euclidean_algorithm):

````python
def xgcd(a, b):
    x0, x1, y0, y1 = 0, 1, 1, 0
    while a != 0:
        q, b, a = b // a, a, b % a
        y0, y1 = y1, y0 - q * y1
        x0, x1 = x1, x0 - q * x1

    return b, x0, y0
````

Now let's combine all our snippets into an attack:

````python
p, q = fermat_factorise(n)
phi = (p - 1)*(q - 1)
gcd, d, b = xgcd(e, phi)
plaintext = gmpy2.powmod(c, gmpy2.mpz(d), n)
````

The plaintext derived with our method is: `0x666c61677b52346e64304d5f7072316d34735f345f5253415f506c33347333217d`, which translates to a bunch of `0` bytes and the flag: `flag{R4nd0M_pr1m4s_4_RSA_Pl34s3!}`.

The whole program can be seen in the file `pierre.py`.

## Conclusion

Even a secure cryptosystem can be broken if used incorrectly. This is a textbook example of insecure key generation. Always use a random `p` and `q` :)
