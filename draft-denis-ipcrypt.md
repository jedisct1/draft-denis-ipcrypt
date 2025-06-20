---
title: "Methods for IP Address Encryption and Obfuscation"
abbrev: "ipcrypt"
docname: draft-denis-ipcrypt-latest
category: info
ipr: trust200902
keyword: Internet-Draft
author:
  - name: "Frank Denis"
    organization: "Fastly Inc."
    email: fde@00f.net
date: "2025"
v: 3
stand_alone: yes
smart_quotes: yes
pi: [toc, sortrefs, symrefs]

normative:
  FIPS-197:
    title: "Advanced Encryption Standard (AES)"
    author:
      - ins: NIST
        org: National Institute of Standards and Technology
    date: 2001-11-26
    seriesinfo:
      FIPS: PUB 197
    target: https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.197.pdf
  NIST-SP-800-38G:
    title: "Recommendation for Block Cipher Modes of Operation: Methods for Format-Preserving Encryption"
    author:
      - ins: NIST
        org: National Institute of Standards and Technology
    date: 2016-03
    seriesinfo:
      NIST: SP 800-38G
    target: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-38G.pdf

informative:
  DEOXYS-BC:
    title: "Deoxys-BC: A Highly Secure Tweakable Block Cipher"
    author:
      - ins: J. Jean
      - ins: I. Nikolić
      - ins: T. Peyrin
    date: 2014
    seriesinfo:
      Cryptology ePrint Archive: Paper 2014/427
    target: https://eprint.iacr.org/2014/427
    eprint: 2014/427
  SKINNY:
    title: "The SKINNY Family of Block Ciphers and its Low-Latency Variant MANTIS"
    author:
      - ins: C. Beierle
      - ins: A. Biryukov
      - ins: L. Perrin
      - ins: A. Udovenko
      - ins: V. Velichkov
      - ins: Q. Wang
    date: 2016
    seriesinfo:
      CRYPTO: 2016
    target: https://eprint.iacr.org/2016/660
    eprint: 2016/660
  LRW2002:
    title: "Tweakable Block Ciphers"
    author:
      - ins: M. Liskov
      - ins: R. Rivest
      - ins: D. Wagner
    date: 2002
    seriesinfo:
      Fast Software Encryption: 2002
    target: https://www.cs.berkeley.edu/~daw/papers/tweak-crypto02.pdf
    doi: 10.1007/3-540-45661-9_17
  IEEE-P1619:
    title: "IEEE Standard for Cryptographic Protection of Data on Block-Oriented Storage Devices"
    author:
      - ins: IEEE
    date: 2007-12-18
    seriesinfo:
      IEEE: 1619-2007
    target: https://standards.ieee.org/ieee/1619/2041/
  BRW2005:
    title: "Format-Preserving Encryption"
    author:
      - ins: M. Bellare
      - ins: P. Rogaway
      - ins: D. Wagner
    date: 2005
    seriesinfo:
      CRYPTO: 2005
    target: https://www.cs.ucdavis.edu/~rogaway/papers/subset.pdf
    doi: 10.1007/11535218_24
  KIASU-BC:
    title: "Tweaks and Keys for Block Ciphers: the TWEAKEY Framework"
    author:
      - ins: J. Jean
      - ins: I. Nikolić
      - ins: T. Peyrin
    date: 2014
    seriesinfo:
      Cryptology ePrint Archive: Paper 2014/831
    target: https://eprint.iacr.org/2014/831
    eprint: 2014/831
  XTS-AES:
    title: "The XTS-AES Mode for Disk Encryption"
    author:
      - ins: J. Black
      - ins: E. Dawson
      - ins: S. Gueron
      - ins: P. Rogaway
    date: 2010
    seriesinfo:
      IEEE: 1619-2007
    doi: 10.1109/TC.2010.58
  IPCRYPT2:
    title: "ipcrypt2: IP address encryption/obfuscation tool"
    author:
      - ins: F. Denis
    date: 2025
    target: https://github.com/jedisct1/ipcrypt2
    license: ISC

--- abstract

This document specifies methods for encrypting and obfuscating IP addresses, providing both deterministic format‑preserving and non‑deterministic constructions. These methods address privacy concerns raised in {{!RFC6973}} and {{!RFC7258}} regarding pervasive monitoring and data collection.

