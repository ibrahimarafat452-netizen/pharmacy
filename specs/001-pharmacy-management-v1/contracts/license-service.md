# Contract: License Service

**Module**: Licensing & Activation (Phase 11)
**Plan**: [../plan.md](../plan.md) | **Data Model**: [../data-model.md](../data-model.md)

---

## ILicenseService

Verifies license at application startup. No permission check — runs before authentication.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| VerifyLicense | — | LicenseResult | Reads license file, verifies signature, checks hardware fingerprint match. Returns Valid/Invalid/Missing/HardwareMismatch |
| GetLicenseInfo | — | LicenseInfo? | Returns license details (pharmacy name, issued date, expiry) if valid |
| GenerateHardwareFingerprint | — | string | Collects hardware identifiers and generates fingerprint hash. For display during reactivation requests |

**LicenseResult**: status (Valid, Invalid, Missing, Expired, HardwareMismatch), message

**Startup flow**:
1. Application calls `VerifyLicense()`
2. If Valid → proceed to login screen
3. If Missing/Invalid/Expired/HardwareMismatch → show activation screen with error message and hardware fingerprint for reactivation contact

**Security notes**:
- Vendor's RSA public key embedded in application as a constant
- License file location: application directory or configurable path
- Hardware fingerprint formula determined by Prototype P-10.3 results
- No internet dependency (FR-076)

---

## ILicenseGenerator (Vendor-Side Tool — Not Part of Main Application)

Separate command-line tool used by the vendor to generate license files.

| Method | Input | Output | Notes |
|--------|-------|--------|-------|
| GenerateLicense | pharmacyId, hardwareFingerprint, expiresAt? | signedLicenseFile | Signs with vendor's private key |

This is a separate utility, not deployed to pharmacies.
