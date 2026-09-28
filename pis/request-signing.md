# How to Sign an Outbound Payment API Request

**Diátaxis type:** How-to guide  
**Outcome:** Produce a deterministic, auditable signed request without exposing the private key

## Procedure

1. **Read the provider contract.** Record the permitted algorithm, protected headers, key identifier format and exact signing input.
2. **Build the final request once.** Fix the HTTP method, encoded request target, selected headers and body bytes before signing.
3. **Canonicalise only as specified.** Do not invent header ordering, whitespace rules or path decoding.
4. **Sign in a trusted backend.** Keep private keys in an approved key-management or secrets system; never ship them to a browser or mobile app.
5. **Attach the signature without rebuilding the body.**
6. **Log safe evidence.** Record the request ID, key ID, algorithm, body hash and verification outcome—not keys, tokens or sensitive payloads.

```text
body_bytes = serialize_once(payment)
signing_input = provider_defined_input(
  method,
  encoded_path_and_query,
  selected_headers,
  body_bytes
)
signature = approved_signer.sign(key_id, algorithm, signing_input)
send(method, path, headers + signature, body_bytes)
```

## Rotation

Support an overlap period where the provider permits it. Select the active signing key explicitly, retain the previous public verification key for the documented period and make rollback possible. A missing or unknown key identifier should fail closed.

## Tests

- the same logical request produces the expected signing input;
- changing one covered byte invalidates the signature;
- encoded and decoded paths are not confused;
- the wrong environment key fails;
- an unapproved algorithm is rejected; and
- logs contain identifiers but no signing material.

## References

- [RFC 7515 — JSON Web Signature](https://www.rfc-editor.org/rfc/rfc7515)
- [RFC 7517 — JSON Web Key](https://www.rfc-editor.org/rfc/rfc7517)