The methods apply uniformly to both IPv4 and IPv6 addresses by converting them into a 16‑byte representation. Two generic constructions are defined—one using a 128‑bit block cipher and the other using a 128‑bit tweakable block cipher—along with three concrete instantiations:

- **`ipcrypt-deterministic`:** Deterministic encryption using AES128 (applied as a single‑block operation).
- **`ipcrypt-nd`:** Non‑deterministic encryption using the KIASU‑BC tweakable block cipher with an 8‑byte tweak.
- **`ipcrypt-ndx`:** Non‑deterministic encryption using the AES‑XTS tweakable block cipher with a 16‑byte tweak.

Deterministic mode produces a 16‑byte ciphertext (enabling format preservation), while non‑deterministic modes prepend a randomly sampled tweak (which MUST be uniformly random when generated, as specified in {{!RFC4086}}) to produce larger ciphertexts that resist correlation attacks.

--- middle

# Introduction

This document specifies methods for the encryption and obfuscation of IP addresses for both operational use and privacy preservation. The objective is to enable network operators, researchers, and privacy advocates to share or analyze data while protecting sensitive address information.

This work addresses concerns raised in {{!RFC7624}} regarding confidentiality in the face of pervasive surveillance. For a detailed discussion of the security properties of these methods, see {{security-considerations}}.

## Use Cases and Motivations

The main motivations include:

- **Privacy Protection:** Encrypting IP addresses prevents the disclosure of user-specific information when data is logged or measured, as discussed in {{!RFC6973}}.

- **Format Preservation:** Ensuring that the encrypted output remains a valid IP address allows network devices to process the data without modification. See {{format-preservation}} for details.

- **Mitigation of Correlation Attacks:** Deterministic encryption reveals repeated inputs; non‑deterministic modes use a random tweak to obscure linkability while keeping the underlying input confidential. See {{non-deterministic-encryption}} for implementation details.

- **Privacy-Preserving Analytics:** Many common operations like counting unique clients or implementing rate limiting can be performed using encrypted IP addresses without ever accessing the original values. This enables privacy-preserving analytics while maintaining functionality.

- **Third-Party Service Integration:** IP addresses are private information that should not be sent in cleartext to potentially untrusted third-party services or cloud providers. Using encrypted IP addresses as keys or identifiers allows integration with external services while protecting user privacy.

For implementation examples, see {{pseudocode-and-examples}}.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{!RFC8174}} when, and only when, they appear in all capitals, as shown here.

Throughout this document, the following terms and conventions apply:

- **IP Address:** An IPv4 or IPv6 address as defined in {{!RFC4291}}.
- **16‑Byte Representation:** A fixed-length representation used for both IPv4 (via IPv4‑mapped IPv6) and IPv6 addresses.
- **Tweak:** A non‑secret, additional input to a tweakable block cipher that further randomizes the output.
- **Deterministic Encryption:** Encryption that always produces the same ciphertext for a given input and key.
- **Non‑Deterministic Encryption:** Encryption that produces different ciphertexts for the same input due to the inclusion of a randomly sampled tweak.
- **(Input, Tweak) Collision:** A scenario where the same input is encrypted with the same tweak; this reveals that the input was repeated but not the input's value.

# IP Address Conversion

This section describes the conversion of IP addresses to and from a 16‑byte representation. This conversion is necessary to operate a 128‑bit cipher on both IPv4 and IPv6 addresses.

## Converting to a 16‑Byte Representation

### IPv6 Addresses

IPv6 addresses are natively 128 bits and are converted directly using network‑byte order (big‑endian) as specified in {{!RFC4291}}.

_Example:_

~~~
IPv6 Address:    2001:0db8:85a3:0000:0000:8a2e:0370:7334
16-Byte Representation: [20 01 0d b8 85 a3 00 00 00 00 8a 2e 03 70 73 34]
~~~

### IPv4 Addresses

IPv4 addresses (32 bits) are mapped using the IPv4‑mapped IPv6 format as specified in {{!RFC4291}}:

~~~
IPv4 Address:    192.0.2.1
16-Byte Representation: [00 00 00 00 00 00 00 00 00 00 FF FF C0 00 02 01]
~~~

## Converting from a 16‑Byte Representation to an IP Address

The conversion algorithm is as follows:

1. Examine the first 12 bytes of the 16-byte representation
2. If they match the IPv4‑mapped prefix (10 bytes of `0x00` followed by `0xFF``, `0xFF``):
   - Interpret the last 4 bytes as an IPv4 address in dotted‑decimal notation
