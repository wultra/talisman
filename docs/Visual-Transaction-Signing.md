# Visual Transaction Signing

The FIDO2 protocol is intended as an authenticaton protocol. As a result, it does not have a straight-forward implementation of transaction signing. When used in the web browser via WebAuthn API, the challenge provided to the web browser is hashed. As a result, the challenge itself cannot be interpreted in the external authenticator. FIDO2 extensions that were intended for solving the use-case of providing visual data to the authenticator (`txAuthSimple`, `txAuthGeneric`) are not implemented in the web browser (see the [discussion on Github](https://github.com/w3c/webauthn/pull/2020)).

## Authenticator Workaround

Talisman implements a stable workaround to enable visual transaction signing with the standard FIDO2 protocol by interpreting payload carried in the `credentialId` value slightly differently than is usual.

Conventionally, the `credentialId` is just a random ID of the authenticator device. The WebAuthn initiated activity commonly results in the CTAP2.x call to the external authenticator with the `credentialId` value. If the `credentialId` sent to the external authenticator matches the known value in the authenticator, the authenticator knows the call was intended for it.

Talisman, however, supports `credenaialId` value with data suffix, where the operation data for signing is appended to the authenticator identifier:

```
final byte[] credentialId = ByteUtils.concat(authenticator.identifier, '&', operation.data)
```

Talisman then looks if the provided `credentialId` **prefix** matches the expected authenticator identifier, and if it does, it looks at the data after such ID (validating the inputs for length and separators, of course). If the `credentialId` **suffix** matches expected operation data format, the authenticator displays visual challenge and additionally signs the operation data.

## Server-Side Workaround

The PowerAuth server can then validate the alternate value of the assertion / signature based on the fact it was calculated via Talisman (evidenced by AAGUID value), like so:

```java
// Get the challenge value from the assertion part related to client data
final String challenge = clientDataJSON.getChallenge();

// Determine if there is expected to be data suffix (Talisman) or not (other FIDO2 devices)
byte[] dataSuffix = null;
if (Fido2DefaultAuthenticators.isWultraModel(aaguid)) {
    final String[] split = challenge.split("&", 2);
    if (split.length != 2) {
        return null;
    }
    dataSuffix = split[1].getBytes(StandardCharsets.UTF_8);
    if (dataSuffix == null) {
        return null;
    }
}

// Build signable data, including the suffix if it is present as additional component
final byte[] signableData;
if (dataSuffix != null) {
    signableData = ByteUtils.concat(authData.getEncoded(), Hash.sha256(clientDataJSON.getEncoded()), dataSuffix);
} else {
    signableData = ByteUtils.concat(authData.getEncoded(), Hash.sha256(clientDataJSON.getEncoded()));
}

// Get public key
final byte[] publicKeyBytes = authenticatorDetail.getPublicKeyBytes();
final PublicKey publicKey = keyConvertor.convertBytesToPublicKey(publicKeyBytes);

// Validate signature using desired algorithm
return signatureUtils.validateECDSASignature(signableData, signature, publicKey);
```

This alternate matching does not go against FIDO2 standard, as FIDO2 supports various interpretations of the `credentialId` value, including use-cases such as stateless authenticators. The standard uses `byte[]` as the `credentialId` value type, to ensure the value can contain any data payload.

The modified signing also does not cause any practical issues - the web browser has no way of validating the signature value is correct, as it is server's responsibility. Note that the modified signing is only applied for Talisman devices - other FIDO2 devices follow the standard with no modifications, and the validation works as well.