# CLAUDE.md - secp256k1-embedded-ecdh

This file provides guidance for working with the secp256k1 elliptic curve library for embedded systems.

## Project Overview

**secp256k1-embedded-ecdh** is a port of Bitcoin Core's [secp256k1](https://github.com/bitcoin-core/secp256k1/) elliptic curve cryptography library for use on embedded systems, specifically:
- **Arduino IDE** (ESP32, etc.)
- **ARM Mbed** (STM32, etc.)
- **MicroPython** (ESP32, STM32, RP2, etc.)

This library provides optimized cryptographic primitives for:
- **ECDSA** (Elliptic Curve Digital Signature Algorithm) - signing and verification
- **ECDH** (Elliptic Curve Diffie-Hellman) - key agreement/derivation
- **Schnorr signatures** - newer Bitcoin signature scheme
- **Public/private key operations** - key generation, serialization, validation
- **Key recovery** - recover public key from signature

The library is highly optimized, constant-time (resistant to timing attacks), and battle-tested by Bitcoin Core.

## Why This Repository Exists

The upstream Bitcoin Core secp256k1 library is designed for desktop/server use with autoconf-based builds. This repository adds:

1. **Build system compatibility** - Works with Arduino, Mbed, and MicroPython build systems
2. **MicroPython bindings** - Python API wrapping the C library
3. **Memory management** - Uses preallocated context to avoid malloc/free in embedded environments
4. **Portability** - Hacks to work around recursive build systems (Arduino, Mbed)
5. **Configuration** - Pre-configured for embedded constraints (limited stack, no stdlib)

## Repository Structure

### Core Directories

- **`secp256k1/`**: Git submodule - upstream Bitcoin Core secp256k1 library
  - `src/`: Core C implementation
  - `include/`: Public API headers
  - This is the **actual cryptographic code** (never modify directly)

- **`src/`**: Embedded wrapper layer
  - `secp256k1_bundle.c`: Single-file compilation unit (includes all secp256k1 sources)
  - `secp256k1.h`, `secp256k1_ecdh.h`, etc.: Header wrappers
  - `ecmult_static_context.h`: Precomputed multiplication tables (217KB!)
  - `libsecp256k1-config.h`: Configuration for embedded builds

- **`mpy/`**: MicroPython bindings
  - `libsecp256k1.c`: Python↔C binding implementation (1300+ lines)
  - `config/ext_callbacks.c`: Custom memory allocation callbacks for MicroPython
  - This is what makes secp256k1 accessible from Python

- **`examples/`**: Usage examples
  - `secp256k1.py`: MicroPython usage example
  - `basic_example/basic_example.ino`: Arduino example
  - `mbed/main.cpp`: ARM Mbed example

### Build System Files

- **`micropython.mk`**: Makefile for MicroPython builds (legacy build system)
  - Defines which C files to compile
  - Sets compiler flags: `-DHAVE_CONFIG_H -Wno-unused-function -O2`
  - Adds include paths to secp256k1 headers

- **`micropython.cmake`**: CMake for MicroPython builds (modern build system)
  - Used by newer MicroPython ports
  - Equivalent to micropython.mk but for CMake

- **`library.properties`**: Arduino library metadata
  - Name, version, author, category
  - Used by Arduino IDE to identify the library

- **`.mbedignore`**: Files to ignore in Mbed builds

## MicroPython Integration

### Building with MicroPython

The library is compiled as a **user C module** (also called external module or cmodule).

**Clone with submodules**:
```bash
git clone --recursive https://github.com/diybitcoinhardware/secp256k1-embedded-ecdh
# Or if already cloned:
git submodule update --init --recursive
```

**Build MicroPython with module**:
```bash
cd micropython/ports/esp32
make BOARD=GENERIC_S3 \
     USER_C_MODULES=/path/to/secp256k1-embedded-ecdh/micropython.cmake \
     CFLAGS_EXTRA=-DMODULE_SECP256K1_ENABLED=1
```

**Key flags**:
- `USER_C_MODULES`: Path to micropython.mk or micropython.cmake
- `CFLAGS_EXTRA=-DMODULE_SECP256K1_ENABLED=1`: Enable the module
- Use `.cmake` for newer ports, `.mk` for older

