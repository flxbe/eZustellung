# eZustellung

[OSCI 1.2](https://www.xoev.de/osci-xta-3355)
is a complex standard defining a safe way to exchange information (Zustellungen)
between institutions of the german government.
I believe that a lot of the complexity in this standard is an accidental byproduct of
tightly coupling the end-to-end features (encryption and signatures) with
the communication protocol.
This coupling also incurs heavy performance penalties, resulting in additional complexity
to correct for this effect when transmitting large files.
By instead separating the end-to-end features into a transport-independent, efficient file format,
the protocol can concentrate on simply exchanging opaque data.
This project is an experimental effort to define a file format that fixes these issues.

## Design Goals

Based on the specifications described in OSCI, a `eZustellung` must support the following features:

- The format can hold multiple blocks of arbitrary data.
- The content is fully encrypted.
  The encrypted content is distributed along with the encryption key in a way that only the intended
  receiver can decrypt the data.
- The content is cryptographically signed by the sender to ensure that the data was not changed while being transmitted
  from the sender to the receiver.

For the choice of the actual encryption, I currently tend towards selecting suitable algorithms
supported by the [RustCrypto](https://github.com/RustCrypto) project, as those
are a direct dependency of several projects supported by the
[Sovereign Tech Fund](https://sovereigntechfund.de/index.html).

In addition, the format should also have the following properties:

- **space-efficient**: The file format should be able to efficiently pack arbitrary binary data without inflating the provided content.
- **stream-oriented**: The format should allow packing the data without the need to have all of the content in memory at once. Instead it should be possible to create the file on the fly while writing it to the filesystem/a socket. This would allow efficient processing of arbitrarily large files.
- **seekable**: The structure of the format allows to easily jump within a large `eZustellung`-file to the different data blocks.

## Format

```
         Start
+-+-+-+-+-+-+-+-+-+-+-+
|     Magic Number    | 4 bytes
+-+-+-+-+-+-+-+-+-+-+-+
|       Version       | 1 byte
+-+-+-+-+-+-+-+-+-+-+-+
|  Sender Cert Length | 2 bytes
+-+-+-+-+-+-+-+-+-+-+-+
|  Sender X.509 Cert  | Sender Cert Length bytes
+-+-+-+-+-+-+-+-+-+-+-+
|    Receiver Cert    | 2 bytes
|       Length        |
+-+-+-+-+-+-+-+-+-+-+-+
| Receiver X.509 Cert | Receiver Cert Length bytes
+-+-+-+-+-+-+-+-+-+-+-+
|  Symmetric Enc. Key | 32 bytes
+-+-+-+-+-+-+-+-+-+-+-+
|       Block 1       | n_1 bytes
+-+-+-+-+-+-+-+-+-+-+-+
|       Block 2       | n_2 bytes
+-+-+-+-+-+-+-+-+-+-+-+
:                     :
+-+-+-+-+-+-+-+-+-+-+-+
|       Block k       | n_k bytes
+-+-+-+-+-+-+-+-+-+-+-+
|   Signature Block   | 68 bytes
+-+-+-+-+-+-+-+-+-+-+-+
          End
```

The symmetric encryption key is asymmetrically encrypted first using the private key
corresponding to the sender certificate, and than by the public key
from the receiver certificate.
This way, only the receiver can decode the key and the origin of the
encrypted content must be the owner of the sender certificate.

Each data block encodes the total block size and is fully encrypted:

```
         Start
+-+-+-+-+-+-+-+-+-+-+-+
|      Block Size     | 8 bytes
| (must be greater 0) |
+-+-+-+-+-+-+-+-+-+-+-+
|     Block Content   | Block Size bytes
+-+-+-+-+-+-+-+-+-+-+-+
          End
```

The block content is the block body after encryption with the symmetric encryption key.
A block body is defined as follows:

```
         Start
+-+-+-+-+-+-+-+-+-+-+-+
|   Identifier Size   | 2 bytes
+-+-+-+-+-+-+-+-+-+-+-+
|    utf-8 Encoded    | Identifier Size bytes
|      Identifier     |
+-+-+-+-+-+-+-+-+-+-+-+
|        data         | Block Size - (2 + Identifier Size) bytes
+-+-+-+-+-+-+-+-+-+-+-+
          End
```

The final block contains the signature of the complete content up to this point.
Since the block size of each data block must be greater than 0, the
signature block is identified by a fixed 8 bytes 0:

```
         Start
+-+-+-+-+-+-+-+-+-+-+-+
|         0           | 8 bytes
+-+-+-+-+-+-+-+-+-+-+-+
|      Signature      | 32 bytes
+-+-+-+-+-+-+-+-+-+-+-+
          End
```
