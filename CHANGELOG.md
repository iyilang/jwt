# Changelog

## 0.1.0

- **First release: JSON Web Tokens written in iyi, after
  crystal-community/jwt.** `Jwt.encode` signs any JSON object with `none`,
  HS256/384/512, RS256/384/512, PS256/384/512, ES256 or EdDSA (Ed25519);
  `Jwt.decode` verifies those and ES384/ES512, refuses a header naming
  another algorithm than the caller's, and checks `exp`, `nbf`, `aud`, `iss`,
  `sub` and `typ`; a block can pick the key by the header's `kid`.
  `Jwt.decode_unverified` reads a token without checking it. Errors are
  values, one type per crystal-community/jwt exception. Keys are PEM
  (PKCS #1, PKCS #8, SEC 1, SubjectPublicKeyInfo) or hex for Ed25519. Held by
  `iyi test .`, with crystal-community/jwt's vectors and tokens OpenSSL
  signed.