### How MicroPythonOS Uses This

MicroPythonOS integrates secp256k1 for Bitcoin/Lightning/Nostr cryptography:

1. **Build integration**: `scripts/build_mpos.sh` symlinks this directory as a user module
2. **Python wrapper**: `internal_filesystem/lib/secp256k1.py` provides high-level API
3. **Compatibility layer**: `secp256k1_compat.py` adapts the C module for desktop/embedded
4. **Used by**: Nostr (key management), Lightning (payment signatures), Bitcoin wallets

## Python API

The module exposes a `secp256k1` Python module with these functions:

### Context Management

```python
import secp256k1

# Randomize context for sidechannel attack resistance
secp256k1.context_randomize(os.urandom(32))  # 32 random bytes
```

**Note**: The context is **preallocated** (880 bytes on 64-bit, 440 on 32-bit) and created on first use.

### Key Operations

```python
# Verify secret key is valid
is_valid = secp256k1.ec_seckey_verify(secret_key)  # secret_key: 32 bytes

# Create public key from secret key
pubkey = secp256k1.ec_pubkey_create(secret_key)  # Returns 64 bytes (internal format)

# Serialize public key
sec_compressed = secp256k1.ec_pubkey_serialize(pubkey, secp256k1.EC_COMPRESSED)  # 33 bytes
sec_uncompressed = secp256k1.ec_pubkey_serialize(pubkey, secp256k1.EC_UNCOMPRESSED)  # 65 bytes

# Parse serialized public key
pubkey = secp256k1.ec_pubkey_parse(sec_compressed)  # Returns 64 bytes (internal format)
```

**Key formats**:
- **Internal format**: 64 bytes (two 32-byte coordinates, not serialized)
- **Compressed SEC**: 33 bytes (0x02/0x03 prefix + x-coordinate)
- **Uncompressed SEC**: 65 bytes (0x04 prefix + x-coordinate + y-coordinate)

### ECDSA Signing and Verification

```python
import hashlib

# Sign a message hash (must be exactly 32 bytes)
msg_hash = hashlib.sha256(b"hello").digest()
signature = secp256k1.ecdsa_sign(msg_hash, secret_key)  # Returns 64 bytes (internal format)

# Serialize signature to DER format (Bitcoin standard)
der_sig = secp256k1.ecdsa_signature_serialize_der(signature)  # Variable length (70-72 bytes)

# Serialize signature to compact format
compact_sig = secp256k1.ecdsa_signature_serialize_compact(signature)  # 64 bytes

# Parse signature
signature = secp256k1.ecdsa_signature_parse_der(der_sig)
signature = secp256k1.ecdsa_signature_parse_compact(compact_sig)

# Verify signature
is_valid = secp256k1.ecdsa_verify(signature, msg_hash, pubkey)  # True/False
```

### ECDH (Key Agreement)

```python
# Derive shared secret from your private key and their public key
shared_secret = secp256k1.ecdh(their_pubkey, my_secret_key)  # Returns 32 bytes
```

**Use case**: Two parties can derive the same shared secret without exchanging it.

### Schnorr Signatures (BIP340)

```python
# Create keypair for Schnorr (requires x-only pubkey)
keypair = secp256k1.keypair_create(secret_key)

# Sign with Schnorr
schnorr_sig = secp256k1.schnorrsig_sign(msg_hash, keypair, aux_rand=None)  # 64 bytes

# Verify Schnorr signature
xonly_pubkey = secp256k1.xonly_pubkey_from_pubkey(pubkey)
is_valid = secp256k1.schnorrsig_verify(schnorr_sig, msg_hash, xonly_pubkey)
```

### Recoverable Signatures (Ethereum style)

```python
# Sign with recovery info (allows recovering pubkey from signature)
recoverable_sig = secp256k1.ecdsa_sign_recoverable(msg_hash, secret_key)

# Serialize with recovery ID
compact_sig, recovery_id = secp256k1.ecdsa_recoverable_signature_serialize_compact(recoverable_sig)

# Recover public key from signature
recovered_pubkey = secp256k1.ecdsa_recover(recoverable_sig, msg_hash)
```

