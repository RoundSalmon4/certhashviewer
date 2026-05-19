# APK Certificate Hash Viewer

A client-side web tool that extracts signing certificate SHA-256 hashes from APK files. No upload needed — everything runs in your browser.

## How it works

The tool performs these steps entirely in JavaScript:

1. **Reads the APK as a ZIP archive** — uses [JSZip](https://stuk.github.io/jszip/) to extract `AndroidManifest.xml` for the package name, and to locate the APK Signing Block.

2. **Finds the APK Signing Block** — locates the End of Central Directory (EOCD) record, reads the Central Directory offset, then searches backward for the magic bytes `APK Sig Block 42`. This identifies the v2/v3 APK signing block appended between the ZIP content and the Central Directory.

3. **Extracts the signing certificate** — parses the ID-value pairs in the signing block looking for signer IDs (`0x7109871a` for v2, `0xf05368c0` for v3). Within each signer block, scans for DER-encoded X.509 certificates matching `0x30 0x82` (ASN.1 SEQUENCE with long-form length). Non-certificate structures (SubjectPublicKeyInfo, TBSCertificate bodies) are filtered out by verifying the inner SEQUENCE begins with an INTEGER serial or EXPLICIT version tag, not an OID. Only unique non-nested certificates are hashed.

4. **Computes the SHA-256 hash** — feeds the certificate's DER-encoded bytes through the [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API) (`crypto.subtle.digest`). The resulting hash is formatted as colon-separated uppercase hex (e.g., `3A:04:A8:...:02:B2`).

5. **Extracts the package name** — parses the binary `AndroidManifest.xml`'s string pool to find the package identifier.

The APK is never sent to any server. All processing uses `ArrayBuffer`, `DataView`, and native browser APIs.

## Copying results

Each `[copy]` button copies the package name and hash in newline-separated format:

```
dev.soupslurpr.appverifier
3A:04:A8:0B:2A:88:33:4C:74:74:85:F0:B2:15:16:40:A3:8B:B3:D2:D7:3A:8E:AB:81:DF:50:3E:0F:02:02:B2
```

This format is compatible with Android apps that accept clipboard verification, such as [AppVerifier](https://github.com/soupslurpr/AppVerifier).

## Reporting suspected false positives

If you believe a hash shown by this tool is not a real signing certificate, click the `[report]` link next to it. This opens a pre-filled GitHub issue with the hash, APK filename, and package name for inspection.

## Comparison with AppVerifier

AppVerifier uses a **different code path** to obtain the same certificate hash. Cross-verifying between the two provides stronger confidence:

| | This tool | AppVerifier |
|---|---|---|
| **Platform** | Browser (JavaScript) | Android (Kotlin) |
| **APK parsing** | Manually reads bytes: EOCD → signing block magic → ID-value pairs → DER certificate | Delegates to Android's `PackageManager.getPackageArchiveInfo()` with the `GET_SIGNING_CERTIFICATES` flag |
| **Certificate extraction** | Scans the v2/v3 signing block binary for `0x30 0x82` DER SEQUENCE markers | Uses Android's `SigningInfo` class which returns `Signature[]` objects |
| **Hash computation** | Web Crypto API (`crypto.subtle.digest('SHA-256', ...)`) | Java `MessageDigest.getInstance("SHA-256")` |
| **Certificate byte source** | DER bytes directly parsed from the v2/v3 APK signing block | `Signature.toByteArray()` from the Android framework |

Neither tool relies on external command-line tools (`apksigner`, `keytool`, `openssl`). Both implement the same specification (Android APK signing scheme v2/v3) independently. Matching hashes across implementations reduces the chance of a shared bug.

## Usage

Open `index.html` in any modern browser, then drag an APK onto the page or click to browse. The hash displays immediately — no installation or backend required.

## Notes

Certificate extraction uses a targeted byte scanner rather than a full ASN.1 parser. The tool looks for `0x30 0x82` (SEQUENCE with long-form length), then validates the inner SEQUENCE starts with an INTEGER serial or EXPLICIT version tag — which distinguishes X.509 certificates from other DER structures like SubjectPublicKeyInfo. This approach is simpler and more auditable for the specific APK signing block format, but would need to be extended for edge cases like v3 key rotation lineage or non-standard certificate encoding.

## License

[MIT](LICENSE)
