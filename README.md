# iyi-jwt

[JSON Web Tokens](https://datatracker.ietf.org/doc/html/rfc7519) for
[iyi](https://iyi.dev), after
[crystal-community/jwt](https://github.com/crystal-community/jwt): tokens
signed with HMAC, RSA, RSA-PSS, ECDSA or Ed25519, decoded with their
signature verified and their registered claims checked. Written in iyi, with
no dependency outside iyi's own library.

Errors are values (SPEC.md III.1): `encode` answers the token or an
`EncodeError`, `decode` the `Token` or a `Failure`.

## Requirements

iyi 0.18.0 or later.

## Installation

```sh
iyi get github.com/iyilang/jwt --as jwt
```

Files then `import jwt` for the root module (`Jwt.encode`, `Jwt.decode`,
`Jwt.decode_unverified`, `Jwt::Token`, `Jwt::VERSION`) and its other modules
as `jwt/jwt/...`: the short name, then the module's path in the package.

## Usage

```iyi
module app

import jwt
import jwt/jwt/algorithms::{Algorithm}
import jwt/jwt/errors::{Failure}

def run : Nil | Failure
  token = Jwt.encode({"foo" => "bar"}, "SecretKey", Algorithm::HS256)!
  # => "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJmb28iOiJiYXIifQ.Y3shN5Wh..."

  decoded = Jwt.decode(token, "SecretKey", Algorithm::HS256)!
  decoded.payload["foo"].as_s # => "bar"
  decoded.header["alg"].as_s  # => "HS256"
  nil
end
```

The payload is anything `JSON.to_json` writes as an object: a
`Hash(String, V)`, a `Hash(String, Any)` for mixed values, a type that
derives `serializable`. A decoded token's `header` and `payload` are
`Hash(String, Any)`.

Header parameters go in a hash after the algorithm. `typ` replaces `JWT`;
`alg` is passed over, since the header names the algorithm the token is
signed with:

```iyi
Jwt.encode(payload, key, Algorithm::RS256, {"kid" => "2026-10"})!
```

The key can be chosen from the token's header — by its `kid`, say — with a
block. The header and the payload reach the block unverified; nil refuses the
token.

```iyi
Jwt.decode(token, Algorithm::RS256) do |header, payload|
  public_keys[header["kid"].as_s]?
end!
```

`Jwt.decode_unverified(token)` reads a token with nothing checked, neither
its signature nor its claims: for looking at a token, never for trusting it.

### Errors

```iyi
case result = Jwt.decode(token, key, Algorithm::HS256, aud: "api")
in Jwt::Token            then serve(result.payload)
in ExpiredSignatureError then puts "log in again"
in Failure               then puts "refused: #{result.message}"
end
```

| Error | When |
|---|---|
| `DecodeError` | not three segments of unpadded base64url JSON objects; `alg` missing; `exp` or `nbf` not a number; a `crit` header |
| `VerificationError` | the signature is not the key's; the header names another algorithm than the one asked for; the key block answered nil |
| `InvalidKeyError` | the key is not one the algorithm takes |
| `UnsupportedAlgorithmError` | signing with ES384 or ES512 |
| `ExpiredSignatureError` | `exp` is now or past |
| `ImmatureSignatureError` | `nbf` is still to come |
| `InvalidAudienceError` | `aud` asked for, and missing or another |
| `InvalidIssuerError` | `iss` asked for, and missing or another |
| `InvalidSubjectError` | `sub` asked for, and missing or another |
| `InvalidTypeError` | `typ` asked for, and the header's is missing or another |

`ValidationError` is the last six, `EncodeError` what `encode` answers, and
`Failure` all of them; each is in `jwt/jwt/errors`.

## Algorithms

| Algorithm | Signs with | Verifies with |
|---|---|---|
| `None` | nothing: the signature is empty | an empty signature only |
| `HS256`, `HS384`, `HS512` | the secret | the secret |
| `RS256`, `RS384`, `RS512`, `PS256`, `PS384`, `PS512` | a PEM `PRIVATE KEY` or `RSA PRIVATE KEY` | a PEM `PUBLIC KEY` or `RSA PUBLIC KEY`, or the private key |
| `ES256` | a PEM `PRIVATE KEY` or `EC PRIVATE KEY` on P-256 | a PEM `PUBLIC KEY`, or the private key |
| `ES384`, `ES512` | — | a PEM `PUBLIC KEY` on P-384 or P-521, or the private key |
| `EdDSA` (Ed25519) | the 32-byte seed in hex, or a PEM `PRIVATE KEY` | the 32-byte public key in hex, a PEM `PUBLIC KEY`, or the private key in PEM |

RSA keys must be 2048 bits or more (RFC 7518 §3.3). An EC key must be on the
algorithm's curve. ES384 and ES512 verify only: iyi's `std/ecdsa` signs on
P-256 alone.

## Claims

`decode` checks, once the signature holds:

- `exp` — refused from that second on (RFC 7519 §4.1.4).
- `nbf` — refused before that second (§4.1.5).
- `aud:` — when given, `aud` must be that string or an array holding it.
- `iss:`, `sub:` — when given, the claim must be that string.
- `typ:` — when given, the header's `typ` must be it, ignoring case.

`validate: false` skips the claims and keeps the signature check; `typ:` is
checked either way.

```iyi
Jwt.decode(token, key, Algorithm::HS256, aud: "api", iss: "auth.example.com")!
```

## Differences from crystal-community/jwt

- `encode` and `decode` answer errors as values rather than raising.
- The algorithm is always the caller's. `decode` refuses a token whose header
  names another one, and the block form takes the algorithm as an argument
  rather than reading it from the header.
- No `verify: false`; `Jwt.decode_unverified` reads a token without checking
  it.
- A hex Ed25519 key given to `decode` is the public key, never a seed to
  derive one from: trying the public key as a seed lets anyone sign a token
  it accepts.
- `exp` refuses the token at its own second, as RFC 7519 says, where
  crystal-community/jwt accepted it for that second.
- A header with `crit` is refused (RFC 7515 §4.1.11), and so is a payload
  that is not a JSON object.
- RSA keys under 2048 bits are refused.
- Not ported: ES256K (iyi's library has no secp256k1), signing with ES384
  and ES512, and JWKS.

## Development

```sh
iyi test .
iyi fmt --check jwt.iyi jwt_test.iyi jwt examples
```
