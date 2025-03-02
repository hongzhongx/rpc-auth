
@taiyinet/rpc-auth
=================

JSONRPC 2.0 authentication with taiyi authorities


Specification
-------------

### Overview

Request signing for [JSON-RPC 2.0](http://www.jsonrpc.org/specification) implemented using [taiyi](https://taiyi.io) authorities.

### Design Goals

* Do not require request header modification.
    * Result: Signature/auth must be in message body
* Signed requests do not violate json-rpc spec.
    * Result: Extensions must go into `params`.
* Method name is not obscured so that it may be routed properly to the correct handler/backend.
    * Result: `method` remains unchanged by signing.

### Signed request

Requests are signed with taiyi keys belonging to the sender.

Example JSON-RPC request:
```json
{
    "jsonrpc": "2.0",
    "id": 123,
    "method": "foo.bar",
    "params": {
        "hello": "there"
    }
}
```

Above request signed with the posting key belonging to `foo`:
```json
{
    "jsonrpc": "2.0",
    "method": "foo.bar",
    "id": 123,
    "params": {
        "__signed": {
            "account": "foo",
            "nonce": "1773e363793b44c3",
            "params": "eyJoZWxsbyI6InRoZXJlIn0=",
            "signatures": [
                "1f02df499f15c8757754c11251a6e5238296f56b17f7229202fce6ccd7289e224c49c32eaf77d5905e2b4d8a8a5ddcc215c51ce45c207ef0f038328200578d1bee"
            ],
            "timestamp": "2017-11-26T16:57:40.633Z"
        }
    }
}
```

Signature creation pseudocode:
```python

# JSON+Base64 request params
params = base64(json_encode(request['params']))

# 8 byte nonce
nonce = random_bytes(8)

# ISO 8601 formatted timestamp
timestamp = date_now() # "2017-11-26T16:57:40.633Z"

# Signer account name
account = 'foo'

# Private posting key belonging to foo
signing_key = PrivateKey('...')

# Signing constant K (sha256('taiyi_jsonrpc_auth'))
K = bytes_from_hex('7701ccaaa966600b83b68937a077587941fd9ab1ff144e396eb4239df7a8a06b')

# first round of sha256
first = sha256(timestamp + account + method + params)

# message to be signed
message = sha256(K + first + nonce)


signature = ecdsa_sign(message, signing_key)
```

### Signature validation

  1. Entire request must be <64k for sanity/anti-DoS
  1. Request must be valid json and json-rpc
  1. `request['params']['__signed']` must exist
  1. `request['params']['__signed']` must be the only item in `request['params']`
  1. `request['params']['__signed']['params']` must be valid base64
  1. `request['params']['__signed']['params']` when base64 decoded must be valid json
  1. `request['params']['__signed']['nonce']` must exist and be a hex string of length 16 (8 bytes decoded)
  1. `request['params']['__signed']['timestamp']` must exist and be a valid iso8601 datetime ending in Z
  1. `request['params']['__signed']['timestamp']` must be within the last 60 seconds
  1. `request['params']['__signed']['account']` must be a valid taiyi blockchain account
  1. `request['params']['__signed']['signature']` must be a hex string >= 64 chars (32+ bytes decoded)
  1. construct `first = sha256( request['params']['__signed']['timestamp'] + request['params']['__signed']['account'] + request['method'] + request['params']['__signed']['params'] ).bytes()`
  1. construct `signedstring = sha256( K + first + unhexlify(nonce)).bytes()`
  1. check signature, signedstring against posting authorities for `request['params']['__signed']['account']`
