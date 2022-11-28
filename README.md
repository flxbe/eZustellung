# eZustellung

[OSCI 1.2](https://www.xoev.de/osci-xta-3355)
is a complex standard defining a secure way to exchange information (Zustellungen)
between institutions of the german government.
I believe that a lot of the complexity in this standard is an accidental byproduct of
tightly coupling the end-to-end features (encryption and signatures) with
the SOAP-based communication protocol.
The specific details of this coupling also incur heavy performance penalties, resulting in additional complexity
to correct for this effect when transmitting large files.
By instead separating the end-to-end features into a transport-independent, efficient file format,
the protocol could concentrate on simply exchanging opaque data.
On top, the transport mechanism and the file format could develop independently,
allowing for greater flexibility in the future.
This project is an experimental effort to define a suitable file format.

## Design Goals

Based on the specifications described in OSCI, a `eZustellung` must support the following features:

- The format can hold multiple blocks of arbitrary data.
- The content is fully encrypted.
  The encrypted content is distributed along with the encryption key in a way that only the intended
  receiver can decrypt the data.
- The content is cryptographically signed by the sender to ensure that the data was not changed while being transmitted.

In addition to these requirements derived from the OSCI standard, the format should also have the following properties:

- **space-efficient**: The file format should be able to efficiently pack arbitrary binary data without inflating the provided content, i.e. no base64 encoding of the binary content.
- **stream-oriented**: The format should allow packing the data without the need to have all of the content in memory at once.
  Instead, it should be possible to create the file on the fly while writing it to the target (filesystem, network, ...).
  This would allow efficient processing of arbitrarily large files with minimal memory requirements.
- **seekable**: The structure of the format allows to easily jump within a large `eZustellung`-file to the different data blocks.
- **easy to use**: It should be possible to create an intuitive interface to interact with the format.

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
corresponding to the sender certificate, and then by the public key
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
|          0          | 8 bytes
+-+-+-+-+-+-+-+-+-+-+-+
|      Signature      | 32 bytes
+-+-+-+-+-+-+-+-+-+-+-+
          End
```

## Implementation Goals

A possible interface in Python could look like this:

```python
import eZustellung

# Somehow create a writable target for the data (file, byte array, socket, ...)
target = create_target()

sender_cert = load_certificate("...")
sender_key = load_private_key("...")
receiver_cert = load_certificate("...")

with eZustellung.Writer(target, sender_cert, sender_key, receiver_cert) as writer:
  # At this point, the certificates and the symmetric key have already been written
  # to the target. The file is now ready to write the different blocks of data,
  # which are encrypted automatically.

  writer.add_file(
    "./path/to/payload.json",
    identifier="payload_1.json",
  )

  writer.add_bytes(
    b'{"value": 1}',
    identifier="payload_2.json",
  )

  # In the OSCI standard, one discussed use case is to encrypt
  # the content so that it can only be read by combining multiple certificates.
  # This can be achieved by wrapping another eZustellung internally as a data block.
  inner_receiver_cert = load_certificate("...")
  with writer.create_inner_writer(inner_receiver_cert, identifier="internal") as inner_writer:
    inner_writer.add_bytes(
      b'{"value": 2}',
      identifier="payload_3.json"
    )

  # After leaving the context the signature block is created, marking the end of the file.
```
