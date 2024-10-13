---
layout: post
title: 'HMAC and Digital Signatures'
date: 2024-10-12 16:35:57 +0700
categories: dev
---

**HMAC** stands for **Hash-based Message Authentication Code**, it is a symmetric, which mean there is only one key (for example API secret key) that both parties use to hash and verify messages.
**Digital Signatures** is asymmetric, which mean there is two keys (a keypair). The sender keeps the private key and the receiver keeps the public key. The private key is used to sign the message, and the public key is used to verify it.

Let’s see how **HMAC** is used in Ruby in the following example.

```ruby
require 'openssl'

def prepare_message(request_body, nonce, api_key)

# Concatenate the request body with the nonce
message = request_body + nonce.to_s

# Compute the HMAC with SHA256
OpenSSL::HMAC.hexdigest('sha256', api_key, message)
end

def verify_message(request_body, nonce, signature, api_key)

# Generate the expected signature
expected_signature = prepare_message(request_body, nonce, api_key)

# Compare the computed HMAC with the signature
expected_signature == signature
end

# Example usage:
request_body = '{"hello": "world"}'
nonce = 1243549809
api_key = 'sk_test_secret'

# Prepare the message signature
signature = prepare_message(request_body, nonce, webhook_key)
puts "Generated Signature: #{signature}"

# Verify the signature
if verify_message(request_body, nonce, signature, webhook_key)
puts "Signature verified!"
else
puts "Signature verification failed."
end
```

In second example, I implemented **Digital Signatures** in Ruby. We will see that, instead of using a single `secret key` as in the first example, we use a `keypair`: `public key` and `private key` (The library changes from `OpenSSL::HMAC` to `OpenSSL::PKey`).

```ruby
require 'openssl'

# Generate a new RSA key pair
private_key = OpenSSL::PKey::RSA.new(2048)

# Get the public key
public_key = key.public_key

def prepare_message(request_body, timestamp, private_key)

# Concatenate the request body with the timestamp
message = request_body + timestamp.to_s

# Create a digest (SHA256 is commonly used)
digest = OpenSSL::Digest::SHA256.new

# Sign the message with the private key
signature = private_key.sign(digest, message)

# Convert binary format to hexadecimal format for human readable
signature.unpack1('H\*')
end

def verify_message(request_body, timestamp, hex_signature, public_key)

# Concatenate the request body with the timestamp
message = request_body + timestamp.to_s

# Create a digest (SHA256 is commonly used)
digest = OpenSSL::Digest::SHA256.new

# Convert hexadecimal to binary before verify
signature_bytes = [hex_signature].pack('H\*')

# Verify the signature with the public key
public_key.verify(digest, signature_bytes, message)
end

# Example usage:
request_body = '{"hello": "world"}'
timestamp = Time.now.to_i # Get current timestamp as an integer

# Prepare the message signature
hex_signature = prepare_message(request_body, timestamp, private_key)
puts "Generated Signature: #{hex_signature}" # Print signature in hex format

# Verify the signature
if verify_message(request_body, timestamp, hex_signature, public_key)
puts "Signature verified!"
else
puts "Signature verification failed."
end
```

In the second example, I changed `nonce` to `timestamp` to reflect the preferences of different companies when design their APIs.
It is also important to note that `private_key.sign` returns a `binary` output instead of `hex`, unlike `OpenSSL::HMAC.hexdigest`. Therefore, we need to `unpack` it to `hexadecimal` for human readability before transferring the data, and `pack` it back to `binary` before verifying.
