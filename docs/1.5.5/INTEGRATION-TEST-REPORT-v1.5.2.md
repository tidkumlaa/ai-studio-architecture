# AI Studio Platform v1.5.2 — Integration Test Report
**Date:** 2026-06-27 | **Execution time:** ~4 seconds

---

## Test Execution Summary

| Suite | File | Tests | Passed | Failed | Time |
|---|---|---|---|---|---|
| AISF Core Fixes | `tests/test_v152_aisf_core.py` | 10 | 10 | 0 | 0.18s |
| AISF Proxy Routes | `tests/test_v152_proxy.py` | 8 | 8 | 0 | 0.14s |
| Installer & Release | `tests/test_v152_installer.py` | 16 | 16 | 0 | 0.34s |
| Desktop v1.5.2 | `tests/test_v152_desktop.py` | 18 | 18 | 0 | 0.22s |
| E2E Integration | `tests/test_integration_e2e.py` | 109 | 109 | 0 | 2.35s |
| **TOTAL** | | **161** | **161** | **0** | **~4s** |

**All 161 tests pass. Zero failures. Zero errors.**

---

## Test Detail: AISF Core Fixes (`test_v152_aisf_core.py`)

| Test | Covers | Result |
|---|---|---|
| `test_verify_api_key_allows_when_key_empty` | Dev mode — no auth when key not configured | PASSED |
| `test_verify_api_key_rejects_wrong_key` | Auth — 401 on wrong key | PASSED |
| `test_verify_api_key_allows_correct_key` | Auth — 200 on correct key | PASSED |
| `test_restore_backup_rejects_invalid_id` | Input sanitization — rejects `../../../etc/passwd` | PASSED |
| `test_restore_backup_accepts_valid_id` | Input sanitization — accepts `backup-2026-01-01` | PASSED |
| `test_capabilities_has_resume_and_update` | Capabilities schema — new keys present | PASSED |
| `test_capabilities_content_flags_true_when_cf_alive` | Dynamic content flags — True when CF reachable | PASSED |
| `test_capabilities_content_flags_false_when_cf_down` | Dynamic content flags — False when CF offline | PASSED |
| `test_platform_version_is_152` | Version sync — platform_version.py | PASSED |
| `test_pyproject_toml_version_is_152` | Version sync — AISF pyproject.toml | PASSED |

---

## Test Detail: AISF Proxy Routes (`test_v152_proxy.py`)

| Test | Covers | Result |
|---|---|---|
| `test_get_episodes_503_when_cf_offline` | Proxy — 503 on CF connection failure | PASSED |
| `test_get_episodes_200_when_cf_online` | Proxy — passes CF 200 response through | PASSED |
| `test_queue_stats_maps_to_health` | Path mapping — /queue/stats → CF /health | PASSED |
| `test_post_episode_run_maps_to_retry` | Path mapping — /episodes/{id}/run → CF retry | PASSED |
| `test_get_episode_timeline_maps_to_dag` | Path mapping — /episodes/{id}/timeline → CF dag | PASSED |
| `test_inventory_channels_maps_to_publish_channels` | Path mapping — /inventory/channels → CF channels | PASSED |
| `test_queue_priority_returns_stub` | Stub — /queue/{id}/priority returns {"ok": true} | PASSED |
| `test_proxy_router_registered_in_app` | Registration — proxy_router in app.routes | PASSED |

---

## Test Detail: Installer & Release (`test_v152_installer.py`)

| Test | Covers | Result |
|---|---|---|
| `TestDownloadVerifier::test_placeholder_bypass_removed` | SHA256 bypass removed — placeholder now fails | PASSED |
| `TestDownloadVerifier::test_correct_sha256_passes` | SHA256 — correct hash accepted | PASSED |
| `TestDownloadVerifier::test_correct_sha256_with_prefix_passes` | SHA256 — "sha256:" prefix format | PASSED |
| `TestDownloadVerifier::test_wrong_sha256_fails` | SHA256 — wrong hash rejected | PASSED |
| `TestDownloadVerifier::test_empty_checksum_returns_true_with_warning` | SHA256 — empty = warning + allow | PASSED |
| `TestDownloadVerifier::test_none_checksum_returns_true` | SHA256 — None = allow | PASSED |
| `TestDownloadVerifier::test_missing_file_returns_false` | SHA256 — missing file = reject | PASSED |
| `TestBootstrapManagerChecksum::test_placeholder_bypass_removed` | Bootstrap SHA256 bypass removed | PASSED |
| `TestBootstrapManagerChecksum::test_correct_sha256_passes` | Bootstrap SHA256 — correct hash | PASSED |
| `TestBootstrapManagerChecksum::test_wrong_sha256_fails` | Bootstrap SHA256 — wrong hash rejected | PASSED |
| `TestBootstrapManagerChecksum::test_empty_checksum_returns_true_with_warning` | Bootstrap SHA256 — empty = warning | PASSED |
| `TestBootstrapManagerChecksum::test_missing_file_returns_false` | Bootstrap SHA256 — missing file | PASSED |
| `TestStepSign::test_no_dist_returns_ok_false` | Signing — no dist/ → ok=False | PASSED |
| `TestStepSign::test_writes_checksums_sha256` | Signing — CHECKSUMS.sha256 written | PASSED |
| `TestStepSign::test_gpg_missing_handled_gracefully` | Signing — GPG absent → graceful | PASSED |
| `TestStepSign::test_ok_true_even_when_gpg_unavailable` | Signing — ok=True without GPG | PASSED |