3. Otherwise:
   - Interpret the 16 bytes as an IPv6 address in colon‑hexadecimal notation

(For additional illustration, see {{diagrams}})

# Generic Constructions

This specification defines two generic cryptographic constructions:

1. **128-bit Block Cipher Construction:**
   - Used in deterministic encryption (see {{deterministic-encryption}})
   - Operates on a single 16-byte block
   - Example: AES‑128 treated as a permutation

2. **128-bit Tweakable Block Cipher (TBC) Construction:**
   - Used in non‑deterministic encryption (see {{non-deterministic-encryption}})
   - Accepts a key, a tweak, and a message
   - The tweak is typically randomly sampled (and MUST be uniformly random when generated)
   - Reuse of the same tweak on different inputs does not compromise confidentiality

Valid options for implementing a tweakable block cipher include, but are not limited to:

- **SKINNY** (see {{SKINNY}})
- **DEOXYS-BC** (see {{DEOXYS-BC}})
- **KIASU-BC** (see {{implementing-kiasu-bc}} for implementation details)
- **AES-XTS** (see {{ipcrypt-ndx}} for usage)

Implementers MUST choose a cipher that meets the required security properties and provides robust resistance against related-tweak and other cryptographic attacks.

# Deterministic Encryption

Deterministic encryption applies a 128‑bit block cipher directly to the 16‑byte representation of an IP address. For implementation details, see {{pseudocode-and-examples}}.

> **Note:**
> All instantiations documented in this specification (`ipcrypt-deterministic`, `ipcrypt-nd`, and `ipcrypt-ndx`)
> are invertible - encrypted IP addresses can be decrypted back to their original values using the same key.
> For non-deterministic modes, the tweak must be preserved along with the ciphertext to enable decryption.

## ipcrypt-deterministic

The `ipcrypt-deterministic` instantiation employs AES128 in a single‑block operation. The key MUST be exactly 16 bytes (128 bits) in length. Since AES128 is a permutation, every distinct 16‑byte input maps to a unique 16‑byte ciphertext, preserving the IP address format.

For test vectors, see {{ipcrypt-deterministic-test-vectors}}.

~~~
      +---------------------+
      |      IP Address     |
      |    (IPv4 or IPv6)   |
      +---------------------+
                 |
                 v
      +---------------------+
      | Convert to 16 Bytes |
      +---------------------+
                 |
                 v
      +---------------------+
      |   AES128 Encrypt    |
      |   (Single Block)    |
      +---------------------+
                 |
                 v
      +---------------------+
      |    16-Byte Output   |
      +---------------------+
                 |
                 v
      +---------------------+
      | Convert to IP Format|
      +---------------------+
~~~

## Format Preservation

- If the 16‑byte ciphertext begins with an IPv4‑mapped prefix, it **MUST** be rendered as a dotted‑decimal IPv4 address.
- Otherwise, it is interpreted as an IPv6 address.

> **Note:**
> To ensure IPv4 format preservation, implementers **MUST** consider using cycle‑walking, a 32-bit random permutation, or an FPE mode if required.

# Non‑Deterministic Encryption {#non-deterministic-encryption}

Non‑deterministic encryption leverages a tweakable block cipher together with a random tweak. For implementation details, see {{pseudocode-and-examples}}.

> **Alternatives to Random Tweaks:**
> While this specification recommends the use of uniformly random tweaks for non-deterministic encryption, other approaches are possible.
> For example, a monotonic counter could be used as a tweak, but this is difficult to maintain in distributed systems and, if the counter is not encrypted and the tweakable block cipher is not secure against related-tweak attacks, this could enable correlation attacks.
> Another alternative is to use UUIDs (such as UUIDv6 or UUIDv7) as tweaks; however, these would reveal the original timestamp of the logged IP addresses, which may not be desirable from a privacy perspective.
> Although the birthday bound is a concern with random tweaks, the use of random tweaks remains the recommended and most practical approach, offering the best tradeoffs for most real-world use cases.

## Encryption Process

The encryption process for non-deterministic modes consists of the following steps:

1. Generate a random tweak using a cryptographically secure random number generator
2. Convert the IP address to its 16-byte representation
3. Encrypt the 16-byte representation using the key and the tweak
4. Concatenate the tweak with the encrypted output to form the final ciphertext

The tweak is not considered secret and is included in the ciphertext. This allows the same tweak to be used for decryption.

## Decryption Process