**Use case**: Ethereum uses this to derive sender address from transaction signature.

## C API (Low-Level)

The binding wraps these core secp256k1 C functions:

**Core types**:
```c
secp256k1_context      // Precomputed tables for operations
secp256k1_pubkey       // Public key (64 bytes internal format)
secp256k1_ecdsa_signature  // ECDSA signature (64 bytes internal format)
```

**Key operations**:
```c
secp256k1_ec_seckey_verify()        // Validate private key
secp256k1_ec_pubkey_create()        // Derive public key
secp256k1_ec_pubkey_parse()         // Parse SEC-encoded pubkey
secp256k1_ec_pubkey_serialize()     // Serialize to SEC format
```

**Signing/verification**:
```c
secp256k1_ecdsa_sign()              // Create ECDSA signature
secp256k1_ecdsa_verify()            // Verify ECDSA signature
secp256k1_ecdsa_signature_parse_der()     // Parse DER signature
secp256k1_ecdsa_signature_serialize_der() // Serialize to DER
```

## Memory Management

### Preallocated Context

The binding uses a **preallocated context** to avoid dynamic allocation:

```c
// From mpy/libsecp256k1.c
#define PREALLOCATED_CTX_SIZE 880  // 440 for 32-bit
static unsigned char preallocated_ctx[PREALLOCATED_CTX_SIZE];
static secp256k1_context *ctx = NULL;

void maybe_init_ctx() {
    if (ctx != NULL) return;
    ctx = secp256k1_context_preallocated_create(
        (void *)preallocated_ctx,
        SECP256K1_CONTEXT_VERIFY | SECP256K1_CONTEXT_SIGN
    );
}
```

**Why**: Embedded systems have limited heap; preallocating avoids fragmentation.

### Custom Allocators

The binding redirects malloc/free to MicroPython's garbage collector:

```c
#define malloc(b) gc_alloc((b), false)
#define free gc_free
```

This ensures all memory is managed by MicroPython's GC, preventing leaks.

## Configuration

### Compile-Time Options (src/libsecp256k1-config.h)

Key settings for embedded builds:

```c
#define USE_NUM_NONE 1              // No bignum library (use secp256k1's)
#define USE_FIELD_INV_BUILTIN 1     // Use built-in field inversion
#define USE_SCALAR_INV_BUILTIN 1    // Use built-in scalar inversion
#define ECMULT_WINDOW_SIZE 15       // Precomputed table size (tradeoff: speed vs memory)
#define ECMULT_GEN_PREC_BITS 4      // Generator multiplication precision
#define ENABLE_MODULE_RECOVERY 1    // Enable signature recovery
#define ENABLE_MODULE_ECDH 1        // Enable ECDH
#define ENABLE_MODULE_SCHNORRSIG 1  // Enable Schnorr signatures
#define ENABLE_MODULE_EXTRAKEYS 1   // Enable x-only pubkeys
```

### Precomputed Tables

The file `src/ecmult_static_context.h` (217KB!) contains **precomputed multiplication tables**.

**Why so big?**: These tables speed up elliptic curve point multiplication by 100-1000x. The tradeoff is binary size for speed.

**Generated by**: `secp256k1/src/gen_context.c` (run during upstream build)

## Important Constraints

### Stack Usage

⚠️ **secp256k1 uses significant stack space** (several KB per operation)

**For ARM Mbed**: Increase stack size in `mbed_app.json`:
```json
{
    "target_overrides": {
        "*": {
            "rtos.main-thread-stack-size": "8192"
        }
    }
}
```

**For MicroPython**: Ensure thread stack size is adequate (ESP32 default 16KB is usually sufficient)

### Message Hash Requirements

⚠️ **ECDSA/Schnorr require exactly 32-byte hashes**

```python
# WRONG - will raise ValueError
secp256k1.ecdsa_sign(b"hello", secret_key)

# CORRECT - hash first
msg_hash = hashlib.sha256(b"hello").digest()
secp256k1.ecdsa_sign(msg_hash, secret_key)
```

### Binary Size

