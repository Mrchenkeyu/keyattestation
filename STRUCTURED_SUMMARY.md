# keyattestation Project Structure Summary

## 1. Root Directory Overview
```
C:\data\code\keyattestation\
├── .git/                      # Git repository
├── .github/                    # GitHub Actions CI/CD
├── .gradle/                    # Gradle cache/state
├── .idea/                      # IntelliJ IDEA project
├── .kotlin/                    # Kotlin compiler cache
├── build.gradle.kts            # Build configuration (Kotlin DSL)
├── settings.gradle.kts         # Gradle settings (rootProject.name = "keyattestation")
├── gradle.properties           # Gradle configuration (configuration-cache=true)
├── gradle/                     # Gradle wrapper
├── gradlew / gradlew.bat       # Gradle wrapper scripts
├── LICENSE                     # Apache License 2.0
├── README.md                   # Project documentation
├── roots.json                  # Mirror of Google root certificates for key attestation
├── build/                      # Build output (not yet built)
└── src/                        # Source code
    ├── main/                   # Main Kotlin source
    │   └── kotlin/
    │       └── com.android.keyattestation.verifier/
    │           ├── Verifier.kt              # Main verifier class
    │           ├── VerificationResult.kt    # Sealed result interface
    │           ├── Extension.kt             # CBOR/ASN.1 parsing, data classes
    │           ├── ConstraintConfig.kt      # Verification constraints
    │           ├── GoogleRevocationList.kt  # Revocation list fetching/parsing
    │           ├── SoftwareRoot.kt          # Software root certificates
    │           ├── provider/                # JCA provider package
    │           │   ├── KeyAttestationProvider.kt  # JCA Provider for CertPathValidator
    │           │   ├── KeyAttestationCertPath.kt  # CertPath representation
    │           │   ├── KeyAttestationCertPathValidator.kt  # Cert path validation SPI
    │           │   └── RevocationChecker.kt     # PKIXRevocationChecker impl
    │           └── challengecheckers/       # Challenge checking implementations
    │               ├── ChallengeChecker.kt    # Interface
    │               ├── ChallengeMatcher.kt    # ByteString challenge matcher
    │               ├── ChainedChallengeChecker.kt  # Chained checkers
    │               └── InMemoryLruCache.kt    # LRU cache for challenge dedup
    └── test/                   # Test source
        └── kotlin/
            └── com.android.keyattestation.verifier/
                ├── VerifierTest.kt
                ├── VerifierCliTest.kt
                └── ... (many test files)
```

## 2. Key Configuration Files

### build.gradle.kts
- **Plugins**: `test-logger`, `kotlin-jvm` (2.2.0)
- **Repositories**: mavenCentral(), google()
- **Dependencies**: 
  - `androidx.annotation:annotation:1.9.1`
  - `co.nstant.in:cbor:0.9` (CBOR parsing)
  - `com.google.code.gson:gson:2.11.0` (JSON parsing for revocation)
  - `com.google.protobuf:protobuf-javalite:4.28.3` & `protobuf-kotlin-lite:4.28.3`
  - `org.bouncycastle:bcpkix-jdk18on:1.78.1` (crypto)
  - `org.jetbrains.kotlin:kotlin-stdlib:2.2.0`, `kotlinx-coroutines-*:1.10.2`
  - `com.google.guava:guava:33.5.0-jre`
- **Java toolchain**: Java 21
- **Custom tasks**: `googleTrustAnchors` (generates `GoogleTrustAnchors.kt` from `roots.json`), `generateSources`, compile ordering

### settings.gradle.kts
```kotlin
plugins { id("org.gradle.toolchains.foojay-resolver-convention") version "0.8.0" }
rootProject.name = "keyattestation"
```

### roots.json
- Contains 2 Google root certificates (PEM-encoded X.509)
- Used by the `googleTrustAnchors` Gradle task to generate `GoogleTrustAnchors.kt`
- Mirrors https://android.googleapis.com/attestation/root

### LICENSE
- Apache License 2.0

## 3. Project Type & Technology Stack

| Category | Details |
|---|---|
| **Project Type** | Kotlin JVM library for Android Key Attestation certificate chain verification |
| **Build System** | Gradle (Kotlin DSL) |
| **Language** | Kotlin 2.2.0, targeting Java 21 |
| **Key Libraries** | - BouncyCastle (ASN.1/Crypto)<br>- Google CBOR (extension parsing)<n>- Gson (revocation JSON)<br>- Protobuf (lite)<br>- Guava (collections, coroutines)<n>- kotlinx-coroutines (async)<n>- AndroidX annotations |
| **API Surface** | ~40 Kotlin source files, main packages:<br>`com.android.keyattestation.verifier`<br>`com.android.keyattestation.verifier.provider`<br>`com.android.keyattestation.verifier.challengecheckers` |
| **Android API Level** | `@RequiresApi(24)` (Nougat MR1) - minimum for key attestation |