The decryption process consists of the following steps:

1. Split the ciphertext into the tweak and the encrypted IP
2. Decrypt the encrypted IP using the key and the tweak
3. Convert the resulting 16-byte representation back to an IP address

Although the tweak is generated uniformly at random (and thus may occasionally collide per birthday bounds), such collisions are benign when they occur with different inputs. An `(input, tweak)` collision reveals that the same input was encrypted with the same tweak but does not disclose the input's value.

The usage limits discussed below apply per cryptographic key; rotating keys can extend secure usage beyond these bounds.

## Output Format and Encoding

The output of non-deterministic encryption is binary data. For applications that require text representation (e.g., logging, JSON encoding, or text-based protocols), the binary output MUST be encoded. Common encoding options include hexadecimal and Base64.

The choice of encoding is application-specific and outside the scope of this specification. However, implementations SHOULD document their chosen encoding method clearly.

## Concrete Instantiations

This document defines two concrete instantiations:

- **`ipcrypt-nd`:** Uses the KIASU‑BC tweakable block cipher with an 8‑byte (64‑bit) tweak.
  See [KIASU-BC] for details.
- **`ipcrypt-ndx`:** Uses the AES‑XTS tweakable block cipher with a 16‑byte (128‑bit) tweak.
  See [XTS-AES] for background. Since only a single block is encrypted, only the first tweak
  needs to be computed, avoiding the need for a full key schedule.

In both cases, if a tweak is generated randomly, it **MUST be uniformly random**. Reusing the same randomly generated tweak on different inputs is acceptable from a confidentiality standpoint.

For test vectors, see {{ipcrypt-nd-test-vectors}} and {{ipcrypt-ndx-test-vectors}}.

### ipcrypt-nd (KIASU‑BC) {#ipcrypt-nd}

The `ipcrypt-nd` instantiation uses the KIASU‑BC tweakable block cipher with an 8‑byte (64‑bit) tweak. For implementation details, see {{implementing-kiasu-bc}}. The output is 24 bytes total, consisting of an 8‑byte tweak concatenated with a 16‑byte ciphertext.

Random sampling of an 8‑byte tweak yields an expected collision for a specific tweak value after about 2^(64/2) = 2^32 operations. If an `(input, tweak)` collision occurs, it indicates that the same input was processed with that tweak without revealing the input's value.

These collision bounds apply per cryptographic key. By rotating keys regularly, secure usage can be extended well beyond these bounds. Ultimately, the effective security is determined by the underlying block cipher's strength.

For test vectors, see {{ipcrypt-nd-test-vectors}}.

### ipcrypt-ndx (AES‑XTS) {#ipcrypt-ndx}

The `ipcrypt-ndx` instantiation uses the AES‑XTS tweakable block cipher with a 16‑byte (128‑bit) tweak. The output is 32‑byte total, consisting of a 16‑byte tweak concatenated with a 16‑byte ciphertext.

Since only a single block is encrypted, only the first tweak needs to be computed, avoiding the need for a full key schedule. Independent sampling of a 16‑byte tweak results in an expected collision after about 2^(128/2) = 2^64 operations.

As with `ipcrypt-nd`, an `(input, tweak)` collision reveals repetition without compromising the input value. These limits are per key, and regular key rotation further extends secure usage. The effective security is governed by the strength of AES‑128 (approximately 2^128 operations).

> **Technical Note:**
> For a single block of AES-XTS, the key is split into two halves (K1, K2). The tweak is
> first encrypted using AES128 with K2 to produce an encrypted tweak (ET). The IP address
> is then encrypted as: AES128(IP ⊕ ET, K1) ⊕ ET (where ⊕ denotes the bitwise XOR operation).
> This construction provides the security properties of XTS while only requiring two AES
> operations per block.

~~~pseudocode
function AES_XTS_encrypt(key, tweak, block):
    // Split the key into two halves
    K1, K2 = split_key(key)

    // Encrypt the tweak with the second half of the key
    ET = AES128_encrypt(K2, tweak)

    // Encrypt the block: AES128(block ⊕ ET, K1) ⊕ ET
    return AES128_encrypt(K1, block ⊕ ET) ⊕ ET
~~~

### Comparison of Modes

- **Deterministic (`ipcrypt-deterministic`):**
  Produces a 16‑byte output; preserves format but reveals repeated inputs.
