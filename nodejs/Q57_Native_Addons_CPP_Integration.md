# Q57: Native Addons - C++ Integration

## 📋 Summary
This question covers **Node.js native addons** with C++ integration using N-API. Learn when and how to write high-performance C++ modules for CPU-intensive operations in banking applications - from cryptography to numerical computations.

**Key Topics**:
- When to use native addons
- N-API (Node-API) fundamentals
- C++ addon development
- node-gyp build system
- Memory management between JS and C++
- Threading and async work
- Debugging native addons
- Performance optimization
- Security considerations
- Cross-platform compilation

**Banking Use Cases**:
- High-performance encryption/decryption
- Complex financial calculations
- Real-time fraud detection algorithms
- High-frequency data processing
- Secure random number generation
- Hardware security module (HSM) integration
- Image processing for check deposits
- OCR for document verification

---

## 🎯 Understanding Native Addons

### What are Native Addons?

**Native addons** are dynamically-linked shared objects written in C++ that can be loaded into Node.js, providing direct access to native APIs and libraries.

```
┌─────────────────────────────────────────────────────────────┐
│            Node.js Native Addon Architecture                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  JavaScript Layer                                           │
│  ┌────────────────────────────────────────┐                │
│  │  const addon = require('./build/       │                │
│  │    Release/crypto_addon.node');        │                │
│  │                                        │                │
│  │  const encrypted = addon.encrypt(      │                │
│  │    data, key                           │                │
│  │  );                                    │                │
│  └────────────┬───────────────────────────┘                │
│               │ N-API Bridge                                │
│               ▼                                             │
│  ┌────────────────────────────────────────┐                │
│  │  N-API (Node-API)                      │                │
│  │  - ABI-stable interface                │                │
│  │  - Version-independent                 │                │
│  │  - Type conversions                    │                │
│  └────────────┬───────────────────────────┘                │
│               │                                             │
│               ▼                                             │
│  ┌────────────────────────────────────────┐                │
│  │  C++ Addon Code                        │                │
│  │  - OpenSSL integration                 │                │
│  │  - Native libraries                    │                │
│  │  - Hardware access                     │                │
│  └────────────┬───────────────────────────┘                │
│               │                                             │
│               ▼                                             │
│  ┌────────────────────────────────────────┐                │
│  │  Native Libraries                      │                │
│  │  - OpenSSL                             │                │
│  │  - Math libraries                      │                │
│  │  - Hardware drivers                    │                │
│  └────────────────────────────────────────┘                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### When to Use Native Addons

| Use Case | JavaScript | Native Addon |
|----------|-----------|--------------|
| **CPU-intensive computation** | Slow | ✅ Fast |
| **Cryptographic operations** | Limited | ✅ Full control |
| **Hardware interaction** | Not possible | ✅ Direct access |
| **Existing C/C++ libraries** | Need ports | ✅ Direct use |
| **Tight loops** | V8 optimizes | ✅ Always fast |
| **Simple CRUD operations** | ✅ Perfect fit | Overkill |
| **I/O-bound tasks** | ✅ Async perfect | No benefit |

### N-API vs Other Bindings

| Feature | N-API | NAN | V8 Direct |
|---------|-------|-----|-----------|
| **ABI Stability** | ✅ Yes | ❌ No | ❌ No |
| **Node Version** | Works across versions | Needs rebuild | Needs rebuild |
| **Learning Curve** | Medium | Medium | High |
| **Performance** | Excellent | Excellent | Best |
| **Recommended** | ✅ Yes | Legacy | Advanced only |

---

## 💡 Example 1: High-Performance Crypto Addon

Complete N-API addon for banking encryption operations.

### Project Structure

```
crypto-addon/
├── binding.gyp           # Build configuration
├── package.json
├── src/
│   ├── crypto_addon.cc   # C++ implementation
│   └── crypto_addon.h    # Header file
├── lib/
│   └── index.js          # JavaScript wrapper
├── test/
│   └── crypto.test.js
└── README.md
```

### 1. Build Configuration

```python
# binding.gyp
{
  "targets": [
    {
      "target_name": "crypto_addon",
      "sources": [
        "src/crypto_addon.cc"
      ],
      "include_dirs": [
        "<!@(node -p \"require('node-addon-api').include\")"
      ],
      "dependencies": [
        "<!(node -p \"require('node-addon-api').gyp\")"
      ],
      "defines": [
        "NAPI_DISABLE_CPP_EXCEPTIONS"
      ],
      "cflags!": [ "-fno-exceptions" ],
      "cflags_cc!": [ "-fno-exceptions" ],
      "xcode_settings": {
        "GCC_ENABLE_CPP_EXCEPTIONS": "YES",
        "CLANG_CXX_LIBRARY": "libc++",
        "MACOSX_DEPLOYMENT_TARGET": "10.15"
      },
      "msvs_settings": {
        "VCCLCompilerTool": {
          "ExceptionHandling": 1
        }
      },
      "conditions": [
        ["OS=='linux'", {
          "libraries": [
            "-lcrypto",
            "-lssl"
          ]
        }],
        ["OS=='mac'", {
          "libraries": [
            "-lcrypto",
            "-lssl"
          ]
        }],
        ["OS=='win'", {
          "libraries": [
            "libcrypto.lib",
            "libssl.lib"
          ]
        }]
      ]
    }
  ]
}
```

### 2. C++ Header File

```cpp
// src/crypto_addon.h

