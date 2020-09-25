## cl-ssh-keys

`cl-ssh-keys` is a Common Lisp system, which provides the following
features.

* Decode OpenSSH public keys as defined in [RFC 4253][RFC 4253],
  section 6.6.
* Decode OpenSSH private private keys as defined in
  [PROTOCOL.key][PROTOCOL.key]
* Generate new private/public key pairs in OpenSSH compatible
  binary format.

## Requirements

* [Quicklisp][Quicklisp]

## Installation

Clone the [cl-ssh-keys][cl-ssh-keys] repo in
your [Quicklisp local-projects
directory][Quicklisp FAQ].

``` shell
git clone https://github.com/dnaeon/cl-ssh-keys.git
```

Load the system.

``` common-lisp
CL-USER> (ql:quickload :cl-ssh-keys)
```

## Supported Key Types

The following public and private key pairs can be decoded, encoded and
generated by `cl-ssh-keys`.

| Type    | Status    |
|---------|-----------|
| RSA     | Supported |
| DSA     | Supported |
| ED25519 | Supported |
| ECDSA   | Supported |

## Usage

The following section provides various examples showing you how to decode,
encode, and generate new OpenSSH private and public key pairs.

For additional examples, make sure to check the [test
suite](./t/test-suite.lisp).

### Public keys

A public key can be parsed from a given string using the
`SSH-KEYS:PARSE-PUBLIC-KEY` function, or from a file using the
`SSH-KEYS:PARSE-PUBLIC-KEY-FILE` function.

``` common-lisp
CL-USER> (defparameter *public-key*
           (ssh-keys:parse-public-key-file #P"~/.ssh/id_rsa.pub"))
*PUBLIC-KEY*
```

You can retrieve the comment associated with a public key by using the
`SSH-KEYS:KEY-COMMENT` accessor.

``` common-lisp
CL-USER> (ssh-keys:key-comment *public-key*)
"john.doe@localhost"
```

The key kind can be retrieved using `SSH-KEYS:KEY-KIND`.

``` common-lisp
CL-USER> (ssh-keys:key-kind *public-key*)
(:NAME "ssh-rsa" :PLAIN-NAME "ssh-rsa" :SHORT-NAME "RSA" :ID :SSH-RSA :IS-CERT NIL)
```

The number of bits for a key can be retrieved using the
`SSH-KEYS:KEY-BITS` generic function, e.g.

``` common-lisp
CL-USER> (ssh-keys:key-bits *public-key*)
3072
```

`SSH-KEYS:WITH-PUBLIC-KEY` and `SSH-KEYS:WITH-PUBLIC-KEY-FILE`
are convenient macros when working with public keys, e.g.

``` common-lisp
CL-USER> (ssh-keys:with-public-key-file (key #P"~/.ssh/id_rsa.pub")
           (format t "Comment: ~a~%" (ssh-keys:key-comment key))
           (format t "MD5 fingerprint: ~a~%" (ssh-keys:fingerprint :md5 key))
           (format t "Number of bits: ~a~%" (ssh-keys:key-bits key)))
Comment: john.doe@localhost
MD5 fingerprint: 04:02:4b:b2:43:39:a4:8e:89:47:49:6f:30:78:94:1e
Number of bits: 3072
NIL
```

### Private keys

A private keys can be parsed using the `SSH-KEYS:PARSE-PRIVATE-KEY`
function, which takes a string representing a private key in [OpenSSH
private key format][PROTOCOL.key], or you can use the
`SSH-KEYS:PARSE-PRIVATE-KEY-FILE` function, e.g.

``` common-lisp
CL-USER> (defparameter *private-key*
           (ssh-keys:parse-private-key-file #P"~/.ssh/id_rsa"))
*PRIVATE-KEY*
```

Key kind, comment and number of bits can be retrieved using
`SSH-KEYS:KEY-KIND`, `SSH-KEYS:KEY-COMMENT` and `SSH-KEYS:KEY-BITS`,
similarly to the way you would for public keys, e.g.

``` common-lisp
CL-USER> (ssh-keys:key-kind *private-key*)
(:NAME "ssh-rsa" :PLAIN-NAME "ssh-rsa" :SHORT-NAME "RSA" :ID :SSH-RSA :IS-CERT NIL)
CL-USER> (ssh-keys:key-comment *private-key*)
"john.doe@localhost"
CL-USER> (ssh-keys:key-bits *private-key*)
3072
```

OpenSSH private keys embed the public key within the binary blob of
the private key. From a private key you can get the embedded public
key using `SSH-KEYS:EMBEDDED-PUBLIC-KEY`, e.g.

``` common-lisp
CL-USER> (ssh-keys:embedded-public-key *private-key*)
#<CL-SSH-KEYS:RSA-PUBLIC-KEY {100619EAB3}>
```

