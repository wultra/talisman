# Visual Transaction Signing

The FIDO2 protocol is intended as an authentication protocol. As a result, it does not have a straightforward implementation of transaction signing. When used in the web browser via WebAuthn API, the challenge provided to the web browser is hashed. As a result, the challenge itself cannot be interpreted in the external authenticator. FIDO2 extensions that were intended for solving the use-case of providing visual data to the authenticator (`txAuthSimple`, `txAuthGeneric`) are not implemented in the web browser (see the [discussion on Github](https://github.com/w3c/webauthn/pull/2020)).

## Authenticator Workaround

Talisman implements a stable workaround to enable visual transaction signing with the standard FIDO2 protocol by interpreting the payload carried in the `credentialId` value slightly differently than usual.

Conventionally, the `credentialId` is simply a random ID assigned to the authenticator device. The WebAuthn-initiated activity commonly results in the CTAP2.x call to the external authenticator with the `credentialId` value. If the `credentialId` sent to the external authenticator matches the known value in the authenticator, the authenticator knows the call was intended for it.

Talisman, however, supports `credenaialId` value with data suffix, where the operation data for signing is appended to the authenticator identifier:

```
final byte[] credentialId = ByteUtils.concat(authenticator.identifier, operation.data)
```

If you provide the concatenated identifier to `allowCredentials` item in the WebAuthn request (non-discoverable credentials only), Talisman checks if the provided `credentialId` **prefix** matches the expected authenticator identifier, and if it does, it looks at the data after such ID (validating the inputs for length and separators, of course). If the `credentialId` **suffix** matches the expected [operation data format](https://developers.wultra.com/components/enrollment-server/develop/documentation/Operation-Data), the authenticator displays a visual challenge and additionally signs the operation data.

## Server-Side Workaround

The PowerAuth server can then validate the alternate value of the assertion/signature based on the fact that it was calculated via Talisman (evidenced by the AAGUID value), like so:

```java
// Determine if there is expected to be a data suffix (Talisman) or not (other FIDO2 devices)
byte[] dataSuffix = null;
if (Fido2DefaultAuthenticators.isWultraModel(aaguid)) {

    // Get the challenge value from the client data
    final String challenge = clientDataJSON.getChallenge();

    // Get the operation data for the challenge
    final String operationData = service.getOperationDataForChallenge(challenge);
    if (operationData == null) { // missing operation data
        return null;
    }
    dataSuffix = operationData.getBytes(StandardCharsets.UTF_8);
    if (dataSuffix == null) { // encoding error
        return null;
    }
}

// Build signable data, including the suffix if it is present as an additional component
final byte[] signableData;
if (dataSuffix != null) {
    signableData = ByteUtils.concat(authData.getEncoded(), Hash.sha256(clientDataJSON.getEncoded()), dataSuffix);
} else {
    signableData = ByteUtils.concat(authData.getEncoded(), Hash.sha256(clientDataJSON.getEncoded()));
}

// Get authenticator's public key
final Authenticator authenticator = service.getAuthenticatorForCredentialId(credentialId);
final PublicKey publicKey = authenticator.getPublicKey();

// Validate signature provided by authenticator
return SignatureUtils.validateECDSASignature(signableData, signature, publicKey);
```

This alternative matching does not conflict with the FIDO2 standard, as FIDO2 supports various interpretations of the `credentialId` value, including use cases such as stateless authenticators. The standard uses `byte[]` as the `credentialId` value type, to ensure the value can contain any data payload.

The modified signing also does not cause any practical issues - the web browser has no way of validating that the signature value is correct, as it is the server's responsibility. The modified signing is only applied to Talisman devices. Other FIDO2 devices follow the standard with no modifications, and the validation works as well.
