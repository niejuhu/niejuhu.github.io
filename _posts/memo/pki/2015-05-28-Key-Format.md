---
title: Key format
layout: post
category: PKI
---

Some notes about key format.

# ASN.1

"ASN.1 is a formal notation used for describing data transmitted by telecommunications protocols, regardless of language implementation and physical representation of these data, whatever the application, whether complex or very simple."

Thare are two points:

1) The notation provides a certain number of pre-defined types such as INTEGER, BOOLEAN, and makes it possible to define constructed types such as SEQUENCE to describe a data protocol.

2) The encoding rules such as BER, DER, CER define the encoding and decoding rules for data transfer by telecommunications.

[The official document](http://www.itu.int/en/ITU-T/asn1/Pages/introduction.aspx)

[Tools](http://www.itu.int/en/ITU-T/asn1/Pages/Tools.aspx)


# Key format

## Public key format

Many tools use the format defined in [x509](https://tools.ietf.org/html/rfc5280#section-4.1.2.3)

    SubjectPublicKeyInfo  ::=  SEQUENCE  {
        algorithm            AlgorithmIdentifier,
        subjectPublicKey     BIT STRING
    }

    AlgorithmIdentifier  ::=  SEQUENCE  {
        algorithm               OBJECT IDENTIFIER,
        parameters              ANY DEFINED BY algorithm OPTIONAL
    }

For the rsa public key, The AlgorithmIdentifier should be:

    algorithm ::= rsaEncryption
    parameters :: = NULL

and the subjectPublicKey should be BIT STRING of RSAPublicKey. See [RFC3279](https://tools.ietf.org/html/rfc3279)

## Private key format

Defined in [pkcs#8](https://www.ietf.org/rfc/rfc5208.txt)

    PrivateKeyInfo ::= SEQUENCE {
        version                   Version,
        privateKeyAlgorithm       PrivateKeyAlgorithmIdentifier,
        privateKey                PrivateKey,
        attributes           [0]  IMPLICIT Attributes OPTIONAL
    }

    Version ::= INTEGER
    PrivateKeyAlgorithmIdentifier ::= AlgorithmIdentifier
    PrivateKey ::= OCTET STRING
    Attributes ::= SET OF Attribute

    EncryptedPrivateKeyInfo ::= SEQUENCE {
        encryptionAlgorithm  EncryptionAlgorithmIdentifier,
        encryptedData        EncryptedData
    }

    EncryptionAlgorithmIdentifier ::= AlgorithmIdentifier
    EncryptedData ::= OCTET STRING

## Rsa Public&Private key format

Defined in [pkcs#1](https://www.ietf.org/rfc/rfc3447.txt)

    RSAPublicKey ::= SEQUENCE {
        modulus           INTEGER,  -- n
        publicExponent    INTEGER   -- e
    }

    RSAPrivateKey ::= SEQUENCE {
        version           Version,
        modulus           INTEGER,  -- n
        publicExponent    INTEGER,  -- e
        privateExponent   INTEGER,  -- d
        prime1            INTEGER,  -- p
        prime2            INTEGER,  -- q
        exponent1         INTEGER,  -- d mod (p-1)
        exponent2         INTEGER,  -- d mod (q-1)
        coefficient       INTEGER,  -- (inverse of q) mod p
        otherPrimeInfos   OtherPrimeInfos OPTIONAL
    }

# Key file format

DER key file is the asn.1 DER encoding of key or certificate.

PEM key file is the base64 encoding of asn.1 DER encoding of key or certificate with text HEADER, FOOTER and other information.