You can also use the `SSH-KEYS:WITH-PRIVATE-KEY` and
`SSH-KEYS:WITH-PRIVATE-KEY-FILE` macros when working with private
keys.

``` common-lisp
CL-USER> (ssh-keys:with-private-key-file (key #P"~/.ssh/id_rsa")
           (format t "Comment: ~a~%" (ssh-keys:key-comment key))
           (format t "MD5 fingerprint: ~a~%" (ssh-keys:fingerprint :md5 key)))
Comment: john.doe@localhost
MD5 fingerprint: 04:02:4b:b2:43:39:a4:8e:89:47:49:6f:30:78:94:1e
```

### Encrypted keys

In order to parse an encrypted private key you need to provide a
passphrase, e.g.

``` common-lisp
CL-USER> (ssh-keys:with-private-key-file (key #P"~/.ssh/id_rsa" :passphrase "my-secret-password")
           (ssh-keys:key-cipher-name key))
"aes256-ctr"
```

### Changing passphrase of an encrypted key

The passphrase for an encrypted private key can be changed by setting
a new value for the passphrase using the `SSH-KEYS:KEY-PASSPHRASE`
accessor.

This example changes the passphrase for a given key and saves it on
the filesystem.

``` common-lisp
CL-USER> (ssh-keys:with-private-key-file (key #P"~/.ssh/id_rsa" :passphrase "OLD-PASSPHRASE")
           (setf (ssh-keys:key-passphrase key) "MY-NEW-PASSPHRASE")
           (ssh-keys:write-key-to-pathkey #P"~/.id_rsa-new-passphrase"))
```

### Setting passphrase for an existing un-encrypted key

In order to set a passphrase for an existing un-encrypted private key,
simply set a passphrase using the `SSH-KEYS:KEY-PASSPHRASE` accessor,
e.g.

``` common-lisp
CL-USER> (ssh-keys:with-private-key-file (key #P"~/.ssh/id_rsa")
           (setf (ssh-keys:key-passphrase key) "my-secret-password")
           (ssh-keys:write-key-to-pathkey #P"~/.id_rsa-encrypted"))
```

### Removing passphrase of an encrypted key

You can remove the passphrase of a private key and make it
un-encrypted by setting the passphrase to `nil`.

``` common-lisp
CL-USER> (ssh-keys:with-private-key-file (key #P"~/.ssh/id_rsa" :passphrase "PASSPHRASE")
           (setf (ssh-keys:key-passphrase key) nil)
           (ssh-keys:write-key-to-pathkey #P"~/.id_rsa-unencrypted"))
```

### Changing the cipher of an encrypted key

The cipher to be used for encryption of a private key be set by using
the `SSH-KEYS:KEY-CIPHER-NAME` accessor. The value should be one of
the known and supported ciphers as returned by
`SSH-KEYS:GET-ALL-CIPHER-NAMES`.

First, list the known cipher names.

``` common-lisp
CL-USER> (ssh-keys:get-all-cipher-names)
("3des-cbc" "aes128-cbc" "aes192-cbc" "aes256-cbc" "aes128-ctr" "aes192-ctr" "aes256-ctr" "none")
```

Then set a new cipher.

``` common-lisp
CL-USER> (ssh-keys:with-private-key-file (key #P"~/.ssh/id_rsa" :passphrase "PASSPHRASE")
           (setf (ssh-keys:key-cipher-name key) "3des-cbc")
           (ssh-keys:write-key-to-pathkey #P"~/.id_rsa-3des-cbc"))
```

### Changing the KDF number of iterations

By default `ssh-keygen(1)` and `cl-ssh-keys` will use `16` rounds of
iterations in order to produce an encryption key. You can set this to
a higher value, if needed, which would help against brute-force
attacks.

``` common-lisp
CL-USER> (ssh-keys:with-private-key-file (key #P"~/.ssh/id_rsa" :passphrase "PASSPHRASE")
           (setf (ssh-keys:key-kdf-rounds key) 32)
           (ssh-keys:write-key-to-pathkey #P"~/.id_rsa-stronger"))
```

### Fingerprints

Key fingerprints can be generated using the `SSH-KEYS:FINGERPRINT`
generic function.

The following examples show how to generate the SHA-256, SHA-1 and MD5
fingerprints of a given public key.

``` common-lisp
CL-USER> (ssh-keys:fingerprint :sha256 *public-key*)
"VmYpd+5gvA5Cj57ZZcI8lnFMNNic6jpnnBd0WoNG1F8"
CL-USER> (ssh-keys:fingerprint :sha1 *public-key*)
"RnLPLG93GrABjOqc6xOvVFpQXsc"
CL-USER> (ssh-keys:fingerprint :md5 *public-key*)
"04:02:4b:b2:43:39:a4:8e:89:47:49:6f:30:78:94:1e"
```

Fingerprints of private keys are computed against the embedded public
key.

### Writing Keys