The library adds ~150-200KB to your binary:
- secp256k1 core: ~80KB
- Precomputed tables: ~217KB (in ecmult_static_context.h)
- Bindings: ~20KB

**Optimization**: Use `-O2` flag (already set in micropython.mk)

## Security Considerations

### Constant-Time Operations

All cryptographic operations are **constant-time** (execution time doesn't depend on secret values).

**Why**: Prevents timing sidechannel attacks that could leak private keys.

### Context Randomization

```python
# Randomize context periodically (every few operations)
secp256k1.context_randomize(os.urandom(32))
```

**Why**: Adds additional protection against sidechannel attacks by randomizing internal state.

### Private Key Generation

⚠️ **Never use weak randomness for private keys**

```python
# WRONG - deterministic, weak
secret_key = hashlib.sha256(b"my password").digest()

# BETTER - use hardware RNG
import os
secret_key = os.urandom(32)

# VERIFY - always check key is valid
if not secp256k1.ec_seckey_verify(secret_key):
    raise ValueError("Invalid key - regenerate")
```

## Testing

### Unit Tests

The upstream secp256k1 library has extensive tests:
```bash
cd secp256k1
./autogen.sh
./configure
make check
```

### MicroPython Example

Run the example on device:
```bash
mpremote run examples/secp256k1.py
```

Or on desktop:
```bash
python3 examples/secp256k1.py  # Requires desktop secp256k1 library
```

## Common Issues

### "Failed to randomize context"

**Cause**: Called `context_randomize()` with non-32-byte seed
**Fix**: Use exactly 32 bytes: `secp256k1.context_randomize(os.urandom(32))`

### "Private key should be 32 bytes long"

**Cause**: Secret key is wrong length
**Fix**: Ensure secret key is exactly 32 bytes

### "Pubkey should be 64 bytes long"

**Cause**: Passed serialized pubkey instead of internal format
**Fix**: Use `secp256k1.ec_pubkey_parse()` first to convert SEC format → internal format

### Stack overflow on embedded devices

**Cause**: Insufficient stack size
**Fix**: Increase thread/task stack size (see "Stack Usage" above)

### Build fails with "secp256k1.h not found"

**Cause**: Submodule not initialized
**Fix**: `git submodule update --init --recursive`

## Performance

Typical operation speeds on ESP32 @ 240MHz:

- **Key generation**: ~10ms
- **ECDSA sign**: ~15ms
- **ECDSA verify**: ~25ms
- **ECDH**: ~20ms
- **Schnorr sign**: ~15ms
- **Schnorr verify**: ~20ms

**Note**: First operation is slower (context initialization)

## Use Cases in MicroPythonOS

1. **Nostr protocol** (`micropython-nostr/`):
   - Event signing (Schnorr)
   - Key generation
   - Event verification

2. **Bitcoin/Lightning** (future apps):
   - Transaction signing
   - Payment channel operations
   - PSBT (Partially Signed Bitcoin Transactions)

3. **Authentication**:
   - Challenge-response protocols
   - Secure device pairing (ECDH)

## Upstream References

- **Bitcoin Core secp256k1**: https://github.com/bitcoin-core/secp256k1
- **This repository**: https://github.com/diybitcoinhardware/secp256k1-embedded-ecdh
- **Original MicroPython binding**: Part of DIY Bitcoin Hardware project

## Important Notes

- **Never modify** `secp256k1/` submodule - it's upstream Bitcoin Core code
- **Test thoroughly** - cryptographic code must be bug-free
- **Use hardware RNG** - never use weak randomness for keys
- **Verify keys** - always check `ec_seckey_verify()` after generation
- **Hash messages** - ECDSA/Schnorr require 32-byte hashes, not raw messages
- **Constant-time** - library is designed to resist timing attacks
- **Stack usage** - be aware of stack requirements on embedded devices

## Getting Help

- **Examples**: See `examples/` directory for usage patterns
- **API reference**: See `secp256k1/include/*.h` for C API docs
- **MicroPython binding**: See `mpy/libsecp256k1.c` for Python API implementation
- **Issues**: Report bugs to https://github.com/diybitcoinhardware/secp256k1-embedded-ecdh/issues
