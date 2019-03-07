---
title = "ss protocol"
tags = ["cryptography", "network"]
date = "2018-05-04"
menu = "main"
---

# Protocol
以下引用自[ss官网](https://shadowsocks.org/en/index.html)

Ss is a secure split proxy loosely based on SOCKS5.

    client <-|SOCKS5|-> ss-local <--[encrypted]--> ss-remote <---> target

The Shadowsocks local component (ss-local) acts like a traditional SOCKS5 server and provides proxy service to clients. It encrypts and forwards data streams and packets from the client to the Shadowsocks remote component (ss-remote), which decrypts and forwards to the target. Replies from target are similarly encrypted and relayed by ss-remote back to ss-local, which decrypts and eventually returns to the original client.
## Addressing

Addresses' format:

    [1-byte type][variable-length host][2-byte port]

The following address types are defined:

    0x01: host is a 4-byte IPv4 address.
    0x03: host is a variable length string, starting with a 1-byte length, followed by up to 255-byte domain name.
    0x04: host is a 16-byte IPv6 address.

The port number is a 2-byte big-endian unsigned integer.

## TCP

ss-local initiates a TCP connection to ss-remote by sending an encrypted data stream starting with the target address followed by payload data. The exact encryption scheme differs depending on the cipher used.

[target address][payload]

ss-remote receives the encrypted data stream, decrypts and parses the leading target address. It then establishes a new TCP connection to the target and forwards payload data to it. ss-remote receives reply from the target, encrypts and forwards it back to the ss-local, until ss-local disconnects.

## UDP

ss-local sends an encrypted data packet containing the target address and payload to ss-remote.

[target address][payload]

Upon receving the encrypted packet, ss-remote decrypts and parses the target address. It then sends a new data packet containing only the payload to the target. ss-remote receives data packets back from target and prepends the target address to the payload in each packet, then sends encrypted copies back to ss-local.

[target address][payload]

Essentially, ss-remote is performing Network Address Translation for ss-local.
# AEAD Ciphers

[AEAD](https://en.wikipedia.org/wiki/Authenticated_encryption) stands for Authenticated Encryption with Associated Data. AEAD ciphers simultaneously provide **confidentiality, integrity, and authenticity**. They have excellent performance and power efficiency on modern hardware. Users should use AEAD ciphers whenever possible.

The following AEAD ciphers are recommended. Compliant Shadowsocks implementations must support AEAD_CHACHA20_POLY1305. Implementations for devices with hardware AES acceleration should also implement AEAD_AES_128_GCM, AEAD_AES_192_GCM, and AEAD_AES_256_GCM.

|Name|	Alias|	Key Size	|Salt Size|	Nonce Size	|Tag Size|
|----|-------|--------|--|-------|--------|------|
|AEAD_CHACHA20_POLY1305	|chacha20-ietf-poly1305|	32	|32	|12	|16|
|AEAD_AES_256_GCM	|aes-256-gcm	|32	|32	|12	|16|
|AEAD_AES_192_GCM	|aes-192-gcm	|24|	24	|12	|16|
|AEAD_AES_128_GCM	|aes-128-gcm	|16|	16|	12	|16|

Please refer to IANA AEAD registry for naming scheme and specification.

The way Shadowsocks using AEAD ciphers is specified in SIP004 and amended in SIP007. SIP004 was proposed by @Mygod with design inspirations from @wongsyrone, @Noisyfox and @breakwa11. SIP007 was proposed by @riobard with input from @madeye, @Mygod, @wongsyrone, and many others.

## Key Derivation

The master key can be input directly from user or generated from a password. The key derivation is still following EVP_BytesToKey(3) in OpenSSL. The detailed spec can be found [here](https://wiki.openssl.org/index.php/Manual:EVP_BytesToKey(3)).

HKDF_SHA1 is a function that takes a secret key, a non-secret salt, an info string, and produces a subkey that is cryptographically strong even if the input secret key is weak.

    HKDF_SHA1(key, salt, info) => subkey

The info string binds the generated subkey to a specific application context. In our case, it must be the string "ss-subkey" without quotes.

We derive a per-session subkey from a pre-shared master key using HKDF_SHA1. Salt must be unique through the entire life of the pre-shared master key.
Authenticated Encryption/Decryption

AE_encrypt is a function that takes a secret key, a non-secret nonce, a message, and produces ciphertext and authentication tag. Nonce must be unique for a given key in each invocation.

    AE_encrypt(key, nonce, message) => (ciphertext, tag)

AE_decrypt is a function that takes a secret key, non-secret nonce, ciphertext, authentication tag, and produces original message. If any of the input is tampered with, decryption will fail.

    AE_decrypt(key, nonce, ciphertext, tag) => message

## TCP

An AEAD encrypted TCP stream starts with a randomly generated salt to derive the per-session subkey, followed by any number of encrypted chunks. Each chunk has the following structure:

    [encrypted payload length][length tag][encrypted payload][payload tag]

Payload length is a 2-byte big-endian unsigned integer capped at 0x3FFF. The higher two bits are reserved and must be set to zero. Payload is therefore limited to 16*1024 - 1 bytes.

The first AEAD encrypt/decrypt operation uses a counting nonce starting from 0. After each encrypt/decrypt operation, the nonce is incremented by one as if it were an unsigned little-endian integer. Note that each TCP chunk involves two AEAD encrypt/decrypt operation: one for the payload length, and one for the payload. Therefore each chunk increases the nonce twice.
## UDP

An AEAD encrypted UDP packet has the following structure

    [salt][encrypted payload][tag]

The salt is used to derive the per-session subkey and must be generated randomly to ensure uniqueness. Each UDP packet is encrypted/decrypted independently, using the derived subkey and a nonce with all zero bytes.

# Stream Ciphers

Stream ciphers provide only confidentiality. Data integrity and authenticity is not guaranteed. Users should use AEAD ciphers whenever possible.

The following stream ciphers provide reasonable confidentiality.
    
|Name	|Key Size|	IV Length|
|-|-|-|
|aes-128-ctr	|16|	16|
|aes-192-ctr	|24	|16|
|aes-256-ctr	|32	|16|
|aes-128-cfb|	16|	16|
|aes-192-cfb|	24|	16|
|aes-256-cfb|	32|	16|
|camellia-128-cfb	|16	|16|
|camellia-192-cfb	|24	|16|
|camellia-256-cfb	|32	|16|
|chacha20-ietf|	32|	12|

The following stream ciphers have inherent weaknesses (see discussion at #36). DO NOT USE. Implementors are advised to remove them as soon as possible.
|Name	|Key Size	|IV Length|
|-|-|-|
|bf-cfb	|16	|8|
|chacha20|	32|	8|
|salsa20|	32|	8|
|rc4-md5|	16|	16|

## Stream Encryption/Decryption

Stream_encrypt is a function that takes a secret key, an initialization vector, a message, and produces a ciphertext with the same length as the message.

    Stream_encrypt(key, IV, message) => ciphertext

Stream_decrypt is a function that takes a secret key, an initializaiton vector, a ciphertext, and produces the original message.

    Stream_decrypt(key, IV, ciphertext) => message

The key can be input directly from user or generated from a password. The key derivation is following EVP_BytesToKey(3) in OpenSSL. The detailed spec can be found here: https://wiki.openssl.org/index.php/Manual:EVP_BytesToKey(3)
## TCP

A stream cipher encrypted TCP stream starts with a randomly generated initializaiton vector, followed by encrypted payload data.

    [IV][encrypted payload]

## UDP

A stream cipher encrypted UDP packet has the following structure

    [IV][encrypted payload]

Each UDP packet is encrypted/decrypted independently with a randomly generated initialization vector.