## 4. Exposed API / Modules

### Core Verification (`Verifier.kt`)
- `Verifier()` class - main entry point
  - `verify(chain, challengeChecker?, log?)` → `VerificationResult` (blocking)
  - `verifyAsync(coroutineScope, chain, challengeChecker?, log?)` → `ListenableFuture<VerificationResult>`
- `VerificationResult` sealed interface with:
  - `Success(publicKey, challenge, securityLevel, verifiedBootState, deviceLocked, deviceInformation, attestedDeviceIds)`
  - `ChallengeMismatch`, `PathValidationFailure`, `ChainParsingFailure`, `ExtensionParsingFailure`, `ConstraintViolation`, `SoftwareAttestationUnsupported`

### Data Models (`Extension.kt`)
- `KeyDescription` - attestation data parsed from leaf cert extension
  - `attestationVersion`, `attestationSecurityLevel`, `keyMintVersion`, `keyMintSecurityLevel`
  - `attestationChallenge`, `uniqueId`, `softwareEnforced`, `hardwareEnforced`
- `SecurityLevel` enum: `SOFTWARE(0)`, `TRUSTED_ENVIRONMENT(1)`, `STRONG_BOX(2)`
- `Origin` enum, `VerifiedBootState` enum, `VerifiedBootKey`, `deviceLocked`
- `DeviceIdentity` - brand, device, product, serial, IMEIs, MEID, manufacturer, model
- `RootOfTrust` - verifiedBootKey, deviceLocked, verifiedBootState, verifiedBootHash
- `ProvisioningInfoMap` - certificatesIssued
- `AttestationApplicationId`, `AttestationPackageInfo`
- `AuthorizationList`, `PatchLevel`, `InputLimits`

### Challenge Checking (`challengecheckers/`)
- `ChallengeChecker` interface: `checkChallenge(challenge: ByteString): ListenableFuture<Boolean>`
- `ChallengeMatcher` - matches against expected ByteString challenge
- `ChainedChallengeChecker` - compose multiple checkers (halt on first failure)
- `InMemoryLruCache` - dedup cache for challenge results

### JCA Provider (`provider/`)
- `KeyAttestationProvider` - Java Security Provider registration
  - Maps to `KeyAttestation` cert path validator algorithm
  - Returns `KeyAttestationCertPathValidator` instance
- `KeyAttestationCertPathValidator` - `CertPathValidatorSpi` implementation
  - Performs permissive PKIX validation for Android key attestation chains
  - Handles factory/RKP/software provisioning methods
  - Basic name chaining, signature, validity, step expectation checks
- `RevocationChecker` - `PKIXRevocationChecker` impl
  - Checks cert serial numbers against revoked set
- `KeyAttestationCertPath` - `CertPath` subclass
  - Represents full attestation chain with ordering conventions
  - `provisioningMethod()`, `securityLevel()`, `leafCert()`, `attestationCert()`, `intermediateCert()`

### Constraint System (`ConstraintConfig.kt`)
- `ConstraintConfig` - configurable verification constraints
  - `allowSoftwareRoot`, `keyOrigin`, `securityLevel`, `rootOfTrust`, `additionalConstraints`
- `Constraint` sealed interface + `Result` sealed interface
- `SecurityLevelConstraint` - STRICT, NOT_SOFTWARE, CONSISTENT, MATCHES_CERTIFICATE
- `ProvisioningMethodConstraint` - FACTORY, REMOTE
- `TagOrderConstraint` - STRICT tag ordering check
- `ConstraintConfigBuilder` - fluent builder API

### Google Revocation List (`GoogleRevocationList.kt`)
- `getGoogleRevocationStatusFromWeb()` - fetches from `https://android.googleapis.com/attestation/status`
- `getRevocationStatusFromWeb()` - generic URL fetcher
- `parseAttestationStatus()` - Gson-based parsing of revocation status file

### Trust Anchors
- `GoogleTrustAnchors` - generated from `roots.json` by Gradle task
- `SoftwareRoot` - hardcoded software attestation roots for testing

## 5. Git Status
- **Remote**: `git@github.com:Mrchenkeyu/keyattestation.git`
- **Current branch**: `main`
- **Remote tracking**: `origin/main`, `origin/HEAD -> origin/main`
- **Recent commits** (last 10):
  - `a48898a` Make SecurityLevel.STRICT constraint even more strict
  - `fd8c937` Add ProvisioningMethodConstraint
  - `a7d217a` Including a test cert from before Keymaster 4.0
  - `2dd8708` Bump default maxPackages to 32
  - `2b0f4f3` Add bounds checks to AttestationApplicationId
  - `2e3561b` Move function-local revocation status types up a level
  - `a69e6a6` Add workflow to build Maven repo
  - `4d1362e` Remove RequiresApi annotations
  - `47f970d` Support software root for testing.
  - `bf3ccbf` Support software root for testing.

