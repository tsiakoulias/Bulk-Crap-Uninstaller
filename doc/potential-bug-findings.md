# Potential Fixable Bug Findings

This is a living document for collecting likely, fixable bugs found in the local `Bulk-Crap-Uninstaller` codebase.
It intentionally mixes:

- issues that already appear to be reported upstream, and
- additional code-audit findings that do not appear to have a matching upstream issue yet.

As of this pass, the document captures 10 candidates that look actionable and worth validating with repro steps or targeted tests.

## Summary

| ID | Title | Source | Upstream issue | Confidence | Primary area |
| --- | --- | --- | --- | --- | --- |
| 1 | Registry key copy can crash on a missing or inaccessible source key | Local audit | None found | High | Registry tools |
| 2 | Startup and browser-helper scans can create registry keys while only reading | Local audit | None found | High | Startup scanning |
| 3 | Registry access failures can falsely mark an entry as already uninstalled | Local audit | None found | High | Uninstall workflow |
| 4 | Foreground worker threads can hang startup or uninstall flows indefinitely | Upstream-backed | [#551](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/551), [#793](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/793) | High | Background work / threading |
| 5 | Concurrent factory failures can be silently converted into an empty result set | Local audit | None found | High | Factory orchestration |
| 6 | Unrealistic registry `EstimatedSize` values are trusted as-is | Upstream-backed | [#792](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/792) | High | Size detection |
| 7 | Install date detection uses unreliable heuristics and weak fallbacks | Upstream-backed | [#784](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/784) | High | Date detection |
| 8 | Auto-loaded uninstall lists can surface XML errors during startup | Upstream-backed | [#775](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/775) | Medium-High | Startup / list loading |
| 9 | Checked-item state can disagree with the uninstall selection that is actually executed | Upstream-backed | [#781](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/781) | Medium | List selection / filtering |
| 10 | Only the current HKCU uninstall roots are scanned, so user-context changes can hide apps | Upstream-backed | [#683](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/683) | Medium | Registry enumeration |

---

## 1. Registry key copy can crash on a missing or inaccessible source key

- Source: Local audit only
- Primary files:
  - `source\KlocTools\Tools\RegistryTools.cs`
- Relevant code:
  - `CopySubKey(...)`
  - `RecurseCopyKey(...)`

### Why this looks buggy

`CopySubKey(...)` opens the source key with `parentKey.OpenSubKey(subKeyName, true)` and immediately passes the result into `RecurseCopyKey(...)`.
If the source key is missing or access is denied, `OpenSubKey(...)` can return `null`.
`RecurseCopyKey(...)` then dereferences the key with `GetValueNames()` and `GetSubKeyNames()` without a null guard.

### User impact

Any feature that tries to rename, move, or copy registry-backed data can fail with a null-reference crash instead of producing a useful error.
This is especially likely on partially deleted keys or protected registry paths.

### Potential fix direction

- Guard `sourceKey` and `sourceSubKey` before recursion.
- Decide whether the desired behavior is:
  - throw a specific exception,
  - skip inaccessible subkeys, or
  - return a failure result.
- Avoid opening the source key as writable unless write access is actually required.

---

## 2. Startup and browser-helper scans can create registry keys while only reading

- Source: Local audit only
- Primary files:
  - `source\UninstallTools\Startup\Browser\BrowserEntryFactory.cs`
  - `source\UninstallTools\Startup\Normal\StartupEntryFactory.cs`
  - `source\KlocTools\Tools\RegistryTools.cs`

### Why this looks buggy

`BrowserEntryFactory.GetBrowserHelpers()` uses `RegistryTools.CreateSubKeyRecursively(...)` to access browser-helper startup points.
That helper does not just open keys; it creates missing keys on the way down.
This means a read-only scan can mutate the registry.

`StartupEntryFactory` has a similar side effect: if opening a startup point throws an `ArgumentException`, it creates the key in the catch path.

### User impact

Merely opening BCU or refreshing startup-related data can leave behind newly created registry keys that did not exist before.
That is surprising behavior for a scanner and can make troubleshooting harder.

### Potential fix direction

- Replace create-on-read paths with safe open-only helpers.
- Only create keys inside explicit write operations such as enable/disable or edit flows.
- Keep scan code side-effect free.

---

## 3. Registry access failures can falsely mark an entry as already uninstalled

- Source: Local audit only
- Primary files:
  - `source\UninstallTools\ApplicationUninstallerEntry.cs`
  - `source\UninstallTools\Uninstaller\BulkUninstallEntry.cs`

### Why this looks buggy

`ApplicationUninstallerEntry.RegKeyStillExists()` catches all exceptions and returns `false`.
`BulkUninstallEntry.RunUninstaller(...)` treats `!RegKeyStillExists()` as evidence that the entry is already gone and marks the uninstall as completed without running anything.

This collapses at least two very different states into one:

- the registry key is truly gone, and
- the registry key exists but could not be opened because of permissions or transient registry failures.

### User impact

An app can be shown as successfully handled even though the uninstaller was never launched.
This is the kind of bug that can leave users with stale entries and misleading status.

### Potential fix direction

- Distinguish "not found" from "failed to inspect".
- Only return `false` for verified key absence.
- Surface access-denied or registry-read failures as explicit errors or warnings.

---

## 4. Foreground worker threads can hang startup or uninstall flows indefinitely

- Source: Upstream-backed
- Upstream references:
  - [Issue #551 - Forever loading on startup](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/551)
  - [Issue #793 - Program freezes when deleting any application](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/793)
- Primary files:
  - `source\UninstallTools\ThreadedWorkSpreader.cs`
  - `source\UninstallTools\Factory\ApplicationUninstallerFactory.cs`

### Why this looks buggy

`ThreadedWorkSpreader` creates worker threads with `IsBackground = false`.
Its `Join()` method then blocks on every worker thread with no timeout and no cancellation escape hatch.

If any worker gets stuck on slow registry access, disk access, a network-backed location, or another blocking operation, the caller can wait forever.

### User impact

This is a strong fit for startup hangs and delete/uninstall freezes reported by users.
Even one wedged worker can stall the whole operation.

### Potential fix direction

- Use background workers or task-based async orchestration.
- Add cancellation and bounded waits.
- Log which subsystem is still running when a timeout is exceeded.

---

## 5. Concurrent factory failures can be silently converted into an empty result set

- Source: Local audit only
- Primary files:
  - `source\UninstallTools\Factory\ConcurrentApplicationFactory.cs`
  - `source\UninstallTools\Factory\ApplicationUninstallerFactory.cs`

### Why this looks buggy

`ConcurrentApplicationFactory` only catches `OperationCanceledException` inside its worker thread.
If the worker throws a different exception, `_threadResults` never gets assigned.
Later, `GetResults(...)` returns `_threadResults ?? new List<ApplicationUninstallerEntry>()`.

That fallback hides the failure and makes it look like there simply were no results from those factories.

### User impact

Store apps, Scoop apps, or other factory-backed results can disappear silently instead of surfacing a clear failure.
This can be hard to diagnose because the UI still receives a valid empty collection.

### Potential fix direction

- Capture non-cancellation exceptions from the worker thread.
- Re-throw them on `GetResults(...)` or attach them to the progress/error channel.
- Avoid treating "thread crashed" and "factory returned an empty list" as the same outcome.

---

## 6. Unrealistic registry `EstimatedSize` values are trusted as-is

- Source: Upstream-backed
- Upstream reference:
  - [Issue #792 - File size wrong](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/792)
- Primary files:
  - `source\UninstallTools\Factory\RegistryFactory.cs`
  - `source\UninstallTools\Factory\InfoAdders\FastSizeGenerator.cs`

### Why this looks buggy

`RegistryFactory.GetEstimatedSize(...)` reads the registry `EstimatedSize` value and converts it directly to a `FileSize` in kilobytes.
There is no sanity check for obviously broken values.

That matters because some installers populate `EstimatedSize` incorrectly, and BCU currently appears willing to trust those values even when they are absurd.
`FastSizeGenerator` is another candidate contributor because it parses external tool output very optimistically, but the registry path is the first thing to verify for wildly inflated numbers.

### User impact

Users can see tiny applications reported as hundreds of gigabytes in size.
That undermines sort order, filtering, and user trust in the list.

### Potential fix direction

- Reject impossible or highly suspicious registry sizes.
- Prefer recomputation when the registry value is outside a reasonable bound.
- Consider annotating whether a size came from the registry or from a computed scan.

---

## 7. Install date detection uses unreliable heuristics and weak fallbacks

- Source: Upstream-backed
- Upstream reference:
  - [Issue #784 - Install date questionable](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/784)
- Primary files:
  - `source\UninstallTools\Factory\RegistryFactory.cs`
  - `source\UninstallTools\Factory\InfoAdders\InstallDateAdder.cs`
  - `source\UninstallTools\Factory\InfoAdders\MsiInfoAdder.cs`

### Why this looks buggy

`RegistryFactory.GetInstallDate(...)` assumes an 8-character registry value is `YYYYMMDD`, then falls back to a guessed `YYYYDDMM` interpretation.
If the registry does not help, `InstallDateAdder` falls back to file or directory creation timestamps.

Both approaches are shaky:

- registry values can be malformed or vendor-specific,
- creation timestamps are not reliable proxies for installation time,
- the code does not apply much validation to the resulting date.

### User impact

Sorting by install date can place recent installs far from the top of the list.
This lines up well with reports that recently installed apps appear to have been installed on the previous day or much earlier.

### Potential fix direction

- Tighten date validation before accepting registry-derived values.
- Prefer authoritative MSI/app-store metadata when available.
- Treat file-system creation time as a low-confidence fallback rather than a normal source of truth.

---

## 8. Auto-loaded uninstall lists can surface XML errors during startup

- Source: Upstream-backed
- Upstream reference:
  - [Issue #775 - First launch of an app produces an error](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/775)
- Primary files:
  - `source\BulkCrapUninstaller\Forms\Windows\MainWindow.cs`
  - `source\BulkCrapUninstaller\Controls\Settings\AdvancedFilters.cs`
  - `source\UninstallTools\Lists\UninstallList.cs`

### Why this looks buggy

On first refresh, `MainWindow` will try to auto-load:

- the first command-line argument as an uninstall list, and then
- `Default.bcul` from the application directory when auto-load is enabled.

`AdvancedFilters.LoadUninstallList(...)` deserializes the file with `XmlSerializer` and shows the exact "this is not an uninstall list" dialog when deserialization fails.
That is a plausible match for users seeing an XML error at startup even before they intentionally interact with uninstall lists.

### User impact

A stray or invalid `.bcul` file in the application folder, or an unexpected command-line argument, can generate a confusing XML error on launch.
Because this happens during startup, users can interpret it as a general application corruption issue.

### Potential fix direction

- Be stricter about when auto-load is attempted.
- Suppress or soften the error during startup and attach more context to it.
- Validate the target file before deserializing, or quarantine known-bad default lists.

---

## 9. Checked-item state can disagree with the uninstall selection that is actually executed

- Source: Upstream-backed
- Upstream reference:
  - [Issue #781 - Filtered for publisher, selected program but BCU says no uninstallers selected](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/781)
- Primary files:
  - `source\BulkCrapUninstaller\Functions\ApplicationList\UninstallerListViewUpdater.cs`
  - `source\BulkCrapUninstaller\Forms\Windows\MainWindow.cs`
  - `source\BulkCrapUninstaller\Forms\Wizards\BeginUninstallTaskWizard.cs`

### Why this looks buggy

The selection model has at least two different representations:

- `SelectedUninstallerCount` uses `_listView.CheckedObjects.Count` when checkboxes are enabled.
- `SelectedUninstallers` uses `GetAllObjectsWithMappedCheckState(CheckState.Checked)` and then filters the result with `AllUninstallers.Contains(...)`.

That mismatch creates room for "I can see something selected" versus "the command path thinks nothing is selected," especially after filtering or view-state changes.

### User impact

The user can check an item and still get the "No uninstallers were selected" message when moving into the uninstall flow.
The wizard path confirms this can happen because it explicitly bails out when the resulting task list is empty.

### Potential fix direction

- Normalize all command paths to the same selection source.
- Re-test checked-item behavior under active filters and after clearing filters.
- Add a regression test for the exact publisher-filter scenario from the upstream issue.

---

## 10. Only the current HKCU uninstall roots are scanned, so user-context changes can hide apps

- Source: Upstream-backed
- Upstream reference:
  - [Issue #683 - After changing user rights, BCU does not see the current user's applications](https://github.com/Klocman/Bulk-Crap-Uninstaller/issues/683)
- Primary files:
  - `source\UninstallTools\Factory\RegistryFactory.cs`

### Why this looks buggy

`RegistryFactory.GetParentRegistryKeys()` enumerates uninstall entries from:

- `HKLM\...\Uninstall`
- current `HKCU\...\Uninstall`

That is fine for the normal case, but it does not help when applications are associated with a changed user context, another SID, or a different effective profile arrangement after user-rights changes.

### User impact

BCU can appear to "lose" per-user installs after account and privilege changes.
This matches the upstream report describing apps no longer showing up for either the old or new admin account.

### Potential fix direction

- Investigate whether affected installs live under another user SID or a per-user MSI context that is not being enumerated.
- Consider optional scanning of additional user hives or MSI user contexts where safe.
- Add diagnostics so the UI can show where a scan did and did not look.

---

## Notes for Future Expansion

- Not every finding above is already reported upstream.
- Some entries are probably direct root causes; others are strong leads that still need a clean repro.
- When adding more findings later, prefer keeping the same structure:
  - source,
  - upstream reference (if any),
  - primary files,
  - why it looks buggy,
  - user impact,
  - potential fix direction.

---

## Open Upstream Issue Audit - Current Open Issues

This pass expands beyond the initial 10 findings and reviews the **100 currently open upstream issues**.
For actionability, I split the current open issue list into:

- **25 open issues that do not look like bug reports** for this audit pass, and
- **75 open issues that are bug reports or bug-like user symptoms**.

This section is intentionally a triage pass, not a claim that every issue already has a confirmed root cause.
The goal is to identify which reports look fixable in this repo, which ones probably need repro data first, and which ones are not currently actionable from this repo alone.

### Open issues excluded from bug-fix triage

These currently open issues look primarily like feature requests, support requests, or translation/interface requests rather than fixable defects:

`#863, #841, #814, #813, #763, #756, #736, #703, #685, #640, #634, #616, #572, #568, #567, #566, #556, #544, #532, #514, #506, #493, #481, #451, #431`

### Fixability summary for the 75 bug-like open issues

| Bucket | Count | Meaning |
| --- | --- | --- |
| Likely fixable in current repo | 33 | Clear code surface, plausible root cause, and no obvious external blocker |
| Maybe fixable / needs repro | 37 | Likely bug report, but more logs, repro steps, or vendor-specific investigation are needed |
| Not currently actionable from this repo | 5 | Unsupported platform, AV reputation issue, vendor self-protection, or too little data |

### Cluster A - Startup, loading, serialization, and initialization failures

- Likely fixable: `#866, #816, #775, #552, #551`
- Maybe fixable / needs repro: `#827, #717, #655, #475`
- Not currently actionable from this repo: `#628`
- Likely code areas:
  - `source\KlocTools\IO\MsiTools.cs`
  - `source\BulkCrapUninstaller\Controls\Settings\AdvancedFilters.cs`
  - `source\UninstallTools\Lists\UninstallList.cs`
  - `source\BulkCrapUninstaller\Functions\Ratings\RatingManagerWrapper.cs`
  - `source\BulkCrapUninstaller\Functions\Ratings\UninstallerRatingManager.cs`
  - `source\UninstallTools\ThreadedWorkSpreader.cs`
- Why this cluster looks fixable:
  - `#866` already points at MSI enumeration and the issue discussion references concrete `MsiTools` call sites.
  - `#816` and `#775` match the startup `.bcul` / XML deserialization path.
  - `#552` and `#551` line up with the known JSON cache loading and foreground-thread hang risks.
  - `#827, #717, #655, #475` sound real but need logs or a clean repro to avoid guessing.
  - `#628` currently does not provide enough detail to map safely.

### Cluster B - Uninstall execution, quiet-uninstall stalls, and blocker-process logic

- Likely fixable: `#793, #579, #669`
- Maybe fixable / needs repro: `#818, #576, #631, #447, #788`
- Not currently actionable from this repo: `#571`
- Likely code areas:
  - `source\UninstallTools\Uninstaller\BulkUninstallEntry.cs`
  - `source\UninstallTools\Uninstaller\BulkUninstallTask.cs`
  - `source\UninstallTools\Uninstaller\UninstallManager.cs`
  - process / blocker detection around quiet uninstall automation
- Why this cluster looks fixable:
  - `#579` already includes a concrete suspected logic problem in stall detection.
  - `#793` fits the broader foreground-thread / endless-wait problems.
  - `#669` matches over-broad process relationship detection.
  - `#818, #576, #631, #447, #788` are plausible but may depend on vendor uninstallers, Edge hardening, or permissions.
  - `#571` likely runs into IOBIT self-protection rather than a repo-local defect.

### Cluster C - App detection, enumeration, visibility, and classification gaps

- Likely fixable: `#862, #800, #716, #683, #680, #443`
- Maybe fixable / needs repro: `#809, #649, #608, #560, #538, #418`
- Likely code areas:
  - `source\UninstallTools\Factory\RegistryFactory.cs`
  - `source\UninstallTools\Factory\StoreAppFactory.cs`
  - `source\UninstallTools\Factory\DirectoryFactory.cs`
  - `source\UninstallTools\Factory\ApplicationUninstallerFactory.cs`
  - platform-specific helpers for Steam / Store / Windows app enumeration
- Why this cluster looks fixable:
  - `#862` and `#443` strongly resemble misclassification under the protected/system-component filters.
  - `#800, #716, #680` look like missing detection rules or incomplete enumeration coverage.
  - `#683` maps well to HKCU / user-context scanning limitations.
  - `#809, #649, #608, #560, #538, #418` are plausible detection gaps, but some may depend on how Windows Store, OEM, or region-specific apps are exposed.

### Cluster D - Metadata, merge quality, selection state, size, and naming accuracy

- Likely fixable: `#792, #784, #781, #690, #468`
- Maybe fixable / needs repro: `#807, #794, #780, #779, #509, #465`
- Likely code areas:
  - `source\UninstallTools\Factory\RegistryFactory.cs`
  - `source\UninstallTools\Factory\InfoAdders\FastSizeGenerator.cs`
  - `source\UninstallTools\Factory\ScoopFactory.cs`
  - `source\BulkCrapUninstaller\Functions\ApplicationList\UninstallerListViewUpdater.cs`
  - `source\BulkCrapUninstaller\Forms\Windows\MainWindow.cs`
  - entry-merging logic in `source\UninstallTools\Factory\ApplicationEntryTools.cs`
- Why this cluster looks fixable:
  - `#792`, `#784`, and `#781` already have strong likely root causes in the current codebase.
  - `#468` appears to be the same family of selection-state bug as `#781`.
  - `#690` is a straightforward candidate for excluding symlink targets from app detection.
  - `#807, #794, #780, #779, #509, #465` are believable merge/metadata problems, but the exact bad input still needs to be captured.

### Cluster E - Leftover scanning, removal safety, and junk-confidence issues

- Likely fixable: `#758, #727, #724, #699, #584, #540, #520, #445`
- Maybe fixable / needs repro: `#821, #734, #728, #725, #647, #534, #499, #488, #504, #436, #435, #561`
- Likely code areas:
  - `source\UninstallTools\Junk\Finders\...`
  - junk confidence scoring and matching logic
  - `source\ScriptHelper\...`
  - service / registry deletion ordering
- Why this cluster looks fixable:
  - `#727, #724, #699, #520, #436` all point to over-aggressive or clearly wrong leftover confidence decisions.
  - `#758` suggests version-specific leftover grouping is too coarse.
  - `#584` and `#540` look like missing guardrails around missing folders or deletion order.
  - `#445` looks like a match-quality problem for leftover paths such as `LGHUB`.
  - `#821, #734, #728, #725, #647, #534, #499, #488, #504, #435, #561` all sound plausible, but they need careful repro data because this subsystem can be dangerous if fixed based only on anecdotal reports.

### Cluster F - UI layout, installer language behavior, DPI, dialogs, and visual state

- Likely fixable: `#810, #769, #714, #644, #558, #505`
- Maybe fixable / needs repro: `#691, #682`
- Likely code areas:
  - WinForms layouts and resource strings
  - installer packaging / language selection
  - file dialog initialization
  - DPI-aware rendering and splash/startup layout
- Why this cluster looks fixable:
  - `#810`, `#644`, and `#505` are classic layout/scaling defects.
  - `#769` looks like installer packaging logic rather than a deep architecture problem.
  - `#714` looks like a release/resource regression.
  - `#558` has reproducible multi-user file-picker steps.
  - `#691` and `#682` may still be fixable, but they are more sensitive to theme / rendering environment details.

### Cluster G - Performance, unsupported environments, reputation, and misc. edge cases

- Likely fixable: none from this cluster on current evidence
- Maybe fixable / needs repro: `#627, #494`
- Not currently actionable from this repo: `#770, #752, #495`
- Likely code areas:
  - performance hot paths across scanning / filtering / rendering
  - memory pressure around list generation or icon loading
- Why this cluster is lower confidence:
  - `#627` and `#494` look like real symptoms but need profiling or memory diagnostics before a safe fix can be proposed.
  - `#770` is primarily an antivirus reputation / packaging problem, not a normal code defect.
  - `#752` and `#495` are tied to unsupported legacy Windows versions rather than the current app target.

### Best high-confidence upstream candidates to validate first

If the goal is to turn this audit into actual fixes, these are the best current upstream-backed candidates to validate first:

`#866, #862, #793, #792, #784, #781, #775, #769, #727, #724, #716, #683, #669, #644, #584, #579, #558, #552, #551, #540, #520, #468, #445, #443`

These have the best mix of:

- concrete symptoms,
- a likely code owner inside this repo, and
- no obvious external blocker.
