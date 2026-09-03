# GTFS Validator — Memory optimization: implementation report

**Issue:** [#2183](https://github.com/MobilityData/gtfs-validator/issues/2183) · **Base:** `21b27a4a` (master) · **Commits:** 13, split into the 6 pull requests listed in §8
**Modules touched:** `core`, `main`, `processor`

Intervention IDs (`M-xx`) below are the ones used in the analysis behind this work.

This document is written for reviewers. It states what was changed, why each change was necessary,
what it costs and what it buys, and where behaviour is observably different.

---

## 1. Summary

The validator was unable to process a large feed inside a container with a 3 GB memory limit. The
cause was not the size of the loaded entities, which are already compact and well designed, but four
structural amplifiers: an index built for every row of the two largest files and read almost never,
notice retention limits that the row-by-row parsing loop bypassed entirely, a complete JSON tree
materialized for every notice although the report prints fifty per code, and the whole loaded feed
held alive while the reports — the second most allocation-heavy phase of the run — were generated.

Thirteen commits address them, one intervention per commit. On the reference feed described in
§2, the smallest heap in which validation completes drops from **2226 MB to 127 MB**, and the run
becomes **four times faster**, because most of its time was being spent collecting garbage.

The `notices` object of `report.json` is byte-for-byte identical before and after. One behaviour
change is deliberate and documented in §5; one further difference is a bug fix, also in §5.

| Measurement | Before | After |
|---|---:|---:|
| Smallest `-Xmx` in which the feed validates | **2226 MB** | **127 MB** |
| Wall-clock validation, both at `-Xmx2560m` | 40.6 s | 9.6 s |
| Time spent collecting garbage, at `-Xmx2560m` | 17.3 s | 0.24 s |
| Collections during validation, at `-Xmx2560m` | 95 | 31 |
| `report.html` written to disk | 32.9 MiB | 0.29 MiB |
| `notices` object of `report.json` | identical | identical |

---

## 2. How the measurements were taken

**Acceptance metric.** The plan fixes the metric as *the smallest `-Xmx` at which validation of a
reference feed completes successfully*, found by binary search. This is a whole-system number that
does not depend on a profiler's accounting, and it is the number that decides whether the library
fits in a given container. It was re-measured after each phase.

**Reference feed.** A synthetic feed of 52 MB of CSV: 800 000 rows in `stop_times.txt`, 300 000
points in `shapes.txt`, 8 000 trips, 2 000 stops, 1 000 shapes. Every `stop_times.txt` row carries a
quoted trailing space in `stop_headsign`, so parsing emits one `LeadingOrTrailingWhitespacesNotice`
per row: that is the notice explosion M-02 has to bound, and it is a realistic shape — a single
column with trailing whitespace is a common export defect. The feed also produces 193 800
`stop_too_far_from_shape` warnings from a multi-file validator, which exercises the notice path that
does *not* go through `addAll`.

**Environment.** JDK 17 (Corretto), Serial GC, `gtfs-validator-cli` shadow jar, `-svu` to skip the
version check. Timing figures are the mean of three runs; both builds were timed at `-Xmx2560m`, the
smallest round heap the *current release* can run this feed in. GC counts and pause totals come from
`-Xlog:gc`.

**Caveat.** These numbers come from one synthetic feed chosen to exercise the amplifiers the plan
identified. They establish the direction and order of magnitude. The project's acceptance-test run
over the reference corpus is the check that should gate merging — see §7.

---

## 3. Phase 1 — reducing the live set

Ordered as implemented. Each is an independent commit.

### M-05 — Release the feed container before generating reports
`main/.../runner/ValidationRunner.java`

**Problem.** `run()` held a strong reference to the `GtfsFeedContainer` while `exportReport` ran,
because `printSummary` needed the container afterwards to log the per-table totals. Report
generation is itself the second most allocation-heavy phase of the process, so the peak was
`live(model) + peak(report)` instead of `max(live(model), peak(report))`. On a large feed the
container alone is over a gigabyte.

**Change.** The table totals are computed before the reporting phase and the container reference is
dropped, so the entities become collectable. `printSummary` gains an overload taking the totals as a
`String`.

**Compatibility.** `printSummary(FeedMetadata, GtfsFeedContainer, GtfsFeedLoader, ValidationRunnerConfig)`
is a published `public static` method; it is kept as a `@Deprecated` delegate to the new overload,
and made null-safe on the container so that the pre-existing early return for `stdoutOutput` still
behaves the same.

**Tests.** `ValidationRunnerTest.printSummary_tableTotalsOverload_logsSameOutputAsFeedContainerOverload`
captures what both overloads log and asserts the output is identical.

---

### M-02 — Enforce notice retention limits when merging containers
`core/.../notice/NoticeContainer.java`

**Problem.** `CsvFileLoader` creates a `NoticeContainer` per parsed row and merges it into the
container of the table with `addAll`, which applied **no limit at all**. Each per-row container held
0–3 notices, so the per-container limits never triggered, and every row-level parsing notice was
retained. Row-level notices are all `WARNING`, so they do not stop loading, and they accumulate: on
a feed where one column has trailing whitespace on every row, that is one retained notice per row,
each holding its own non-trimmed `String` — unbounded growth proportional to the file.

**Change.** The retention limits now live in a single private `canRetain`, used both by
`addValidationNoticeWithSeverity` and by `addAll`. The counters that feed `totalNotices` are merged
in full regardless of retention, in a map kept separate from the new one that counts what was
actually retained, so the exported counts remain exact.

The defaults are lowered: at most `MAX_EXPORTS_PER_NOTICE_TYPE_AND_SEVERITY = 1 000` notices per
type and severity are ever exported and the HTML report lists 50, so retaining 100 000 of them was
pure overhead.

| Constant | Was | Now |
|---|---:|---:|
| `MAX_VALIDATION_NOTICES_TYPE_AND_SEVERITY` | 100 000 | 2 000 |
| `MAX_TOTAL_VALIDATION_NOTICES` | 10 000 000 | 200 000 |
| `MAX_EXPORTS_PER_NOTICE_TYPE_AND_SEVERITY` | 1 000 | 1 000 (unchanged) |

The 2× margin over the export limit leaves room for consumers that read more than the report shows.
Callers needing different bounds still have the three-argument constructor.

**One subtlety worth reviewing.** A notice type is described in the exported report only if at least
one of its notices was retained: `ValidationReportDeserializer.serialize` builds the reported set by
grouping the *retained* list and only then looks up the exact count. Enforcing the total limit in
`addAll` therefore made it possible for a type to vanish from `report.json` entirely, count
included. `canRetain` keeps the first notice of each type and severity even past the total limit, so
that cannot happen. This is a small, deliberate relaxation of the total limit, bounded by the number
of distinct notice types.

**Tests.** Four new cases in `NoticeContainerTest` covering the per-type cap under `addAll`, the
total cap under `addAll`, exact `totalNotices` beyond the retention limit, and the error/warning
flags surviving when notices are dropped; plus
`addAll_totalLimitReached_stillRetainsTheFirstNoticeOfEachType`.

**Behaviour change.** See §5.

---

### M-04 — Cache notice documentation comments per notice class
`core/.../notice/schema/NoticeSchemaGenerator.java`

**Problem.** `loadComments` opened a classpath resource and parsed JSON on every call, and
`NoticeView` calls it once per notice. Generating the HTML report for a feed with many notices
repeated the same resource read and JSON parse hundreds of thousands of times, and retained one
identical `NoticeDocComments` — each with its own map — per notice. This was simultaneously the
largest CPU cost of report generation and a per-notice memory cost.

**Change.** Memoized in a `ConcurrentHashMap` keyed by the notice class. The result depends only
on the class, and notice classes are loaded once and live for the lifetime of the process, so
nothing more elaborate is needed and the cache behaves the same on any runtime.

Both per-class caches read with `get` before falling back to `computeIfAbsent`. That is not
redundant: `computeIfAbsent` locks the bin of a key that is already cached unless the key happens to
be the first entry of that bin, which on 53 notice classes it often is not. Measured on those
classes, per lookup: `ClassValue` 2.5 ns, `computeIfAbsent` 6.1 ns, `get` first 3.9 ns
single-threaded; 0.65 / 2.34 / 1.05 ns per operation with four threads.

**Note for reviewers.** `NoticeDocComments` is mutable and the cached instance is now shared. No
caller in the repository mutates it (`NoticeView` and `generateSchemaForNotice` only read), and the
cache declaration says so in its javadoc. If the project prefers a hard guarantee, the type could be
made immutable in a follow-up.

**Tests.** `loadComments_isCachedPerNoticeClass` asserts the same instance is returned, and
`loadComments_noticeWithoutComments_returnsEmptyComments` covers the missing-resource path.

---

### M-03 — Build only the notice views the HTML report lists
`main/.../reportsummary/model/ReportSummary.java`, `HtmlReportGenerator.java`, `main/src/main/resources/report.html`

**Problem.** `ReportSummary` created a `NoticeView` for every retained notice, and each `NoticeView`
materializes the complete Gson tree of its notice. The template lists at most 50 records per notice
code, so the overwhelming majority of those trees were built and discarded without ever being read —
and built at the very end of the run, when the heap is already at its peak.

**Change.** Views are built only up to the limit the report renders. The limit is defined once, as
`ReportSummary.MAX_NOTICES_PER_CODE`, and read by the template through `summary.maxNoticesPerCode`
instead of being repeated as the literal `50` in two places in `report.html`.

`HtmlReportGenerator.getUniqueFieldsForCodes`, which derives the table columns for a code, now sees
the truncated lists. That is the intended scope: the columns are derived from exactly the rows that
are rendered. A column whose field is set only on the 51st occurrence and later would previously
have been emitted with `N/A` in all 50 rendered rows; it is now omitted. A comment on the method
records this.

**Result.** On the reference feed the produced `report.html` goes from 32.9 MiB to 0.29 MiB, and the
rendered page is unchanged apart from the counts corrected in §5.

**Tests.** `HtmlReportGeneratorTest` is new — the class had no test at all. It renders a report with
more notices than the limit and asserts on the visible text of the page: the list is truncated, the
"Only the first 50 of N" line is correct, and the per-code total is the true one. Plus
`ReportSummaryTest.noticesMapTest_isTruncatedButCountsAreNot`.

---

### M-01 — Detect duplicate keys without a per-row map
`processor/.../processor/TableContainerIndexGenerator.java`

This is the only change to generated code, and the one that most deserves a careful read.

**Problem.** For every table with a multi-column primary key, the generator emitted a
`Map<CompositeKey, Entity> byCompositeKeyMap` populated with **one entry per row**. It serves two
purposes: detecting duplicate primary keys during `setupIndices`, and answering
`byTranslationKey(recordId, recordSubId)` — which is called only from
`TranslationFieldAndReferenceValidator`, that is, only when the feed contains a `translations.txt`
referring to that table.

The cost is roughly 60 bytes per row (a `HashMap.Node`, an AutoValue `CompositeKey`, and a table
slot). For `stop_times.txt` and `shapes.txt` — the two tables with millions of rows — that is
hundreds of megabytes spent on an index that a typical feed never reads.

**Why those two tables can avoid it.** Both have a primary key of exactly two columns, where one
column carries `@Index` and the other is declared `isSequenceUsedForSorting = true`. `setupIndices`
therefore already builds a multimap keyed on the first column and **sorts each group** by the second.
Two entities share a primary key if and only if they are in the same group with the same sequence
value.

**Change.** The generator now recognises that shape (`resolveSortedGroupIndexField`) and, for those
tables only:

1. moves the index population and sort *before* the duplicate detection;
2. visits entities in the order they were loaded and, for each one, locates by **leftmost binary
   search** the first entity of its group with the same sequence value. If that entity is not the
   one being visited, the visited entity is a duplicate of it, and the notice is emitted immediately.
3. builds `byCompositeKeyMap` lazily, on first call of `byTranslationKey`, with `putIfAbsent` so the
   first occurrence still wins. The accessor is `synchronized`, because multi-file validators may run
   on several threads over the same container. For tables that do not support translation lookup at
   all, the field is not generated.

**Why this is equivalent, not merely similar.** Four properties make the result identical to the
map-based detection:

- the index population loop is unconditional, so every entity lands in a group even when its key
  field is absent (the accessor returns the type default, never null): the groups are a true
  partition of `entities`;
- the sort is `List.sort`, which is stable, so entities with equal sequence values keep their
  original relative order, and the leftmost one is therefore the first occurrence;
- every duplicate is reported against the **first** entity holding that key, not against its
  predecessor — matching the existing semantics, where three duplicate rows produce the pairs `(1,2)`
  and `(1,3)`, not `(1,2)` and `(2,3)`;
- `CompositeKey.getDefinedKeys/getDefinedValues` depend only on their entity argument, and are
  called with the first entity, exactly as before.

Because entities are visited in their original order, the notices reach the `NoticeContainer` in that
same order. This matters: the JSON export takes the *first* 1 000 notices of a type, so a different
order would change the exported sample.

**What is deliberately not changed.** Every other table with a composite primary key —
`transfers.txt`, `fare_rules.txt`, `calendar_dates.txt`, `frequencies.txt`, `translations.txt`,
`stop_areas.txt`, `timeframes.txt`, the fare tables — keeps the map-based detection. They are small,
and the existing code is simpler. Note in particular that `calendar_dates.txt` and `frequencies.txt`
*look* eligible — two-column key, first column indexed — but their second column is not a sorting
sequence, so their groups are never sorted and the predicate correctly excludes them.

**Tests.** `GtfsStopTimeDuplicateKeyTest` and `GtfsShapeDuplicateKeyTest` are new, covering: no
false positives, runs of three or more duplicates, duplicates interleaved across groups (which
verifies the reporting order), the `forEntities` path where entities may share a `csvRowNumber`,
`byTranslationKey` before and after duplicates, and the index remaining sorted. Four cases were added
to the processor-level `IdAndSequencePrimaryKeySchemaTest`, including duplicates on a missing id. The
generated output was also read by hand for `GtfsStopTimeTableContainer`, `GtfsShapeTableContainer`
and, as controls, `GtfsTransferTableContainer` and `GtfsCalendarDateTableContainer`.

---

### M-06 — Serialize the JSON reports straight to their destination
`main/.../runner/ValidationRunner.java`

**Problem.** `Files.write(path, gson.toJson(report).getBytes(UTF_8))` builds the entire report as a
`String` and then copies it into a `byte[]`, holding two or three copies of a report that can reach
tens of megabytes — at exactly the moment the heap is at its peak.

**Change.** The report is serialized directly into a `Writer` on the output file; the stdout variant
writes to `System.out`. Gson uses the same serialization path for both APIs, so the produced bytes
are unchanged.

Gson reports a failing writer as `JsonIOException`, which is unchecked, so the system-errors write
now catches it alongside `IOException` — otherwise a full disk would have escaped the `catch` that
the previous `Files.write` form matched.

**Trade-off.** The output file is opened before serialization runs, so a failure part-way through
now leaves a truncated file where previously nothing was written. This is inherent to streaming and
is the accepted cost of not building the document in memory first.

**Tests.** `ValidationRunnerTest.exportReport_writesUtf8EncodedJsonReports` writes both reports with
a non-ASCII filename in a notice and checks encoding and content.

---

### M-14 — Close the `ZipInputStream` used to look for a subfolder
`core/.../input/GtfsInput.java`

**Problem.** `hasSubfolderWithGtfsFile` and `createFromUrlInMemory` each opened a `ZipInputStream`
over the archive and never closed it. Each holds a file descriptor and an `Inflater`, which owns a
zlib stream in **native** memory released only when its cleaner runs. Irrelevant for a single CLI
invocation; a progressive leak of descriptors and resident memory for a long-running service
validating one feed after another — and resident memory is what gets a container killed.

**Change.** try-with-resources at both sites.

**Tests.** `hasSubfolderWithGtfsFile_releasesTheArchive` deletes the archive after the call, which
fails on Windows if a handle is still open, plus a case covering detection itself.

---

## 4. Phase 2 — reducing allocation during loading

Phase 1 addresses what is *retained*. Phase 2 addresses what is *allocated and discarded*, which
under a stop-the-world collector drives premature promotion and therefore full collections. Measured
on its own at `-Xmx512m`, this phase takes collections during validation from 76 to 50 and their
total pause from 299 ms to 206 ms.

### M-10 — Invoke single-entity validators without a lambda per entity
`core/.../validator/ValidatorUtil.java`

`invokeSingleEntityValidators` wrapped each call in `safeValidate(c -> validator.validate(entity, c), …)`.
That lambda captures the entity, so it is a fresh allocation on every call — once per entity per
validator, and `GtfsStopTime` has four single-entity validators. The validators are now invoked
directly, with the same try/catch, extracted into a shared `logRuntimeException`. `safeValidate` is
unchanged for its other callers, which run once per file rather than once per row.

*Tests:* `ValidatorUtilTest` is new, covering invocation order and that a throwing validator is
recorded as a system error without stopping the others.

### M-09 — Reuse one notice container for the row-by-row parsing loop
`core/.../table/CsvFileLoader.java`, `core/.../notice/NoticeContainer.java`

The loop allocated a `NoticeContainer` per row — each with two lists and two maps — to carry the 0–3
notices a row typically produces. It now reuses one container, emptied by the new `NoticeContainer.reset()`
at the start of each row. `addAll` copies the notices into the destination container, so emptying the
source afterwards loses nothing; the test asserts exactly that, and that a reset container regains
its full retention capacity.

### M-08 — Build one cell context per validated cell instead of two
`core/.../parsing/RowParser.java`

Parsing an id, URL, e-mail address or phone number went through `asString`, which built a
`GtfsCellContext` for the validation every field goes through, and then built a **second, identical**
one for the type-specific validation. Selecting that validation with `fieldValidator::validateId`
allocated a capturing lambda as well, on every call.

The context is now built once and passed to both validations. The type-specific validation is
selected by an enum constant rather than a method reference, which also avoids dereferencing the
field validator before it is needed — `RowParser` is constructible with a null validator, and a test
relies on that.

This is the **conservative** variant the plan proposes: `GtfsCellContext` remains immutable, so
nothing changes for a validator that retains it. The mutable-context variant was not implemented.

### M-11 — Compute notice codes and grouping keys once
`core/.../notice/Notice.java`, `core/.../notice/ResolvedNotice.java`

`Notice.getCode` derived the code from the class name on every call, building several intermediate
strings, and `getMappingKey` concatenated code and severity every time it was asked — twice per
notice added to a container, and once more per notice when notices are grouped for export. The code
is memoized per class in a `ConcurrentHashMap`; the key is computed once in the `ResolvedNotice`
constructor. The four bytes this adds per resolved notice are bounded by the retention limits from
M-02, which is why the plan sequences it after them.

### M-15 — Read CSV files on the parsing thread
`core/.../parsing/CsvFile.java`

Univocity reads input on a thread of its own when several CPUs are available, costing a thread and
two buffers of the input buffer size per open file; the buffer size is also set explicitly, below the
default. The validator loads one table at a time by default, so the throughput
benefit is small. Measured: validation is slightly faster (9.7 s against 10.0 s at `-Xmx512m`) and
noticeably more consistent between runs, with the minimum heap unchanged. The plan's condition for
adopting this — that throughput must not worsen — is met.

### Not implemented

- **M-12 (pre-size the entity list).** `CsvFileLoader` is handed an `InputStream` with no size
  information. Plumbing the ZIP entry size through would change the loader API, and the saving is now
  below the resolution of the acceptance measurement. Deliberately skipped, per the plan's rule that
  a change without a measurable improvement should not be made.
- **M-13 (`locations.geojson` parsed twice).** Real, but it affects GTFS-Flex feeds only and deserves
  its own change.
- **Phase 3** (columnar entity storage, binary-search replacement of the lazy map, invasive M-07)
  was not attempted: Phase 1 and 2 already exceed the memory goal by a wide margin.

---

## 5. Observable behaviour changes

There are exactly two, and reviewers should focus their attention here.

### 5.1 Fewer notices retained on pathological feeds — deliberate

This is the intended change from M-02, and the only intended one in the series.

- `totalNotices` in `report.json` is **unaffected** — counts are merged in full.
- The exported sample is **unaffected** for any notice type with up to 2 000 occurrences, and remains
  a valid sample of the first 1 000 above that.
- A caller iterating `NoticeContainer.getValidationNotices()` on a feed producing hundreds of
  thousands of one notice type will receive a shorter list than before.

The three-argument `NoticeContainer` constructor is unchanged, so any consumer needing different
bounds can set them.

### 5.2 The HTML report now agrees with the JSON report — bug fix

`ReportSummary` derived every number it displays — the total, the per-severity counts and the
per-code total — from the notices the container had *retained*, never from the exact counters. That
was equivalent to the true number only as long as nothing was ever dropped.

It was already wrong before this branch: on the reference feed, the current release renders
"100 000" for `stop_too_far_from_shape` while its own `report.json` reports 193 800 for the same run,
because the pre-existing per-type limit of 100 000 had truncated the retained list. Lowering the
limits in M-02 would have made the discrepancy far more visible.

The counts now come from the container, via a new
`NoticeContainer.getNoticeCount(String code, SeverityLevel severity)`. On the reference feed the
rendered page is identical to the one produced by the current release **except** for those corrected
numbers, the version string and the timestamp.

---

## 6. Public API surface

Everything below is additive, except the one deprecation.

| Type | Member | Change |
|---|---|---|
| `NoticeContainer` | `int getNoticeCount(String, SeverityLevel)` | added — exact count including notices not retained |
| `NoticeContainer` | `void reset()` | added — empties the container for reuse |
| `ReportSummary` | `int getNoticeCountForCode(SeverityLevel, String)` | added |
| `ReportSummary` | `int getMaxNoticesPerCode()` | added — read by `report.html` |
| `ValidationRunner` | `printSummary(FeedMetadata, String, GtfsFeedLoader, ValidationRunnerConfig)` | added |
| `ValidationRunner` | `printSummary(FeedMetadata, GtfsFeedContainer, GtfsFeedLoader, ValidationRunnerConfig)` | **deprecated**, delegates to the above |

No method was removed and no signature was changed in place. Generated table containers gain a
private lazy accessor for `byCompositeKeyMap` on `stop_times.txt` and `shapes.txt`; the public
surface of the generated classes is unchanged.

---

## 7. Verification performed, and what is still owed

**Done:**

- **Report equivalence.** The reference feed was validated with a build of `03717cb3` and with this
  branch. The `notices` object of `report.json` is identical; the visible text of `report.html`
  differs only in the version string, the timestamp, and the two counts corrected in §5.2.
- **Tests.** 1009 tests run across all modules, all passing except the one pre-existing failure noted
  at the end of this section. 37 test methods were added across 11 files, 4 of them new files,
  including the first test `HtmlReportGenerator` has ever had.
- **Generated code inspected by hand** for the two optimized containers and two control containers
  that must keep the previous behaviour.
- **Adversarial review of the diff**, which is how the count inconsistency of §5.2 was found — it was
  not caught by any test, before or after.
- `./gradlew spotlessCheck` and `./gradlew assemble` are green.
- **The split into six pull requests was measured, not assumed.** A CLI jar built from the six
  branches and one built from the original single branch were run on the same 46 MB synthetic feed
  (800 000 whitespace notices, 200 000 `stop_too_far_from_shape`): the `notices` object of
  `report.json` is identical, `report.html` is byte-identical in size and heading, wall clock is
  9.4 s against 9.5 s over four interleaved runs each (inside the run-to-run spread of either
  build), garbage collections 15 and pause totals 190-202 ms on both, and the smallest `-Xmx` in
  which the feed validates is **51 MB for both**, against 2796 MB for `master` on the same feed. The
  only behavioural difference between the two builds is the cache implementation described in M-04
  and M-11.

**Still owed before merging:**

- **The project's acceptance test / `output-comparator` run over the reference corpus.** The
  measurements here use a single synthetic feed. The corpus run is the authoritative check that no
  real feed changes its report, and it is the one this report cannot substitute for.

**Not owed.** GraalVM native-image support is explicitly not a project target — everything
MobilityData ships runs on HotSpot through jpackage and jlink — so nothing here needs to be verified
against it. The two per-class caches use a plain `ConcurrentHashMap` rather than `ClassValue` for
that reason: same result, nothing runtime-specific to check.

**Known unrelated failure.** `output-comparator`'s
`ValidationPerformanceCollectorTest.generateLogString_test` fails on a machine whose default locale
formats decimals with a comma: it asserts on `String.format("%.2f", …)` output. It reproduces on
unmodified `master` and is unrelated to this work.

---

## 8. Pull requests

The work is split into six pull requests, in this order, because each one changes the report output
the next one is compared against. Every commit is independently reviewable and revertible.

| PR | Commits | Interventions | Why it is on its own |
|---|---|---|---|
| 1 | `fix: report the exact notice counts in the HTML report` | §5.2 | A bug fix that stands on master today, and it changes the numbers the page shows, so it goes first and every later report comparison starts from correct counts. |
| 2 | `fix: close the ZipInputStream used to look for a subfolder`, `perf: cache notice documentation comments per notice class`, `perf: serialize the JSON reports straight to their destination` | M-14, M-04, M-06 | No observable change: same reports, same bytes, fewer allocations and one leak fewer. |
| 3 | `perf: release the feed container before generating reports` | M-05 | Touches the shape of `ValidationRunner.run` and adds a `printSummary` overload; the deprecated signature has exactly one in-repo caller and is kept for external users. |
| 4 | `perf: enforce notice retention limits when merging notice containers`, `perf: only build the notice views the HTML report lists` | M-02, M-03 | Carries the one deliberate behaviour change (§5.1) and the column-set consequence for `HtmlReportGenerator.getUniqueFieldsForCodes`, so it is the group to discuss rather than something to hide inside a larger diff. |
| 5 | `perf: detect duplicate keys of stop_times and shapes without a per-row map` | M-01 | Changes generated code for every table container and makes `byTranslationKey` a lazily built synchronized fallback used by a multi-threaded loader. |
| 6 | `perf: invoke single-entity validators without a lambda per entity`, `perf: reuse one notice container for the row-by-row parsing loop`, `perf: build one cell context per validated cell instead of two`, `perf: compute notice codes and grouping keys once`, `perf: read CSV files on the parsing thread` | M-10, M-09, M-08, M-11, M-15 | Allocation-rate work on the loading path, which is where the remaining GC time was. |

Dependencies across the groups, all satisfied by this order: PR 6 uses the `ResolvedNotice`
grouping-key helper introduced by PR 1 and the retained-notice bookkeeping introduced by PR 4.
