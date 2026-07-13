# Security Rules

Security is mandatory, not optional. These rules apply to all Android projects.

## 1. Networking

- **HTTPS only**: All remote network communications must use HTTPS. Never declare `android:usesCleartextTraffic="true"` in the production manifest.
- **Certificate Pinning**: Configure certificate pinning using `BuildConfig.SSL_PINNING` and `BuildConfig.SSL_DOMAIN` values defined in your `gradle.properties` file. Do not hardcode pinning certificates or domains directly inside source files.
- **TLS Configuration**: Do not disable TLS validation, trust-all certificates, or accept invalid/self-signed certificates in production code.

---

## 2. Sensitive Data & Storage

- **Token Storage**: Access tokens, session cookies, passwords, and Personally Identifiable Information (PII) must be stored securely using Android's `EncryptedSharedPreferences` (or via the edtslib `HttpHeaderLocalSource`). Plain `SharedPreferences` must not be used for sensitive tokens or credentials.
- **No Hardcoded Secrets**: API keys, credentials, base URLs, and encryption keys must be kept in the gitignored `gradle.properties` and injected via `BuildConfig` properties. Never commit raw secrets or configuration keys to the source repository.
- **Root Detection**: For financial, payment, or enterprise applications requiring device integrity checks, implement root/jailbreak detection utilizing `rootbeer-lib` or Google Play Integrity API.

---

## 3. Logging & Diagnostics

- **Sanitize Logs**: Never log passwords, access tokens, auth headers, credit card numbers, or full HTTP response payloads containing PII.
- **Release Logs**: Keep diagnostic logging restricted to non-sensitive fields. Debug logging (Timber) must be disabled or filtered out using a custom release Tree in release builds.

---

## 4. File & Folder Deletion

- **Never use `rm` or `rm -rf`**: Do not use `rm` or `rm -rf` to delete workspace files or directories. Use the `trash` command instead (e.g. `trash path/to/file`) which moves files safely to the system Trash, preventing accidental permanent data loss.

---

## 5. Sensitive Files Agents Must Not Touch

AI agents must not edit, move, print, or commit these files unless the developer explicitly asks for the exact operation:

```text
local.properties
google-services.json
keystore.jks
*.keystore
.env*
secrets.gradle.kts
```