## 6. Integration with film-app-security

### Consumption Pattern
The `film-app-security` project consumes keyattestation as a **thin Maven artifact**:
- **Coordinate**: `com.bx.thirdparty.android:keyattestation:0.0.0-a48898a`
- **Version**: Fixed to commit `a48898a68337b920cbd368eab5824f696d7bbf3d`
- **Packaging**: `jar` (shaded/relocated dependencies per POM)
- **Source**: Built from `C:\data\code\keyattestation` via `maven-install-plugin:install-file`

### Integration Architecture (OfficialKeyAttestationAdapter)
The adapter follows a **wrapper/adaptor pattern** that:

1. **Certificate chain parsing** → `CertificateChainParser` → `List<X509Certificate>`
2. **Official verification** → `AndroidOfficialVerifierClient.verify()` → calls `com.android.keyattestation.verifier.Verifier`
   - Uses `GoogleTrustAnchors` (from roots.json)
   - Uses `ChallengeMatcher` for challenge binding
   - Provides revoked serials snapshot
3. **Result mapping** → `OfficialKeyAttestationAdapter.mapFailure()` maps official `VerificationResult` subtypes to internal `KeyAttestationResult.FailureCode`:
   - `ChainParsingFailure` → `CHAIN_INVALID`
   - `ChallengeMismatch` → `CHALLENGE_MISMATCH`
   - `PathValidationFailure` → `PATH_VALIDATION_FAILED`
   - `ExtensionParsingFailure` → `EXTENSION_INVALID`
   - `ConstraintViolation` → `CONSTRAINT_VIOLATION`
   - `SoftwareAttestationUnsupported` → `SOFTWARE_ATTESTATION_UNSUPPORTED`
   - Unknown → `INTERNAL_VERIFICATION_ERROR`
4. **Application identity extraction** (only on success) → `AttestationApplicationId` from `KeyDescription`
5. **Result assembly** → `KeyAttestationResult.success(VerifiedAttestation)` with:
   - Application packages, SHA-256 signer fingerprints
   - `SecurityLevel.name()`, `VerifiedBootState.name()`, `deviceLocked`
   - Proof public key SHA-256 fingerprint

### Film-App-Security Dependencies
```xml
<dependency>
  <groupId>com.bx.thirdparty.android</groupId>
  <artifactId>keyattestation</artifactId>
  <version>0.0.0-a48898a</version>
</dependency>
```
Also depends on: Spring Boot 4.1.1, Nacos 2.1.0, Spring Data Redis/MongoDB, JUnit 5, Testcontainers.

## 7. How Other Projects Can Reference/Use This Library

### As a Maven/Gradle Dependency
```xml
<!-- Maven -->
<dependency>
  <groupId>com.bx.thirdparty.android</groupId>
  <artifactId>keyattestation</artifactId>
  <version>0.0.0-a48898a</version>
</dependency>
```

```kotlin
// Gradle
implementation("com.bx.thirdparty.android:keyattestation:0.0.0-a48898a")
```

### Building from Source (if needed)
```bash
# From project root
./gradlew build          # Compile + generate GoogleTrustAnchors.kt
./gradlew test           # Run unit tests
```

### Key Usage Pattern (from README)
```kotlin
val verifier = Verifier(
  GoogleTrustAnchors,           // Trust anchors (from roots.json)
  ::getGoogleRevocationStatusFromWeb,  // Revocation source
  { Instant.now() }             // Time source
)

// Verify chain
val result = verifier.verify(certificateChain)

// Handle result
when (result) {
  is VerificationResult.Success -> {
    val publicKey = result.publicKey
    val securityLevel = result.securityLevel
    val verifiedBootState = result.verifiedBootState
    val deviceInformation = result.deviceInformation
  }
  is VerificationResult.ChallengeMismatch -> // handle
  is VerificationResult.PathValidationFailure -> // handle
  // ... other cases
}
```

### Alternative: Custom Challenge Checker
```kotlin
val challengeChecker = ChallengeMatcher(ByteString.copyFromUtf8("challenge123"))
val result = verifier.verify(certificateChain, challengeChecker)
```

### Revocation List Caching Note
- Default: fetches from web on every `verify()` call
- For production: build revocation list into binary or use periodic refresh

### Java 21 / 25 Compatibility
- Library compiled with Java 21 toolchain
- Bytecode compatible with Java 25 runtime (as noted in film-app-security docs)
- Upgrading Java version requires rebuild + compatibility verification