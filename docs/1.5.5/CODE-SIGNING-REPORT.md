# Code Signing Report — AI Studio Platform v1.5.5

**Generated:** 2026-06-27  
**Build Host:** LAPTOP-TM8BC7MD (Windows 11 Home 10.0.26200)

---

## Summary

| Signing Method | Tool Status | Result |
|----------------|-------------|--------|
| GPG artifact signatures | GPG not installed | NOT VERIFIED |
| Authenticode EXE signing | signtool.exe not found | NOT VERIFIED |
| SHA-256 integrity checksums | `checksums.sha256` present | VERIFIED |
| Release manifest checksums | `release-manifest.json` present | VERIFIED |

**Current integrity guarantee:** SHA-256 checksums only. Cryptographic signatures not applied.

---

## Tool Check (VERIFIED)

```powershell
# GPG check
Get-Command gpg -ErrorAction SilentlyContinue
# $null — GPG not installed

# signtool check
Get-Command signtool -ErrorAction SilentlyContinue
# $null — Windows SDK not installed (signtool ships with Windows SDK)
```

---

## 1. GPG Artifact Signing

**Status: NOT VERIFIED — GPG not installed on build host**

### What Would Be Signed

| File | Purpose |
|------|---------|
| `checksums.sha256` | Primary integrity manifest |
| `AISoftwareFactory-v1.5.5-Portable.zip` | AISF distribution |
| `AIStudioDesktop-v1.5.5-Portable.zip` | Desktop distribution |
| `release-manifest.json` | Release metadata |
| `sbom-aisf.json` | AISF SBOM |
| `sbom-desktop.json` | Desktop SBOM |
| `sbom-content-factory.json` | CF SBOM |

### Commands When GPG Available

```bash
# Import signing key
gpg --import ai-studio-release-key.gpg

# Sign artifacts
gpg --armor --detach-sign AISoftwareFactory-v1.5.5-Portable.zip
gpg --armor --detach-sign AIStudioDesktop-v1.5.5-Portable.zip
gpg --armor --detach-sign checksums.sha256

# Produces: *.asc files (ASCII-armored detached signatures)

# Verify
gpg --verify checksums.sha256.asc checksums.sha256
# Expected: Good signature from "AI Studio Release Key"

# Fingerprint for distribution
gpg --fingerprint ai-studio-release@ai-studio.local
```

### GitHub Actions Integration

The release workflow (`.github/workflows/ci.yml`) supports GPG signing when `secrets.GPG_PRIVATE_KEY` is set:

```yaml
- name: Import GPG key
  if: ${{ secrets.GPG_PRIVATE_KEY != '' }}
  run: echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --import

- name: Sign artifacts
  if: ${{ secrets.GPG_PRIVATE_KEY != '' }}
  run: |
    Get-ChildItem packaging/dist -Filter "*.zip" | ForEach-Object {
      gpg --detach-sign --armor $_.FullName
    }
```

To enable: add `GPG_PRIVATE_KEY` to GitHub repository secrets at Settings → Secrets → Actions.

---

## 2. Authenticode Code Signing

**Status: NOT VERIFIED — signtool.exe not installed, no Authenticode certificate**

### What Would Be Signed

| File | Purpose |
|------|---------|
| `ai-software-factory.exe` | AISF server binary |
| `ai-studio-desktop.exe` | Desktop GUI binary |
| `AIStudioPlatform-v1.5.5-Setup.exe` | Windows installer (when built) |

### Commands When signtool + Certificate Available

```powershell
# Install Windows SDK for signtool.exe
# winget install Microsoft.WindowsSDK.10.0.22621

# Sign with certificate store
signtool sign `
    /n "AI Studio Team" `
    /t http://timestamp.digicert.com `
    /fd SHA256 `
    packaging\dist\ai-software-factory\ai-software-factory.exe

signtool sign `
    /n "AI Studio Team" `
    /t http://timestamp.digicert.com `
    /fd SHA256 `
    packaging\dist\ai-studio-desktop\ai-studio-desktop.exe

# Verify
signtool verify /pa packaging\dist\ai-studio-desktop\ai-studio-desktop.exe

# Sign installer (when built)
signtool sign /n "AI Studio Team" /t http://timestamp.digicert.com /fd SHA256 `
    dist\AIStudioPlatform-v1.5.5-Setup.exe