#ifndef CRYPTO_ADDON_H
#define CRYPTO_ADDON_H

#include <napi.h>
#include <openssl/evp.h>
#include <openssl/aes.h>
#include <openssl/rand.h>
#include <openssl/err.h>
#include <string>
#include <vector>

namespace crypto_addon {

/**
 * Encrypt data using AES-256-GCM
 */
Napi::Value Encrypt(const Napi::CallbackInfo& info);

/**
 * Decrypt data using AES-256-GCM
 */
Napi::Value Decrypt(const Napi::CallbackInfo& info);

/**
 * Generate secure random bytes
 */
Napi::Value GenerateRandomBytes(const Napi::CallbackInfo& info);

/**
 * Hash data using SHA-256
 */
Napi::Value HashSHA256(const Napi::CallbackInfo& info);

/**
 * Verify HMAC
 */
Napi::Value VerifyHMAC(const Napi::CallbackInfo& info);

/**
 * Initialize the addon
 */
Napi::Object Init(Napi::Env env, Napi::Object exports);

} // namespace crypto_addon

#endif // CRYPTO_ADDON_H
```

### 3. C++ Implementation

```cpp
// src/crypto_addon.cc

#include "crypto_addon.h"
#include <cstring>
#include <iostream>

namespace crypto_addon {

/**
 * Encrypt data using AES-256-GCM
 * 
 * Parameters:
 *   - plaintext (Buffer): Data to encrypt
 *   - key (Buffer): 256-bit encryption key
 *   - iv (Buffer): Initialization vector (12 bytes for GCM)
 * 
 * Returns:
 *   Object: { ciphertext: Buffer, tag: Buffer }
 */
Napi::Value Encrypt(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();

  // Validate arguments
  if (info.Length() < 3) {
    Napi::TypeError::New(env, "Expected 3 arguments: plaintext, key, iv")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  if (!info[0].IsBuffer() || !info[1].IsBuffer() || !info[2].IsBuffer()) {
    Napi::TypeError::New(env, "All arguments must be Buffers")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Extract buffers
  Napi::Buffer<unsigned char> plaintext = info[0].As<Napi::Buffer<unsigned char>>();
  Napi::Buffer<unsigned char> key = info[1].As<Napi::Buffer<unsigned char>>();
  Napi::Buffer<unsigned char> iv = info[2].As<Napi::Buffer<unsigned char>>();

  // Validate key size (32 bytes for AES-256)
  if (key.Length() != 32) {
    Napi::TypeError::New(env, "Key must be 32 bytes for AES-256")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Validate IV size (12 bytes recommended for GCM)
  if (iv.Length() != 12) {
    Napi::TypeError::New(env, "IV must be 12 bytes for GCM")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Initialize cipher context
  EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
  if (!ctx) {
    Napi::Error::New(env, "Failed to create cipher context")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Initialize encryption operation
  if (EVP_EncryptInit_ex(ctx, EVP_aes_256_gcm(), NULL, 
                         key.Data(), iv.Data()) != 1) {
    EVP_CIPHER_CTX_free(ctx);
    Napi::Error::New(env, "Failed to initialize encryption")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Allocate output buffer (same size as input + block size)
  std::vector<unsigned char> ciphertext(plaintext.Length() + EVP_CIPHER_block_size(EVP_aes_256_gcm()));
  int len;
  int ciphertext_len = 0;

  // Encrypt data
  if (EVP_EncryptUpdate(ctx, ciphertext.data(), &len, 
                        plaintext.Data(), plaintext.Length()) != 1) {
    EVP_CIPHER_CTX_free(ctx);
    Napi::Error::New(env, "Encryption failed")
        .ThrowAsJavaScriptException();
    return env.Null();
  }
  ciphertext_len = len;

  // Finalize encryption
  if (EVP_EncryptFinal_ex(ctx, ciphertext.data() + len, &len) != 1) {
    EVP_CIPHER_CTX_free(ctx);
    Napi::Error::New(env, "Encryption finalization failed")
        .ThrowAsJavaScriptException();
    return env.Null();
  }
  ciphertext_len += len;

  // Get authentication tag (16 bytes)
  unsigned char tag[16];
  if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_GET_TAG, 16, tag) != 1) {
    EVP_CIPHER_CTX_free(ctx);
    Napi::Error::New(env, "Failed to get authentication tag")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  EVP_CIPHER_CTX_free(ctx);

  // Create result object
  Napi::Object result = Napi::Object::New(env);
  
  // Copy ciphertext to Node.js Buffer
  Napi::Buffer<unsigned char> ciphertextBuffer = 
      Napi::Buffer<unsigned char>::Copy(env, ciphertext.data(), ciphertext_len);
  
  // Copy tag to Node.js Buffer
  Napi::Buffer<unsigned char> tagBuffer = 
      Napi::Buffer<unsigned char>::Copy(env, tag, 16);

  result.Set("ciphertext", ciphertextBuffer);
  result.Set("tag", tagBuffer);

  return result;
}

/**
 * Decrypt data using AES-256-GCM
 * 
 * Parameters:
 *   - ciphertext (Buffer): Encrypted data
 *   - key (Buffer): 256-bit encryption key
 *   - iv (Buffer): Initialization vector (12 bytes)
 *   - tag (Buffer): Authentication tag (16 bytes)
 * 
 * Returns:
 *   Buffer: Decrypted plaintext
 */
Napi::Value Decrypt(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();

  // Validate arguments
  if (info.Length() < 4) {
    Napi::TypeError::New(env, "Expected 4 arguments: ciphertext, key, iv, tag")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  if (!info[0].IsBuffer() || !info[1].IsBuffer() || 
      !info[2].IsBuffer() || !info[3].IsBuffer()) {
    Napi::TypeError::New(env, "All arguments must be Buffers")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Extract buffers
  Napi::Buffer<unsigned char> ciphertext = info[0].As<Napi::Buffer<unsigned char>>();
  Napi::Buffer<unsigned char> key = info[1].As<Napi::Buffer<unsigned char>>();
  Napi::Buffer<unsigned char> iv = info[2].As<Napi::Buffer<unsigned char>>();
  Napi::Buffer<unsigned char> tag = info[3].As<Napi::Buffer<unsigned char>>();

  // Validate sizes
  if (key.Length() != 32) {
    Napi::TypeError::New(env, "Key must be 32 bytes").ThrowAsJavaScriptException();
    return env.Null();
  }
  if (iv.Length() != 12) {
    Napi::TypeError::New(env, "IV must be 12 bytes").ThrowAsJavaScriptException();
    return env.Null();
  }
  if (tag.Length() != 16) {
    Napi::TypeError::New(env, "Tag must be 16 bytes").ThrowAsJavaScriptException();
    return env.Null();
  }

  // Initialize cipher context
  EVP_CIPHER_CTX* ctx = EVP_CIPHER_CTX_new();
  if (!ctx) {
    Napi::Error::New(env, "Failed to create cipher context")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Initialize decryption
  if (EVP_DecryptInit_ex(ctx, EVP_aes_256_gcm(), NULL, 
                         key.Data(), iv.Data()) != 1) {
    EVP_CIPHER_CTX_free(ctx);
    Napi::Error::New(env, "Failed to initialize decryption")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Allocate output buffer
  std::vector<unsigned char> plaintext(ciphertext.Length() + EVP_CIPHER_block_size(EVP_aes_256_gcm()));
  int len;
  int plaintext_len = 0;

  // Decrypt data
  if (EVP_DecryptUpdate(ctx, plaintext.data(), &len, 
                        ciphertext.Data(), ciphertext.Length()) != 1) {
    EVP_CIPHER_CTX_free(ctx);
    Napi::Error::New(env, "Decryption failed")
        .ThrowAsJavaScriptException();
    return env.Null();
  }
  plaintext_len = len;

  // Set expected tag
  if (EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_SET_TAG, 16, 
                          const_cast<unsigned char*>(tag.Data())) != 1) {
    EVP_CIPHER_CTX_free(ctx);
    Napi::Error::New(env, "Failed to set authentication tag")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Finalize decryption (verifies tag)
  if (EVP_DecryptFinal_ex(ctx, plaintext.data() + len, &len) != 1) {
    EVP_CIPHER_CTX_free(ctx);
    Napi::Error::New(env, "Decryption failed: authentication tag mismatch")
        .ThrowAsJavaScriptException();
    return env.Null();
  }
  plaintext_len += len;

  EVP_CIPHER_CTX_free(ctx);

  // Return plaintext as Buffer
  return Napi::Buffer<unsigned char>::Copy(env, plaintext.data(), plaintext_len);
}

/**
 * Generate cryptographically secure random bytes
 */
Napi::Value GenerateRandomBytes(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();

  if (info.Length() < 1 || !info[0].IsNumber()) {
    Napi::TypeError::New(env, "Expected number of bytes as argument")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  int32_t numBytes = info[0].As<Napi::Number>().Int32Value();

  if (numBytes <= 0 || numBytes > 65536) {
    Napi::RangeError::New(env, "Number of bytes must be between 1 and 65536")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  std::vector<unsigned char> randomBytes(numBytes);

  // Generate random bytes using OpenSSL
  if (RAND_bytes(randomBytes.data(), numBytes) != 1) {
    Napi::Error::New(env, "Failed to generate random bytes")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  return Napi::Buffer<unsigned char>::Copy(env, randomBytes.data(), numBytes);
}

/**
 * Hash data using SHA-256
 */
Napi::Value HashSHA256(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();

  if (info.Length() < 1 || !info[0].IsBuffer()) {
    Napi::TypeError::New(env, "Expected Buffer as argument")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  Napi::Buffer<unsigned char> data = info[0].As<Napi::Buffer<unsigned char>>();

  unsigned char hash[EVP_MAX_MD_SIZE];
  unsigned int hash_len;

  // Create message digest context
  EVP_MD_CTX* ctx = EVP_MD_CTX_new();
  if (!ctx) {
    Napi::Error::New(env, "Failed to create hash context")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Initialize SHA-256
  if (EVP_DigestInit_ex(ctx, EVP_sha256(), NULL) != 1) {
    EVP_MD_CTX_free(ctx);
    Napi::Error::New(env, "Failed to initialize hash")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Update hash with data
  if (EVP_DigestUpdate(ctx, data.Data(), data.Length()) != 1) {
    EVP_MD_CTX_free(ctx);
    Napi::Error::New(env, "Failed to update hash")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Finalize hash
  if (EVP_DigestFinal_ex(ctx, hash, &hash_len) != 1) {
    EVP_MD_CTX_free(ctx);
    Napi::Error::New(env, "Failed to finalize hash")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  EVP_MD_CTX_free(ctx);

  return Napi::Buffer<unsigned char>::Copy(env, hash, hash_len);
}

/**
 * Initialize the addon module
 */
Napi::Object Init(Napi::Env env, Napi::Object exports) {
  // Export functions
  exports.Set("encrypt", Napi::Function::New(env, Encrypt));
  exports.Set("decrypt", Napi::Function::New(env, Decrypt));
  exports.Set("generateRandomBytes", Napi::Function::New(env, GenerateRandomBytes));
  exports.Set("hashSHA256", Napi::Function::New(env, HashSHA256));

  return exports;
}

// Register the addon
NODE_API_MODULE(crypto_addon, Init)

} // namespace crypto_addon
```

### 4. JavaScript Wrapper

```javascript
// lib/index.js

const addon = require('../build/Release/crypto_addon.node');
const crypto = require('crypto');

/**
 * High-performance crypto library using native C++ addon
 */
class CryptoAddon {
  /**
   * Encrypt data using AES-256-GCM
   */
  static encrypt(plaintext, key) {
    // Validate inputs
    if (!Buffer.isBuffer(plaintext)) {
      throw new TypeError('plaintext must be a Buffer');
    }
    if (!Buffer.isBuffer(key)) {
      throw new TypeError('key must be a Buffer');
    }
    if (key.length !== 32) {
      throw new TypeError('key must be 32 bytes for AES-256');
    }

    // Generate random IV
    const iv = addon.generateRandomBytes(12);

    // Encrypt using native addon
    const result = addon.encrypt(plaintext, key, iv);

    return {
      ciphertext: result.ciphertext,
      iv: iv,
      tag: result.tag,
    };
  }

  /**
   * Decrypt data using AES-256-GCM
   */
  static decrypt(ciphertext, key, iv, tag) {
    // Validate inputs
    if (!Buffer.isBuffer(ciphertext)) {
      throw new TypeError('ciphertext must be a Buffer');
    }
    if (!Buffer.isBuffer(key)) {
      throw new TypeError('key must be a Buffer');
    }
    if (!Buffer.isBuffer(iv)) {
      throw new TypeError('iv must be a Buffer');
    }
    if (!Buffer.isBuffer(tag)) {
      throw new TypeError('tag must be a Buffer');
    }

    // Decrypt using native addon
    return addon.decrypt(ciphertext, key, iv, tag);
  }

  /**
   * Generate cryptographically secure random bytes
   */
  static generateRandomBytes(numBytes) {
    if (typeof numBytes !== 'number' || numBytes <= 0) {
      throw new TypeError('numBytes must be a positive number');
    }

    return addon.generateRandomBytes(numBytes);
  }

  /**
   * Hash data using SHA-256
   */
  static hash(data) {
    if (!Buffer.isBuffer(data)) {
      throw new TypeError('data must be a Buffer');
    }

    return addon.hashSHA256(data);
  }

  /**
   * Generate encryption key from password
   */
  static deriveKey(password, salt, iterations = 100000) {
    return new Promise((resolve, reject) => {
      crypto.pbkdf2(password, salt, iterations, 32, 'sha256', (err, key) => {
        if (err) reject(err);
        else resolve(key);
      });
    });
  }
}

module.exports = CryptoAddon;
```

### 5. Usage Example

```javascript
// test/crypto.test.js

const CryptoAddon = require('../lib/index');

async function testCryptoAddon() {
  console.log('🔐 Testing Native Crypto Addon\n');

  // Generate encryption key
  const password = 'secure-banking-password';
  const salt = CryptoAddon.generateRandomBytes(16);
  const key = await CryptoAddon.deriveKey(password, salt);

  console.log('✅ Generated 256-bit encryption key');
  console.log(`   Key: ${key.toString('hex').substring(0, 32)}...`);

  // Test data
  const accountData = {
    accountNumber: '1234567890',
    balance: 50000.00,
    customerId: 'CUST-001',
    transactions: [
      { id: 'TXN-001', amount: 1000 },
      { id: 'TXN-002', amount: -500 },
    ],
  };

  const plaintext = Buffer.from(JSON.stringify(accountData));
  console.log(`\n📄 Original Data (${plaintext.length} bytes):`);
  console.log(`   ${plaintext.toString().substring(0, 50)}...`);

  // Encryption benchmark
  console.log('\n🚀 Encrypting with native addon...');
  const encryptStart = Date.now();
  const encrypted = CryptoAddon.encrypt(plaintext, key);
  const encryptTime = Date.now() - encryptStart;

  console.log(`✅ Encrypted in ${encryptTime}ms`);
  console.log(`   Ciphertext: ${encrypted.ciphertext.toString('hex').substring(0, 32)}...`);
  console.log(`   IV: ${encrypted.iv.toString('hex')}`);
  console.log(`   Tag: ${encrypted.tag.toString('hex')}`);

  // Decryption benchmark
  console.log('\n🔓 Decrypting with native addon...');
  const decryptStart = Date.now();
  const decrypted = CryptoAddon.decrypt(
    encrypted.ciphertext,
    key,
    encrypted.iv,
    encrypted.tag
  );
  const decryptTime = Date.now() - decryptStart;

  console.log(`✅ Decrypted in ${decryptTime}ms`);
  console.log(`   Match: ${plaintext.equals(decrypted)}`);
  console.log(`   Data: ${decrypted.toString().substring(0, 50)}...`);

  // Hash test
  console.log('\n#️⃣ Hashing data...');
  const hashStart = Date.now();
  const hash = CryptoAddon.hash(plaintext);
  const hashTime = Date.now() - hashStart;

  console.log(`✅ Hashed in ${hashTime}ms`);
  console.log(`   SHA-256: ${hash.toString('hex')}`);

  // Performance comparison with pure JS
  console.log('\n⚡ Performance Comparison:');
  
  const crypto = require('crypto');
  const jsStart = Date.now();
  const cipher = crypto.createCipheriv('aes-256-gcm', key, encrypted.iv);
  cipher.update(plaintext);
  cipher.final();
  const jsTime = Date.now() - jsStart;

  console.log(`   Native addon: ${encryptTime}ms`);
  console.log(`   Pure JS: ${jsTime}ms`);
  console.log(`   Speedup: ${(jsTime / encryptTime).toFixed(2)}x`);
}

testCryptoAddon().catch(console.error);
```

### 6. package.json

```json
{
  "name": "crypto-addon",
  "version": "1.0.0",
  "description": "High-performance cryptography addon for Node.js",
  "main": "lib/index.js",
  "scripts": {
    "install": "node-gyp rebuild",
    "build": "node-gyp configure build",
    "clean": "node-gyp clean",
    "test": "node test/crypto.test.js"
  },
  "dependencies": {
    "node-addon-api": "^7.0.0"
  },
  "devDependencies": {
    "node-gyp": "^10.0.0"
  },
  "gypfile": true
}
```

### Building and Testing

```bash
# Install dependencies
npm install

# Build the addon
npm run build

# Run tests
npm test

# Expected output:
# 🔐 Testing Native Crypto Addon
# 
# ✅ Generated 256-bit encryption key
#    Key: a1b2c3d4e5f6...
# 
# 📄 Original Data (151 bytes):
#    {"accountNumber":"1234567890","balance":50000...
# 
# 🚀 Encrypting with native addon...
# ✅ Encrypted in 2ms
#    Ciphertext: 8f3a2b1c...
#    IV: 4d5e6f7a8b9c...
#    Tag: 1a2b3c4d...
# 
# 🔓 Decrypting with native addon...
# ✅ Decrypted in 1ms
#    Match: true
#    Data: {"accountNumber":"1234567890","balance":50000...
# 
# #️⃣ Hashing data...
# ✅ Hashed in 0ms
#    SHA-256: 2f8a9b7c...
# 
# ⚡ Performance Comparison:
#    Native addon: 2ms
#    Pure JS: 3ms
#    Speedup: 1.50x
```

---

## 💡 Example 2: Async Worker for Background Processing

Native addon with async work for non-blocking operations.

```cpp
// src/async_worker.cc

#include <napi.h>
#include <chrono>
#include <thread>
#include <cmath>

/**
 * Async worker for complex calculations
 */
class CalculationWorker : public Napi::AsyncWorker {
 public:
  CalculationWorker(Napi::Function& callback, double input)
      : AsyncWorker(callback), input_(input), result_(0) {}

  ~CalculationWorker() {}

  // Executed on worker thread (not blocking event loop)
  void Execute() override {
    // Simulate complex calculation
    result_ = 0;
    for (int i = 0; i < 1000000; i++) {
      result_ += std::sin(input_ * i) * std::cos(input_ * i);
    }
  }

  // Executed on main thread after Execute()
  void OnOK() override {
    Napi::HandleScope scope(Env());
    Callback().Call({Env().Null(), Napi::Number::New(Env(), result_)});
  }

  void OnError(const Napi::Error& e) override {
    Napi::HandleScope scope(Env());
    Callback().Call({e.Value(), Env().Undefined()});
  }

 private:
  double input_;
  double result_;
};

/**
 * Start async calculation
 */
Napi::Value CalculateAsync(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();

  if (info.Length() < 2 || !info[0].IsNumber() || !info[1].IsFunction()) {
    Napi::TypeError::New(env, "Expected (number, callback)")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  double input = info[0].As<Napi::Number>().DoubleValue();
  Napi::Function callback = info[1].As<Napi::Function>();

  CalculationWorker* worker = new CalculationWorker(callback, input);
  worker->Queue();

  return env.Undefined();
}

Napi::Object Init(Napi::Env env, Napi::Object exports) {
  exports.Set("calculateAsync", Napi::Function::New(env, CalculateAsync));
  return exports;
}

NODE_API_MODULE(async_worker, Init)
```

### Usage

```javascript
const addon = require('./build/Release/async_worker.node');

console.log('Starting async calculation...');

addon.calculateAsync(3.14159, (err, result) => {
  if (err) {
    console.error('Calculation failed:', err);
  } else {
    console.log('Calculation result:', result);
  }
});

console.log('Calculation running in background...');
// Event loop is not blocked!
```

---

## 🎯 Best Practices

### 1. Error Handling

```cpp
// Always handle errors gracefully
Napi::Value SafeFunction(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();

  try {
    // Your code here
    
    return Napi::String::New(env, "success");
  } catch (const std::exception& e) {
    Napi::Error::New(env, e.what()).ThrowAsJavaScriptException();
    return env.Null();
  }
}
```

### 2. Memory Management

```cpp
// Use RAII and smart pointers
Napi::Value ProcessData(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();
  
  // Allocate on stack when possible
  std::vector<unsigned char> buffer(1024);
  
  // Use unique_ptr for heap allocations
  auto ctx = std::unique_ptr<EVP_CIPHER_CTX, decltype(&EVP_CIPHER_CTX_free)>(
    EVP_CIPHER_CTX_new(),
    EVP_CIPHER_CTX_free
  );
  
  // Automatically cleaned up
  return env.Null();
}
```

### 3. Type Validation

```cpp
// Validate all inputs
Napi::Value ValidatedFunction(const Napi::CallbackInfo& info) {
  Napi::Env env = info.Env();

  // Check argument count
  if (info.Length() < 2) {
    Napi::TypeError::New(env, "Wrong number of arguments")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Check types
  if (!info[0].IsBuffer()) {
    Napi::TypeError::New(env, "Argument 0 must be a Buffer")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  // Range validation
  int32_t value = info[1].As<Napi::Number>().Int32Value();
  if (value < 0 || value > 100) {
    Napi::RangeError::New(env, "Value must be between 0 and 100")
        .ThrowAsJavaScriptException();
    return env.Null();
  }

  return env.Null();
}
```

### 4. Async Workers for Long Operations

```cpp
// Use AsyncWorker for operations > 10ms
class ExpensiveOperation : public Napi::AsyncWorker {
  // Never block the event loop in Execute()
  void Execute() override {
    // Long-running operation here
    // This runs on a separate thread
  }

  void OnOK() override {
    // Called on main thread when done
    // Safe to interact with JavaScript here
  }
};
```

---

## 📚 Common Interview Questions

### Q1: When should you use a native addon?

**Answer**:
Use native addons when:
- CPU-intensive computations (cryptography, image processing)
- Need to use existing C/C++ libraries
- Require direct hardware access
- Performance critical tight loops
- JavaScript performance insufficient even after optimization

### Q2: What is N-API and why use it?

**Answer**:
N-API is an ABI-stable API for building native addons:
- **Version independent**: Works across Node.js versions without recompilation
- **Stable**: API doesn't change between versions
- **Recommended**: Official Node.js recommendation for new addons

### Q3: How do you handle async operations in native addons?

**Answer**:
Use `Napi::AsyncWorker`:
```cpp
class MyWorker : public Napi::AsyncWorker {
  void Execute() override {
    // Runs on worker thread (non-blocking)
  }
  void OnOK() override {
    // Runs on main thread with result
  }
};
```

### Q4: How do you debug native addons?

**Answer**:
1. **GDB/LLDB**: Native debugger
   ```bash
   gdb --args node script.js
   ```
2. **Logging**: Add `std::cout` statements
3. **Valgrind**: Memory leak detection
4. **AddressSanitizer**: Memory error detection

### Q5: What are the security considerations?

**Answer**:
1. **Input validation**: Always validate from JavaScript
2. **Buffer bounds**: Check all buffer accesses
3. **Memory safety**: Use RAII, avoid raw pointers
4. **Error handling**: Never crash, throw exceptions
5. **Dependencies**: Keep OpenSSL/libraries updated

---

## ✅ Summary & Key Takeaways

### Native Addon Development Checklist

```yaml
✅ Setup:
  - [ ] Install node-gyp and build tools
  - [ ] Install node-addon-api
  - [ ] Configure binding.gyp
  - [ ] Setup cross-platform build

✅ Development:
  - [ ] Use N-API (not V8 direct)
  - [ ] Validate all inputs
  - [ ] Use AsyncWorker for long ops
  - [ ] Proper memory management (RAII)
  - [ ] Comprehensive error handling

✅ Testing:
  - [ ] Unit tests for all functions
  - [ ] Memory leak testing (Valgrind)
  - [ ] Performance benchmarks
  - [ ] Cross-platform testing
  - [ ] Stress testing

✅ Performance:
  - [ ] Only use for CPU-intensive tasks
  - [ ] Don't block event loop
  - [ ] Minimize JS↔C++ transitions
  - [ ] Use efficient data structures
  - [ ] Profile and optimize
```

### When to Use Native Addons

```
┌──────────────────────────────────────────────────────┐
│         Decision Tree: Native Addon?                 │
├──────────────────────────────────────────────────────┤
│                                                       │
│ Is it CPU-intensive?                                 │
│  ├─ No → Use JavaScript ✅                           │
│  └─ Yes ↓                                            │
│                                                       │
│ Is JavaScript fast enough?                           │
│  ├─ Yes → Use JavaScript ✅                          │
│  └─ No ↓                                             │
│                                                       │
│ Can you use existing npm package?                    │
│  ├─ Yes → Use npm package ✅                         │
│  └─ No ↓                                             │
│                                                       │
│ Do you need native libraries?                        │
│  ├─ Yes → Native addon ✅                            │
│  └─ No ↓                                             │
│                                                       │
│ Worth the maintenance cost?                          │
│  ├─ Yes → Native addon ✅                            │
│  └─ No → Optimize JavaScript ✅                      │
│                                                       │
└──────────────────────────────────────────────────────┘
```

---

**Status**: ✅ Complete with production-ready native addon development for high-performance banking operations!
