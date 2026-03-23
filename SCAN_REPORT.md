# Revive Truck Inspect — App Scan Report
**Date:** 2026-03-23
**File scanned:** `ReviveTruckInspect(12).html` (7,880 lines)

---

## Summary

| Category | Severity | Count |
|----------|----------|-------|
| Security | HIGH | 3 |
| Security | MEDIUM | 2 |
| Bugs (null refs) | HIGH | 3 |
| Async/Error handling | MEDIUM | 4 |
| Code quality | MEDIUM | 3 |
| Performance | LOW-MEDIUM | 4 |

---

## Security Issues

### 1. Hardcoded Firebase Config (HIGH — standard for Firebase web apps)
**Lines 2135–2142, 5272–5277**

Firebase API key and project identifiers are embedded in the client HTML. This is standard Firebase practice — the API key cannot be kept secret in a web app. However, access must be locked down via **Firestore security rules** to prevent unauthorized reads/writes.

**Action required:** Audit Firebase security rules in the Firebase console.

---

### 2. Sensitive Data in `localStorage` Without Encryption (HIGH)
**Lines 2120–2130, 2239, 2627–2639**

User objects, inspection records, defect data, parts logs, and signature images (base64 JPEG) are all stored in plaintext `localStorage`. Any XSS on the same origin can exfiltrate all of it.

Keys affected:
- `revive_user`
- `revive_inspections`
- `revive_defects`
- `revive_parts_log`
- `revive_users_cache`
- `revive_local_users`

**Recommendation:** Encrypt sensitive fields at rest using a library like TweetNaCl or the Web Crypto API.

---

### 3. XSS Risk — `innerHTML` with Unsanitized Input (HIGH)
**Line 2177:**
```js
ph.innerHTML = 'Open:<br><strong ...>' + shortUrl + '</strong>';
```
`shortUrl` derives from `window.location.href`. If the URL contains unescaped HTML characters, this is an XSS vector.

**Other locations:** The `ic()` SVG helper is injected via `innerHTML` in dozens of places — safe as long as icon names stay hardcoded.

**Recommendation:**
- Sanitize `shortUrl` with `escHtml()` before injecting.
- Add a Content Security Policy (CSP) header in `firebase.json`.

---

### 4. External CDN Resources Without Subresource Integrity (MEDIUM)
**Lines 8, 4862–4864**

```html
<!-- Line 8 — no integrity attribute -->
<link href="https://fonts.googleapis.com/...">
```
```js
// Lines 4862–4864 — dynamic imports, no integrity
import('https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js')
```

**Recommendation:** Add `integrity` + `crossorigin` attributes to static `<link>` and `<script>` tags. (Dynamic ESM imports cannot use SRI today — pin the CDN URL to a specific version, which is already done.)

---

### 5. Silent localStorage Failures (MEDIUM)
**Lines 2185, 2204:**
```js
try { localStorage.setItem('revive_inspections', JSON.stringify(inspections)); } catch(e) {}
```
Storage quota errors are swallowed. The user has no indication that their data wasn't saved.

**Recommendation:** Catch `QuotaExceededError` specifically and show an in-app warning.

---

## Bug / Logic Issues

### 6. `section` / `item` Not Null-Checked Before Use (HIGH)
**Line 4314 — `reopenIssue()`:**
```js
const section = insp.sections.find(s => s.id === sectionId);
const item = section.items.find(...);  // ❌ crashes if section is undefined
item.resolved = false;                 // ❌ crashes if item is undefined
```

**Line 4291 — `resolveIssue()`:**
```js
const item = section.items.find(it => it.id === itemId);
item.resolved = true;  // ❌ crashes if item is undefined
```

**Fix:** Add guard clauses:
```js
if (!section || !item) return;
```

---

### 7. Firebase Registration — `setDoc` Failure Not Surfaced (MEDIUM)
**Lines 2607–2629 — `doRegister()`**

If `setDoc()` fails (network error, security rule rejection), the error is not caught at the call site. The user is still logged in locally with no notification that cloud sync failed.

**Recommendation:** Wrap `setDoc` in a try/catch and show an appropriate message.

---

### 8. Auth State Listener Accumulates on Re-init (MEDIUM)
**Lines 4906–4943**

`onSnapshot` listener is properly tracked in `window._fbUnsub`, but `onAuthStateChanged` itself can register multiple handlers if Firebase is re-initialized, accumulating stale listeners.

**Recommendation:** Track and cancel the `onAuthStateChanged` unsubscribe function too.

---

## Code Quality

### 9. Repeated Firebase Module Access Pattern (MEDIUM)
Appears in 10+ functions:
```js
if (!window._fbDb || !window._fbModules) return;
const { doc, setDoc } = window._fbModules;
```
Extract into a helper `getDb()` that returns `null` when unavailable.

---

### 10. Full List Re-render on Every Update (MEDIUM)
**Line 2920:**
```js
aEl.innerHTML = active.length ? active.map(i => inspectionCardHTML(i)).join('') : ...;
```
The entire inspection list DOM is replaced on every change. With many inspections this causes layout thrashing.

**Recommendation:** Diff the list before re-rendering, or use a keyed update approach.

---

### 11. Triple-Nested Loop on Every Data Load (LOW)
**Lines 2191–2198:**
```js
inspections.forEach(insp =>
  insp.sections && insp.sections.forEach(s =>
    s.items && s.items.forEach(it => { if (it.hasIssue) { ... } })
  )
);
```
Runs on every `loadData()` call. Build an index once on first load.

---

## Recommended Priority Actions

1. **Fix null-ref crashes** in `reopenIssue()` and `resolveIssue()` (lines 4291, 4314).
2. **Sanitize `shortUrl`** before `innerHTML` injection (line 2177).
3. **Handle `QuotaExceededError`** in localStorage save functions.
4. **Audit Firestore security rules** to ensure unauthenticated access is blocked.
5. **Add CSP header** in `firebase.json` to mitigate XSS impact.
6. **Wrap `setDoc` in `doRegister()`** with proper error handling.