---

## Test Detail: Desktop v1.5.2 (`test_v152_desktop.py`)

| Test | Covers | Result |
|---|---|---|
| `TestDesktopConfig::test_no_hardcoded_path` | Config — no `E:/UserData` in yaml | PASSED |
| `TestDesktopConfig::test_api_key_field_present` | Config — api_key field exists | PASSED |
| `TestDesktopConfig::test_aisf_home_is_empty` | Config — aisf_home is empty string | PASSED |
| `TestDesktopConfig::test_api_key_default_empty` | Config — api_key default is empty | PASSED |
| `TestApiClientApiKey::test_sends_xapikey_header_when_set` | Auth header — present when key set | PASSED |
| `TestApiClientApiKey::test_no_xapikey_header_when_empty` | Auth header — absent when empty | PASSED |
| `TestApiClientApiKey::test_api_key_stored_on_init` | ApiClient — key stored on __init__ | PASSED |
| `TestApiClientApiKey::test_set_api_key_updates_key` | ApiClient — set_api_key() updates | PASSED |
| `TestApiClientApiKey::test_set_api_key_when_closed_does_not_open` | ApiClient — no premature open | PASSED |
| `TestPlatformApiClientApiKey::test_accepts_api_key_param` | PlatformApiClient — accepts key | PASSED |
| `TestPlatformApiClientApiKey::test_set_api_key_propagates_to_inner_api` | PlatformApiClient — propagates | PASSED |
| `TestApplyCapabilitiesResume::test_resume_uses_resume_key_not_restart` | Capability — resume uses right key | PASSED |
| `TestApplyCapabilitiesResume::test_resume_key_true_enables_button` | Capability — resume enabled when true | PASSED |
| `TestPyprojectVersion::test_version_is_152` | Version — Desktop 1.5.2 | PASSED |
| `TestPyprojectVersion::test_old_version_removed` | Version — 1.0.0 gone | PASSED |
| `TestNotificationManagerQuitApp::test_has_quit_app_method` | SysTray — _quit_app exists | PASSED |
| `TestNotificationManagerQuitApp::test_quit_app_is_callable` | SysTray — callable | PASSED |
| `TestNotificationManagerQuitApp::test_quit_app_calls_qapplication_quit` | SysTray — calls QApplication.quit | PASSED |

---

## End-to-End Integration Suite (`test_integration_e2e.py`)

109 tests covering the full Desktop↔AISF↔CF integration chain. No regressions introduced.

| Test class | Count | Focus |
|---|---|---|
| `TestDesktopStartup` | 10 | Bootstrap, config, PID, crash hooks |
| `TestAisfStartup` | 14 | AISF module imports, all routers registered |
| `TestCapabilityDiscovery` | 9 | Capability response schema and values |
| `TestRuntimeApiShape` | 11 | All runtime routes registered with correct signatures |
| `TestHealthApiShape` | 7 | Health/ping/metrics endpoints and response fields |
| `TestContentFactoryIntegration` | 14 | ContentFactoryPlugin methods and error handling |
| `TestFailureRecovery` | 21 | Offline detection, backoff, auto-recovery |
| `TestShutdown` | 8 | PID management, clean teardown |
| `TestRestart` | 15 | AisfChecker start/stop/wait contract |

---

## Warnings (non-blocking)

| Warning | Source | Action |
|---|---|---|
| `PendingDeprecationWarning: import python_multipart` | starlette upstream | No action needed — upstream issue |
| `PytestConfigWarning: Unknown config option: env` | pyproject.toml missing `pytest-dotenv` | No action needed — env key is harmless |
| `PytestRemovedIn10Warning: class-scoped fixture as instance method` | test_integration_e2e.py | Fix before upgrading to pytest 10 |

---

## NOT VERIFIED (requires live service execution)

These items cannot be verified by the static + mock test suite. They require live AISF, CF, and Ollama processes:

| Item | Why not verified |
|---|---|
| End-to-end episode creation flow | Requires running CF Java service |
| AISF→CF proxy with real HTTP traffic | Requires running CF on :8090 |
| GPG signing in CI | Requires GPG key installation |
| Worker supervisor with 10 live threads | Requires full AISF startup |
| Desktop Qt rendering | Requires PySide6 display environment |
| Ollama inference calls | Requires Ollama service |