A public and private key can be written in its text representation
using the `SSH-KEYS:WRITE-KEY` generic function.

``` common-lisp
CL-USER> (ssh-keys:write-key *public-key*)
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCsngzCcay+lQ+34qUeUSH2m1ZYW9B0a2rxpMmvYFcOyL/hRPJwv8XO89T0+HQIZRC+xlM3BSqdFGs+B58MYXPvo3H+p00CJN8tUjvC3VD74kiXSNxIyhBpKCY1s58RxnWS/6bPQIYfnCVBiQZnkNe1T3isxND1Y71TnbSz5QN2xBkAtiGPH0dPM89yWbZpTjTCaIOfyZn2fBBsmp0zUgEJ7o9W9Lrxs1f0Pn+bZ4PqFSEUzlub7mAQ+RpwgGeLeWIFz+o6KQJPFiuRgzQU6ZsY+wjorVefzgeqpRiWGw/bEyUDck09B4B0IWoTtIiKRzd635nOo7Lz/1XgaMZ60WZD9T/labEWcKmtp4Y7NoCkep0DyYyoAgWrco4FD1r0g4WcVbsJQt8HzRy9UaHlh6YPY/xkk0bSiljpygEiT48FxniqE+6HY+7SbC1wz5QThY+UsIiDgFcg3BljskfT8Il3hateXI2wEXqww4+a+DxcHzypclYorbQKUzdzNLZRBNk= john.doe@localhost
NIL
```

Another example, this time using a private key.

``` common-lisp
CL-USER> (ssh-keys:write-key *private-key*)
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
NIL
```

The `SSH-KEYS:WRITE-KEY` generic function takes an optional stream
parameter, so you can write your keys to a given stream, if needed.

``` common-lisp
CL-USER> (with-open-file (out #P"my-rsa-public-key" :direction :output)
           (ssh-keys:write-key *public-key* out))
NIL
```

`SSH-KEYS:WRITE-KEY-TO-PATH` is a convenience function you can use to
write keys to a given path, e.g.

``` common-lisp
CL-USER> (ssh-keys:write-key-to-path (key #P"my-rsa-public-key")
```

### Generating new private/public key pairs

The `SSH-KEYS:GENERATE-KEY-PAIR` generic function creates a new
private/public key pair of a given kind.

The generated keys are identical with what `ssh-keygen(1)` would
produce and you can use them to authenticate to remote systems.

The following example creates an RSA private/public key pair, and
saves the keys on the file system.

``` common-lisp
CL-USER> (multiple-value-bind (priv-key pub-key) (ssh-keys:generate-key-pair :rsa)
           (ssh-keys:write-key-to-path priv-key #P"~/.ssh/my-priv-rsa-key")
           (ssh-keys:write-key-to-path pub-key #P"~/.ssh/my-pub-rsa-key.pub"))
NIL
```

The following example generates DSA private/public key pairs.

``` common-lisp
CL-USER> (ssh-keys:generate-key-pair :dsa)
```

This example shows how to generate Ed25519 private/public key pairs.

``` common-lisp
CL-USER> (ssh-keys:generate-key-pair :ed25519)
```

ECDSA keys can be generated using NIST P-256, NIST P-384 or NIST P-521
curves. The following examples show how to create 256, 384 and 521 bit
ECDSA keys.

``` common-lisp
CL-USER> (ssh-keys:generate-key-pair :ecdsa-nistp256)
CL-USER> (ssh-keys:generate-key-pair :ecdsa-nistp384)
CL-USER> (ssh-keys:generate-key-pair :ecdsa-nistp521)
```

## Tests

Tests are provided as part of the `cl-ssh-keys.test` system.

In order to run the tests you can evaluate the following expressions.

``` common-lisp
CL-USER> (ql:quickload :cl-ssh-keys.test)
CL-USER> (asdf:test-system :cl-ssh-keys.test)
```

Or you can run the tests in a Docker container instead.

First, build the Docker image.

``` shell
docker build -t cl-ssh-keys .
```

Run the tests.

``` shell
docker run --rm cl-ssh-keys
```

## Contributing

`cl-ssh-keys` is hosted on [Github][cl-ssh-keys]. Please contribute by
reporting issues, suggesting features or by sending patches using pull
requests.

## Authors

* Marin Atanasov Nikolov (dnaeon@gmail.com)

## License

This project is Open Source and licensed under the [BSD
License][BSD License].

[RFC 4253]: https://tools.ietf.org/html/rfc4253
[PROTOCOL.key]: https://cvsweb.openbsd.org/src/usr.bin/ssh/PROTOCOL.key?annotate=HEAD
[Quicklisp]: https://www.quicklisp.org/beta/
[Quicklisp FAQ]: https://www.quicklisp.org/beta/faq.html
[cl-ssh-keys]: https://github.com/dnaeon/cl-ssh-keys
[BSD License]: http://opensource.org/licenses/BSD-2-Clause