- **Non‑Deterministic:**
  - **`ipcrypt-nd` (KIASU‑BC):** Produces a 24‑byte output using an 8‑byte tweak; `(input, tweak)` collisions reveal repeated inputs (with the same tweak) but not their values.
  - **`ipcrypt-ndx` (AES‑XTS):** Produces a 32‑byte output using a 16‑byte tweak; supports higher secure operation counts per key. Since only a single block is encrypted, it avoids the need for a full key schedule.

## Security Considerations

For a detailed discussion of the security properties of each mode, see:

- {{deterministic-encryption}} for deterministic mode security considerations
- {{ipcrypt-nd}} and {{ipcrypt-ndx}} for non-deterministic mode security considerations

The ipcrypt constructions focus solely on confidentiality and do not provide integrity. This means that IP addresses in an ordered sequence can be partially removed, duplicated, reordered, or blindly altered by an active adversary.

Applications that require sequences of encrypted IP addresses that cannot be modified must apply an authentication scheme over the entire sequence, such as a HMAC construction or a keyed hash function, or a public key signature.

This is outside the scope of this specification, but implementers should be aware that additional authentication mechanisms are required if protection against active adversaries is needed.

### Deterministic Mode Security

A permutation ensures distinct inputs yield distinct outputs. However, repeated inputs result in identical ciphertexts, thereby revealing repetition.

This property makes deterministic encryption suitable for applications where format preservation is required, but linkability of repeated inputs is acceptable.

### Non-Deterministic Mode Security

The inclusion of a random tweak ensures that encrypting the same input generally produces different outputs. In cases where an `(input, tweak)` collision occurs, an attacker learns only that the same input was processed with that tweak, not the value of the input itself.

Security is determined by the underlying block cipher (≈2^128 for AES‑128) on a per-key basis. Key rotation is recommended to extend secure usage beyond the per-key collision bounds.

### Implementation Security

Implementations MUST ensure that:

1. Keys are generated using a cryptographically secure random number generator
2. Tweak values are uniformly random for non-deterministic modes
3. Side-channel attacks are mitigated through constant-time operations
4. Error handling does not leak sensitive information

# Implementation Status

_This note is to be removed before publishing as an RFC._

Multiple implementations of the schemes described in this document have been developed and verified for interoperability.

