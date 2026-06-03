# Fix: doc-it rule-violation-checker noisy warnings for missing source directories

## Problem

When `rule-violation-checker` runs with default `sourceDirs: "src,app"` on a project that doesn't have a top-level `app/` directory (e.g., shishaKe where app code lives at `src/app/`), it logs noisy warnings:

```
doc-it: error scanning /path/to/project/app for forbidden patterns
doc-it: error scanning /path/to/project/app for routes
```

These come from `scanDirForPatterns()` and `scanDirForRoutes()` in `doc-it-core.js`. The `catch {}` blocks catch **all** errors (including `ENOENT` from missing directories) and log a warning. They should silently skip non-existent directories — other errors (permission denied, etc.) should still warn.

## Existing Logic Analysis

### Affected execution path

1. `rule-violation-checker` is called with `sourceDirs` defaulting to `"src,app"`
2. `checkRuleViolations()` iterates each dir → calls `scanDirForPatterns(base + "/" + dir, ...)` and `scanDirForRoutes(base + "/" + dir, ...)`
3. Both functions call `readdir(dir)` which throws `ENOENT` if dir doesn't exist
4. Catch block logs `console.warn("doc-it: error scanning " + dir + " for ...")`

### Code locations (all in `doc-it` installer script)

| Instance | Function | Lines |
|----------|----------|-------|
| Opencode `doc-it-core.js` | `scanDirForPatterns` | 534–550 |
| Opencode `doc-it-core.js` | `scanDirForRoutes` | 553–572 |
| Kilo `dist/doc-it-core.js` | `scanDirForPatterns` | 964–980 |
| Kilo `dist/doc-it-core.js` | `scanDirForRoutes` | 983–1002 |

Both copies are embedded inline in the `doc-it` install script and are functionally identical.

### Hidden coupling / shared utilities

- Both functions use `readdir`, `stat`, `readFile` from `node:fs/promises` — already imported at top of each module.
- The `checkRuleViolations` function in both copies calls these scan functions identically.
- No other code depends on the warning message being present (it's purely informational).
- The forbidden patterns array (`db.*.create|update|delete|upsert` etc.) and route regex are unaffected.

## Fix Strategy

### Minimal change

Change the catch block in all 4 function instances from:

```javascript
} catch { console.warn("doc-it: error scanning " + dir + " for forbidden patterns") }
```

to:

```javascript
} catch (err) {
  if (err.code !== 'ENOENT') console.warn("doc-it: error scanning " + dir + " for forbidden patterns")
}
```

And similarly for routes.

**Why `ENOENT`?** That's the `err.code` from `readdir` / `stat` / file operations when the path does not exist. This preserves warnings for real errors (permission denied, disk I/O failures, etc.) while silently skipping missing directories.

### Why not implicitly skip at the call site?

The alternative would be to check existence before calling:
```javascript
try { await stat(dir) } catch { continue }
```
But that adds an extra `stat` syscall for every directory that does exist. The catch-filter approach is zero-overhead for the happy path and only activates on error.

## Potential Conflicts

- **None.** The change only affects error handling behavior in catch blocks. No data structures, return values, or function signatures change.
- The `forEach` loops in `checkRuleViolations` continue unaffected — missing dirs just contribute zero violations.

## Backward Compatibility

| Before | After | Impact |
|--------|-------|--------|
| `readdir("/nonexistent")` → logs warning | `readdir("/nonexistent")` → silent skip | **Compatible**: less noise |
| `readdir("/restricted")` (permission denied) → logs warning | Same → still logs warning | **Identical** behavior |
| `readdir("/valid")` → scans normally | Same → scans normally | **Identical** behavior |
| Any hypothetical code parsing the warning text | No longer gets warning for missing dirs | **Low risk**: warning was not part of any returned data |

## Migration/Rollout Strategy

1. Apply edits to the 4 catch blocks in `doc-it` installer script
2. Re-run `install-doc-it` to update installed `doc-it-core.js` files (or deploy via the installer)
3. Verify the `rule-violation-checker` runs cleanly on the shishaKe project

**Rollback:** Revert the 4 catch blocks and re-install. Or manually restore `~/.config/opencode/tools/doc-it-core.js` / Kilo plugin copy from backup.

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| `err` binding missing in some JS runtimes | Very low (Node 14+) | Catch block would throw ReferenceError | All target environments (opencode, Kilo) use Node 18+ |
| Other error codes mistakenly silenced | None — we only filter `ENOENT` | N/A | The check is explicit |
| Warning was being parsed by some monitoring tool | Low | Tool would stop seeing warnings for missing dirs | Warning was unstructured text, not a data contract |

## Regression Prevention

- All existing functionality is preserved — the scan functions still correctly:
  - Recurse into subdirectories
  - Detect forbidden patterns and report violations
  - Detect undocumented routes and report violations
- The only behavioral change is that `ENOENT` on the initial directory is ignored instead of printing a warning.

## Testing Strategy

1. **Manual verification:** Run `rule-violation-checker` on the shishaKe project (which has no top-level `app/`) and confirm zero warnings about `error scanning ... for ...`
2. **Sanity:** Run on a project that has both `src/` and `app/` and confirm all violations are still detected
3. **Edge:** Run with a valid `app/` directory that has no `.ts`/`.js` files — should scan cleanly with zero warnings (unchanged behavior)
4. **Edge:** Run with `sourceDirs: "nonexistent1,nonexistent2"` — should produce zero violations, zero warnings (previously would warn for both)

---

## Progress Log

- [x] Analyze problem: traced to `try/catch` in `scanDirForPatterns` and `scanDirForRoutes` — blanket catch logs warnings for ALL errors including `ENOENT`
- [x] Plan written and approved
- [x] Fix applied to all 4 catch blocks (2 functions × 2 copies: Opencode + Kilo) using `replaceAll`
  - Lines 550-551: `scanDirForPatterns` (opencode copy) — changed `catch {}` to `catch (err) { if (err.code !== 'ENOENT') ... }`
  - Lines 574-575: `scanDirForRoutes` (opencode copy) — same change
  - Lines 984-985: `scanDirForPatterns` (kilo copy) — same change
  - Lines 1008-1009: `scanDirForRoutes` (kilo copy) — same change
- [x] Verified: `rg -n "ENOENT" doc-it` shows all 4 guards in place
- [x] Tool still needs re-installation in target projects for the fix to take effect
