# Oh Bob!
*(crypto60, solved by 167)*
> Alice wants to send Bob a confidential message. They both remember the crypto lecture about RSA. So Bob 
uses openssl to create key pairs. Finally, Alice encrypts the message with Bob's public keys and sends it 
to Bob. Clever Eve was able to intercept it. Can you help Eve to decrypt the message?

After unzipping the challenge, we're presented with four files, three public keys, and one file with three 
encrypted strings.
```
cat bob.pub
-----BEGIN PUBLIC KEY-----
MDgwDQYJKoZIhvcNAQEBBQADJwAwJAIdDVZLl4+dIzUElY7ti3RDcyge0UGLKfHs
+oCT2M8CAwEAAQ==
-----END PUBLIC KEY-----
```
```
cat secret.enc
DK9dt2MTybMqRz/N2RUMq2qauvqFIOnQ89mLjXY=

AK/WPYsK5ECFsupuW98bCFKYUApgrQ6LTcm3KxY=

CiLSeTUCCKkyNf8NVnifGKKS2FJ7VnWKnEdygXY=
```

The description tells us, that we have to decrypt the secret encoded messages. This is [public-key 
cryptography](https://upload.wikimedia.org/wikipedia/commons/f/f9/Public_key_encryption.svg), but all we 
have are the public keys. To decrypt the messages, we need the private keys.

Luckily, judging by the size of the base64 encoded public keys, the keys are very small. RSA is *only* 
secure when a large enough key is used at a minimum 1024-bits, and 2048-bits or more is recommended.

I've never done this before, so my thought was to first figure out what type of key this is, and then 
figure out how to extract the components of the key (asymmetric keys are not typically simple byte arrays, 
but instead several numbers concatenated together), and then see where to go from there. *This lead me no 
where.* Finding the right `openssl` commands to use on the key was proving impossible. Every command I 
found online to simply figure out the type of public key (we don't know if it's RSA, DSA, or something 
else) inside the given files was giving an error. Eventually I stumbled upon something that finally worked:
```
openssl asn1parse -dump -i -in bob2.pub
    0:d=0  hl=2 l=  56 cons: SEQUENCE          
    2:d=1  hl=2 l=  13 cons:  SEQUENCE          
    4:d=2  hl=2 l=   9 prim:   OBJECT            :rsaEncryption
   15:d=2  hl=2 l=   0 prim:   NULL              
   17:d=1  hl=2 l=  39 prim:  BIT STRING        
```
Okay, at this point at least I know it's an RSA key finally. I sort of can guess from the output of this 
that the parts of the key take up 39 bytes. I know that in RSA the "key size" is actually just the modulus 
in the RSA equation, so the key is ~256 bits - bruteforceable.

Through some luck I stumbled upon, https://warrenguy.me/blog/regenerating-rsa-private-key-python. Following 
his guide I'm able import a public key, and obtain the key's modulus and exponent (the 2 numbers stored in 
an RSA public key). The modulus can be factored into `p*q`, through bruteforce, since the modulus is so 
small - this is a significantly harder bruteforce when the modulus is 4096 bits. Using those factors, we 
then obtain the final piece of an RSA private key, the private exponent (a RSA private key is `p,q,d`, 
whereas a public key is `n,e`). `Pycrypto` has a great `construct` function that allows to easily recreate 
the corresponding private key. Applying this approach to each of Bob's public keys we can decrypt each 
message in `secret.enc`. 

Some additional finagling has to be done to apply each recovered private key to each part of the 
`secret.enc` file. For that I just used simple copy pasting into 3 separate files. Padding is also 
typically used in crypto, and we have to account for that. And finally, the authors either messed up or 
intentionally (or maybe I messed something up?) ordered the messages in `secret.enc` as corresponding to 
`bob.pub`,`bob3.pub`, `bob2.pub` in that order.

```
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_v1_5
import gmpy 
import base64
from Crypto import Random
from Crypto.Hash import SHA

bob1 = """-----BEGIN PUBLIC KEY-----
MDgwDQYJKoZIhvcNAQEBBQADJwAwJAIdDVZLl4+dIzUElY7ti3RDcyge0UGLKfHs
+oCT2M8CAwEAAQ==
-----END PUBLIC KEY-----"""
bob2 = """-----BEGIN PUBLIC KEY-----
MDgwDQYJKoZIhvcNAQEBBQADJwAwJAIdCiM3Dn0PsAIyFkrG1kKED8VOkgJDP5J6
YOta29kCAwEAAQ==
-----END PUBLIC KEY-----"""
bob3 = """-----BEGIN PUBLIC KEY-----
MDgwDQYJKoZIhvcNAQEBBQADJwAwJAIdDFtp4ZeeVB+F2s3iqhTSciqEb0Gz24Pm
Z+Oz0R0CAwEAAQ==
-----END PUBLIC KEY-----"""

pub1 = RSA.importKey(bob1)
pub2 = RSA.importKey(bob2)
pub3 = RSA.importKey(bob3)

n1 = long(pub1.n)
e1 = long(pub1.e)
n2 = long(pub2.n)
e2 = long(pub2.e)
n3 = long(pub3.n)
e3 = long(pub3.e)

# Obtained using msieve. Should probably fully pythonize this by using pysieve.
p1 = 17963604736595708916714953362445519
q1 = 20016431322579245244930631426505729
p2 = 16514150337068782027309734859141427
q2 = 16549930833331357120312254608496323
p3 = 17357677172158834256725194757225793
q3 = 19193025210159847056853811703017693

d1 = long(gmpy.invert(e1,(p1-1)*(q1-1)))
d2 = long(gmpy.invert(e2,(p2-1)*(q2-1)))
d3 = long(gmpy.invert(e3,(p3-1)*(q3-1)))

key1 = RSA.construct((n1,e1,d1))
key2 = RSA.construct((n2,e2,d2))
key3 = RSA.construct((n3,e3,d3))

# THE KEYS WERE OUT OF ORDER!!! AAARGH!
with open('secret1.bin','r') as f1:
    enc1 = f1.read()
with open('secret3.bin','r') as f2:
    enc2 = f2.read()
with open('secret2.bin','r') as f3:
    enc3 = f3.read()

dsize = SHA.digest_size
sentinel = Random.new().read(15+dsize)

cipher = PKCS1_v1_5.new(key1)
secret = cipher.decrypt(enc1, sentinel)
print("--- secret 1 ---")
print(secret)

cipher = PKCS1_v1_5.new(key2)
secret = cipher.decrypt(enc2, sentinel)
print("--- secret 2 ---")
print(secret)

cipher = PKCS1_v1_5.new(key3)
secret = cipher.decrypt(enc3, sentinel)
print("--- secret 3 ---")
print(secret)
```
```
IW{WEAK_RSA_K3YS_4R3_SO_BAD!}
```