A comprehensive list of known implementations and integrations can be found at [](https://github.com/jedisct1/draft-denis-ipcrypt), which includes reference implementations closely aligned with the pseudocode provided in this document.

# IANA Considerations

This document does not require any IANA actions.

--- back

# Diagrams {#diagrams}

This appendix provides visual representations of the key operations described in this document. For implementation details, see {{pseudocode-and-examples}}.

## IPv4 Address Conversion Diagram {#ipv4-address-conversion-diagram}

~~~
       IPv4: 192.0.2.1
           |
           v
  Octets:  C0  00  02  01
           |
           v
   16-Byte Array:
[00 00 00 00 00 00 00 00 00 00 | FF FF | C0 00 02 01]
~~~

## Deterministic Encryption Flow

~~~
            IP Address
                |
                v
       [Convert to 16 Bytes]
                |
                v
    [AES128 Single-Block Encrypt]
                |
                v
       16-Byte Ciphertext
                |
                v
     [Convert to IP Format]
                |
                v
       Encrypted IP Address
~~~

## Non‑Deterministic Encryption Flow (ipcrypt-nd)

~~~
              IP Address
                  |
                  v
      [Convert to 16 Bytes] ---> 16-Byte Representation
                  |
                  v
    [Generate Random 8-Byte Tweak]
                  |
                  v
       [KIASU-BC Tweakable Encrypt]
                  |
                  v
          16-Byte Ciphertext
                  |
                  v
    [Concatenate Tweak || Ciphertext]
                  |
                  v
       24-Byte Output (ipcrypt-nd)
~~~

## Non‑Deterministic Encryption Flow (ipcrypt-ndx)

~~~
              IP Address
                  |
                  v
      [Convert to 16 Bytes] ---> 16-Byte Representation
                  |
                  v
    [Generate Random 16-Byte Tweak]
                  |
                  v
       [AES-XTS Tweakable Encrypt]
                  |
                  v
          16-Byte Ciphertext
                  |
                  v
    [Concatenate Tweak || Ciphertext]
                  |
                  v
       32-Byte Output (ipcrypt-ndx)
~~~

# Pseudocode and Examples {#pseudocode-and-examples}

This appendix provides detailed pseudocode for key operations described in this document. For a visual representation of these operations, see {{diagrams}}.

## IPv4 Address Conversion

For a diagram of this conversion process, see {{ipv4-address-conversion-diagram}}.

~~~pseudocode
function IPv4To16Bytes(ipv4_address):
    // Split the IPv4 address into its octets
    parts = ipv4_address.split(".")
    if length(parts) != 4:
         raise Error("Invalid IPv4 address")
    // Create a 16-byte array with the IPv4-mapped prefix
    bytes16 = [0x00] * 10         // 10 bytes of 0x00
    bytes16.append(0xFF)          // 11th byte: 0xFF
    bytes16.append(0xFF)          // 12th byte: 0xFF
    // Append each octet (converted to an 8-bit integer)
    for part in parts:
         bytes16.append(int(part))
    return bytes16
~~~

_Example:_ For `"192.0.2.1"`, the function returns

~~~
[00, 00, 00, 00, 00, 00, 00, 00, 00, 00, FF, FF, C0, 00, 02, 01]
~~~

## IPv6 Address Conversion

~~~pseudocode
function IPv6To16Bytes(ipv6_address):
    // Parse the IPv6 address into eight 16-bit words.
    words = parseIPv6(ipv6_address)  // Expands shorthand notation and returns 8 words
    bytes16 = []
    for word in words:
         high_byte = (word >> 8) & 0xFF
         low_byte = word & 0xFF
         bytes16.append(high_byte)
         bytes16.append(low_byte)
    return bytes16
~~~

_Example:_ For `"2001:0db8:85a3:0000:0000:8a2e:0370:7334"`, the output is the corresponding 16‑byte sequence.

## Conversion from a 16-Byte Array to an IP Address

~~~pseudocode
function Bytes16ToIP(bytes16):
    if length(bytes16) != 16:
         raise Error("Invalid byte array")

    // Check for the IPv4-mapped prefix
    if bytes16[0:10] == [0x00]*10 and bytes16[10] == 0xFF and bytes16[11] == 0xFF:
         ipv4_parts = []
         for i from 12 to 15:
             ipv4_parts.append(str(bytes16[i]))
         ipv4_address = join(ipv4_parts, ".")
         return ipv4_address
    else:
         words = []
         for i from 0 to 15 step 2:
             word = (bytes16[i] << 8) | bytes16[i+1]
             words.append(format(word, "x"))
         ipv6_address = join(words, ":")
         return ipv6_address
~~~

## Deterministic Encryption (ipcrypt-deterministic)

~~~pseudocode
function ipcrypt_deterministic(ip_address, key):
    // The key MUST be exactly 16 bytes (128 bits) in length
    if length(key) != 16:
        raise Error("Key must be 16 bytes")

    bytes16 = convertTo16Bytes(ip_address)
    ciphertext = AES128_encrypt(key, bytes16)
    encrypted_ip = Bytes16ToIP(ciphertext)
    return encrypted_ip
~~~

## Non‑Deterministic Encryption using KIASU‑BC (ipcrypt-nd)

~~~pseudocode
function ipcrypt_nd_encrypt(ip_address, key):
    // The key MUST be exactly 16 bytes (128 bits) in length
    if length(key) != 16:
        raise Error("Key must be 16 bytes")

    // Step 1: Generate random tweak (MUST be exactly 8 bytes)
    tweak = random_bytes(8)  // MUST be uniformly random

    // Step 2: Convert IP to 16-byte representation
    bytes16 = convertTo16Bytes(ip_address)

    // Step 3: Encrypt using key and tweak
    ciphertext = KIASU_BC_encrypt(key, tweak, bytes16)

    // Step 4: Concatenate tweak and ciphertext
    result = concatenate(tweak, ciphertext)  // 8 bytes || 16 bytes = 24 bytes total
    return result

function ipcrypt_nd_decrypt(ciphertext, key):
    // Step 1: Split ciphertext into tweak and encrypted IP
    tweak = ciphertext[0:8]  // First 8 bytes
    encrypted_ip = ciphertext[8:24]  // Remaining 16 bytes

    // Step 2: Decrypt using key and tweak
    bytes16 = KIASU_BC_decrypt(key, tweak, encrypted_ip)

    // Step 3: Convert back to IP address
    ip_address = Bytes16ToIP(bytes16)
    return ip_address
~~~

## Non‑Deterministic Encryption using AES‑XTS (ipcrypt-ndx)

~~~pseudocode
function ipcrypt_ndx_encrypt(ip_address, key):
    // The key MUST be exactly 32 bytes (256 bits) in length, consisting of two 16-byte AES-128 keys
    if length(key) != 32:
        raise Error("Key must be 32 bytes (two AES-128 keys)")

    // Step 1: Generate random tweak (MUST be exactly 16 bytes)
    tweak = random_bytes(16)  // MUST be uniformly random

    // Step 2: Convert IP to 16-byte representation
    bytes16 = convertTo16Bytes(ip_address)

    // Step 3: Encrypt using key and tweak
    // Since only a single block is encrypted, only the first tweak needs to be computed
    ciphertext = AES_XTS_encrypt(key, tweak, bytes16)

    // Step 4: Concatenate tweak and ciphertext
    result = concatenate(tweak, ciphertext)  // 16 bytes || 16 bytes = 32 bytes total
    return result

function ipcrypt_ndx_decrypt(ciphertext, key):
    // Step 1: Split ciphertext into tweak and encrypted IP
    tweak = ciphertext[0:16]  // First 16 bytes
    encrypted_ip = ciphertext[16:32]  // Remaining 16 bytes

    // Step 2: Decrypt using key and tweak
    bytes16 = AES_XTS_decrypt(key, tweak, encrypted_ip)

    // Step 3: Convert back to IP address
    ip_address = Bytes16ToIP(bytes16)
    return ip_address
~~~

# Implementing KIASU-BC {#implementing-kiasu-bc}

This appendix provides a detailed guide for implementing the KIASU-BC tweakable block cipher. KIASU-BC is based on AES-128 with modifications to incorporate a tweak. For more information about the security properties of KIASU-BC, see {{KIASU-BC}}.

## Overview

KIASU-BC extends AES-128 by incorporating an 8-byte tweak into each round. The tweak is padded to 16 bytes and XORed with the round key at each round of the cipher. This construction is used in the `ipcrypt-nd` instantiation.

## Tweak Padding

The 8-byte tweak is padded to 16 bytes using the following method:

1. Split the 8-byte tweak into four 2-byte pairs
2. Place each 2-byte pair at the start of each 4-byte group
3. Fill the remaining 2 bytes of each group with zeros

Example:

~~~
8-byte tweak:    [T0 T1 T2 T3 T4 T5 T6 T7]
16-byte padded:  [T0 T1 00 00 T2 T3 00 00 T4 T5 00 00 T6 T7 00 00]
~~~

## Round Structure

Each round of KIASU-BC consists of the following standard AES operations:

1. **SubBytes:** Apply the AES S-box to each byte of the state
2. **ShiftRows:** Rotate each row of the state matrix
3. **MixColumns:** Mix the columns of the state matrix (except in the final round)
4. **AddRoundKey:** XOR the state with the round key and padded tweak

For details about these operations, see {{FIPS-197}}.

## Key Schedule

The key schedule follows the standard AES-128 key expansion:

1. The initial key is expanded into 11 round keys
2. Each round key is XORed with the padded tweak before use
3. The first round key is used in the initial AddRoundKey operation

## Implementation Steps

1. **Key Expansion:**
   - Expand the 16-byte key into 11 round keys using the standard AES key schedule
   - Each round key is 16 bytes

2. **Tweak Processing:**
   - Pad the 8-byte tweak to 16 bytes as described above
   - XOR the padded tweak with each round key before use

3. **Encryption Process:**
   - Perform initial AddRoundKey with the first tweaked round key
   - For rounds 1-9:
     - SubBytes
     - ShiftRows
     - MixColumns
     - AddRoundKey (with tweaked round key)
   - For round 10 (final round):
     - SubBytes
     - ShiftRows
     - AddRoundKey (with tweaked round key)

## Example Implementation

The following pseudocode illustrates the core operations of KIASU-BC:

~~~pseudocode
function pad_tweak(tweak):
    // Input: 8-byte tweak
    // Output: 16-byte padded tweak
    padded = [0] * 16
    for i in range(0, 8, 2):
        padded[i*2] = tweak[i]
        padded[i*2+1] = tweak[i+1]
    return padded

function kiasu_bc_encrypt(key, tweak, plaintext):
    // Input: 16-byte key, 8-byte tweak, 16-byte plaintext
    // Output: 16-byte ciphertext

    // Expand key and pad tweak
    round_keys = expand_key(key)
    padded_tweak = pad_tweak(tweak)

    // Initial round
    state = plaintext
    state = add_round_key(state, round_keys[0] ^ padded_tweak)

    // Main rounds
    for round in range(1, 10):
        state = sub_bytes(state)
        state = shift_rows(state)
        state = mix_columns(state)
        state = add_round_key(state, round_keys[round] ^ padded_tweak)

    // Final round
    state = sub_bytes(state)
    state = shift_rows(state)
    state = add_round_key(state, round_keys[10] ^ padded_tweak)

    return state
~~~

# Test Vectors {#test-vectors}

This appendix provides test vectors for all three variants of ipcrypt. Each test vector includes the key, input IP address, and encrypted output. For non-deterministic variants (`ipcrypt-nd` and `ipcrypt-ndx`), the tweak value is also included.

Note: The key and tweak sizes are:
- `ipcrypt-deterministic`:
  - Key: 16 bytes (128 bits)
  - No tweak used
- `ipcrypt-nd`:
  - Key: 16 bytes (128 bits)
  - Tweak: 8 bytes (64 bits)
- `ipcrypt-ndx`:
  - Key: 32 bytes (256 bits, two AES-128 keys)
  - Tweak: 16 bytes (128 bits)

## ipcrypt-deterministic Test Vectors {#ipcrypt-deterministic-test-vectors}

~~~
# Test vector 1
Key:          0123456789abcdeffedcba9876543210
Input IP:     0.0.0.0
Encrypted IP: bde9:6789:d353:824c:d7c6:f58a:6bd2:26eb

# Test vector 2
Key:          1032547698badcfeefcdab8967452301
Input IP:     255.255.255.255
Encrypted IP: aed2:92f6:ea23:58c3:48fd:8b8:74e8:45d8

# Test vector 3
Key:          2b7e151628aed2a6abf7158809cf4f3c
Input IP:     192.0.2.1
Encrypted IP: 1dbd:c1b9:fff1:7586:7d0b:67b4:e76e:4777
~~~

## ipcrypt-nd Test Vectors {#ipcrypt-nd-test-vectors}

~~~
# Test vector 1
Key:          0123456789abcdeffedcba9876543210
Input IP:     0.0.0.0
Tweak:        08e0c289bff23b7c
Output:       08e0c289bff23b7cb349aadfe3bcef56221c384c7c217b16

# Test vector 2
Key:          1032547698badcfeefcdab8967452301
Input IP:     192.0.2.1
Tweak:        21bd1834bc088cd2
Output:       21bd1834bc088cd2e5e1fe55f95876e639faae2594a0caad

# Test vector 3
Key:          2b7e151628aed2a6abf7158809cf4f3c
Input IP:     2001:db8::1
Tweak:        b4ecbe30b70898d7
Output:       b4ecbe30b70898d7553ac8974d1b4250eafc4b0aa1f80c96
~~~

## ipcrypt-ndx Test Vectors {#ipcrypt-ndx-test-vectors}

~~~
# Test vector 1
Key:          0123456789abcdeffedcba98765432101032547698badcfeefcdab8967452301
Input IP:     0.0.0.0
Tweak:        21bd1834bc088cd2b4ecbe30b70898d7
Output:       21bd1834bc088cd2b4ecbe30b70898d782db0d4125fdace61db35b8339f20ee5

# Test vector 2
Key:          1032547698badcfeefcdab89674523010123456789abcdeffedcba9876543210
Input IP:     192.0.2.1
Tweak:        08e0c289bff23b7cb4ecbe30b70898d7
Output:       08e0c289bff23b7cb4ecbe30b70898d7766a533392a69edf1ad0d3ce362ba98a

# Test vector 3
Key:          2b7e151628aed2a6abf7158809cf4f3c3c4fcf098815f7aba6d2ae2816157e2b
Input IP:     2001:db8::1
Tweak:        21bd1834bc088cd2b4ecbe30b70898d7
Output:       21bd1834bc088cd2b4ecbe30b70898d76089c7e05ae30c2d10ca149870a263e4
~~~

Note: For non-deterministic variants (`ipcrypt-nd` and `ipcrypt-ndx`), the tweak values shown are examples. In practice, tweaks MUST be randomly generated for each encryption operation.

Implementations SHOULD verify their correctness against these test vectors before deployment.

# Acknowledgments

The author gratefully acknowledges the contributions and insightful comments from members of the IETF independent stream community and the broader cryptographic community that have helped shape this specification.