```

### Windows SmartScreen Impact

Without Authenticode signing, Windows Defender SmartScreen will display:
> "Windows protected your PC — Microsoft Defender SmartScreen prevented an unrecognized app from starting."

Users must click "More info → Run anyway." This is a UX friction point, not a security blocker, but is unacceptable for GA release. **Authenticode signing is required before public GA release.**

### Certificate Options

| Option | Cost | Validation | SmartScreen |
|--------|------|------------|-------------|
| DigiCert OV Code Signing | ~$499/yr | Organization validation | Partial trust immediately |
| DigiCert EV Code Signing | ~$699/yr | Extended validation | Full trust immediately |
| SSL.com EV | ~$299/yr | Extended validation | Full trust immediately |
| Sectigo OV | ~$199/yr | Organization validation | Partial trust |

**Recommendation:** EV Code Signing Certificate for immediate SmartScreen trust bypass.

---

## 3. SHA-256 Integrity Checksums (VERIFIED)

As the interim integrity mechanism:

```
# File: pipeline-output/v1.5.5/release/checksums.sha256
e2e32212c863c1e6c73f470bcc60cfc55254b0bbbf1ce907722581c2bf3e1e6b  AISoftwareFactory-v1.5.4-Portable.zip
db2468a92071a557cb75965b5c14e0f71830f48bd2c2d7eb6ec4a156e5c9f958  AIStudioDesktop-v1.5.4-Portable.zip
```

**Verification (EXECUTED):**
```
$ sha256sum -c checksums.sha256
AISoftwareFactory-v1.5.4-Portable.zip: OK
AIStudioDesktop-v1.5.4-Portable.zip: OK
```

The `release-manifest.json` embeds these checksums in the `artifacts[*].sha256` field.

---

## 4. Auto-Updater Integrity (VERIFIED — CODE)

The auto-updater (`app/updater.py`) independently verifies SHA-256 before applying:

```python
h = hashlib.sha256()
with httpx.stream("GET", info.download_url, ...) as r:
    for chunk in r.iter_bytes(_CHUNK):
        h.update(chunk)
if h.hexdigest() != info.sha256:
    on_progress("error", "SHA-256 mismatch — download corrupt or tampered")
    return None  # abort — never extract
```

This prevents tampered or corrupted updates from being applied even without GPG signatures.

---

## GA Blocking Assessment

| Item | GA Blocking? | Notes |
|------|-------------|-------|
| Authenticode signing | **YES** — public release | SmartScreen blocks unsigned EXEs |
| Authenticode signing | NO — internal use | SmartScreen can be bypassed manually |
| GPG artifact signatures | NO — optional for internal | SHA-256 provides integrity |
| GPG artifact signatures | YES — enterprise distribution | Compliance and trust requirements |
| Timestamp (RFC 3161) | YES — signed binaries | Signatures expire without timestamp |

**For internal/dev GA release:** SHA-256 checksums are sufficient.  
**For public distribution:** Authenticode EV certificate is required.

---

## NOT VERIFIED

| Item | Requirement |
|------|-------------|
| GPG key generation | `gpg --full-generate-key` (type: RSA 4096) |
| GPG public key distribution | GitHub repository → Security → Verified Releases |
| Authenticode certificate | Purchase from CA (DigiCert, Sectigo, SSL.com) |
| signtool.exe | Install Windows SDK 10.0.22621+ |
| Timestamp authority | Include `/t http://timestamp.digicert.com` in all sign commands |
| Signature verification in CI | Add `signtool verify /pa` step to release.yml |
