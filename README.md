# APK Certificate Hash Viewer

A client-side web tool that extracts signing certificate SHA-256 hashes from APK files. No upload needed — everything runs in your browser.

## How it works

The tool performs these steps entirely in JavaScript:

1. **Reads the APK as a ZIP archive** — uses [JSZip](https://stuk.github.io/jszip/) to extract `AndroidManifest.xml` for the package name, and to locate the APK Signing Block.

2. **Finds the APK Signing Block** — locates the End of Central Directory (EOCD) record, reads the Central Directory offset, then searches backward for the magic bytes `APK Sig Block 42`. This identifies the v2/v3 APK signing block appended between the ZIP content and the Central Directory.

3. **Extracts the signing certificate** — parses the ID-value pairs in the signing block looking for signer IDs (`0x7109871a` for v2, `0xf05368c0` for v3). Within each signer block, locates the DER-encoded X.509 certificate by scanning for `0x30 0x82` (ASN.1 SEQUENCE with long-form length) with a reasonable certificate size (300–5000 bytes). The certificate bytes are passed to the browser's native X.509 parser.

4. **Computes the SHA-256 hash** — feeds the certificate's DER-encoded bytes through the [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API) (`crypto.subtle.digest`). The resulting hash is formatted as colon-separated uppercase hex (e.g., `3A:04:A8:...:02:B2`).

5. **Extracts the package name** — parses the binary `AndroidManifest.xml`'s string pool to find the package identifier.

The APK is never sent to any server. All processing uses `ArrayBuffer`, `DataView`, and native browser APIs.

## How this differs from AppVerifier

AppVerifier uses a **different code path** to obtain the same certificate hash, which makes cross-verification useful:

| | This tool | AppVerifier |
|---|---|---|
| **Platform** | Browser (JavaScript) | Android (Kotlin) |
| **APK parsing** | Manually reads bytes: EOCD → signing block magic → ID-value pairs → DER certificate | Delegates to Android's `PackageManager.getPackageArchiveInfo()` with the `GET_SIGNING_CERTIFICATES` flag |
| **Certificate extraction** | Scans the v2/v3 signing block binary for `0x30 0x82` DER SEQUENCE markers | Uses Android's `SigningInfo` class which returns `Signature[]` objects |
| **Hash computation** | Web Crypto API (`crypto.subtle.digest('SHA-256', ...)`) | Java `MessageDigest.getInstance("SHA-256")` |
| **Certificate byte source** | DER bytes directly parsed from the v2/v3 APK signing block | `Signature.toByteArray()` from the Android framework |

Neither tool relies on external command-line tools (`apksigner`, `keytool`, `openssl`). Both implement the same specification (Android APK signing scheme v2/v3) independently. If both produce the same hash, it is strong evidence the hash is correct — a shared bug across independent implementations is unlikely.

## Usage

Open `index.html` in any modern browser, then drag an APK onto the page or click to browse. The hash displays immediately — no installation or backend required.
