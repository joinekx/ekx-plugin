# Import native ETAP data

Use existing ETAP numerical exports when supplied. This path does not digitize PDFs. It either adds
curves to an exact existing private catalogue record or, for a fuse product line or a Siemens BLH
or QRH2 breaker line with no record yet, creates a private record and parts from facts in the ETAP
library, under the family name you state. It uses the same review and
atomic apply workflow as PDF imports.

## Prepare the packet

Use the original point-export record and its matching export manifest, not a hand-transcribed list
or a plot generated from those points. Preserve original text, trace IDs, byte offsets, library and
parser hashes. The server verifies the manifest, record identity, geometry, and target applicability.
The entire request has a 2 MiB limit.

`create_etap_catalog_import` accepts:

- `manifestText`: unchanged text of the collection's `points/manifest.json`, or of a model
  folder's `manifest.json`. A collection's top-level `manifest.json` describes its sources, not
  its records, and fails every record. For a product line that mixes revisions, send `manifests`
  instead (see "Import a product line").
- `recordText`: unchanged exported record file text, including native point arrays.
- `recordFile`: the record's bare filename, such as `100A_490066.json`, with no directory, even
  when the manifest lists a relative path.
- `targetCatalogId`: the exact private catalogue UUID from the applied facts build or supplied URL.
- `skuId`: the exact applicable SKU UUID, or null for record-level evidence. Never invent a SKU ID.
  Applied build reads expose current active private SKU identities in `savedSkus`. Match the exact
  part number and rating there; do not rerun PDF extraction just to rediscover saved SKU IDs.
- `targetApplicabilityEvidence`: evidence connecting manufacturer, release, rating and poles to the
  target. A family name alone is insufficient.
- `series` and `recordName` (only without a target): the names of the record the build creates.
  See "Name the record".

## Name the record

A build without a target must state the maker's product family as `series: { value, evidence }`,
read from a source that names it: the record's `Typ/Model` text, a datasheet or the nameplate.
ETAP's model or folder name is a library label, not a family: `TM210(3VA41)` is Siemens's TM210
trip unit for the `3VA41` frame, so the series is `3VA41` and the trip unit is `TM210`; `LPJ_SP` is
the `LPJ-SP` family. EKX refuses a series with brackets, an underscore, a current rating, `·`, or an
ETAP reference, and says what to change.

The series is also the record name unless the maker already has a record in that family with
different fixed settings. Then send `recordName: { value, evidence }`: the family plus the shortest
real qualifier that tells them apart, such as `3VA52 TM230` or `TEY 30 A`.

A part number is a real orderable catalogue number. Give a record `partNumber: { value, evidence }`
only when a source shows that number; never build one from the series and rating. Without one the
part is named by its variant, the ETAP rating such as `100 A` or `12R`, and the app shows it as
`LPJ-SP · 100 A`.
- `completeMissingFixedSettings`: use `true` only for one supported P10022 BL 100 A or CED6
  125 A configured thermal-magnetic record targeting the exact private manufacturer-facts
  record with `skuId: null`. It completes missing fixed thermal/magnetic types and the reviewed
  nominal instantaneous multiple with ETAP-derived provenance, in the same approved package.
  The nominal must lie inside the native band; do not describe it as a manufacturer factory
  setting. Existing conflicting settings are refused. Omit it to preserve curves-only behavior.
- `interpretations`: explicit units and trace roles, or supported parameter/settings mappings.
- `idempotencyKey`: a new key, reused only for an exact retry with unknown outcome.

Call `inspect_etap_catalog_import` with the unchanged `manifestText`, `recordText`, and
`recordFile` before writing interpretations. It saves nothing and returns the
`inspectionFingerprint`, record identity (model, rating, trip type), trace IDs with their sections
and point counts, pickup evidence, exported electronic selector IDs, and limitations. Use exactly
that fingerprint and those IDs; `create_etap_catalog_import` publishes the interpretation schemas.
A failure returns a specific code, such as `NATIVE_ETAP_RECORD_NOT_IN_MANIFEST` or
`NATIVE_ETAP_HASH_MISMATCH`; fix the packet instead of editing the exported text.
Do not invent a fingerprint or treat raw coordinates as self-describing.

Vendor-wide library exports (manifest `source_type: "etap_vendor_library_point_export"`) cover
every manufacturer in one layout. Their fuse records are importable. When the inspection
reports `averageCurve: true`, import one trace as `published_time_current`; never label an average curve as minimum melt or total clear.
Allen-Bradley TM v11 records support inspection only; their raw parameters and trace counts do not
establish approved units or configured operating curves. Conversion remains unavailable (see the
[native intake limits](../../../../../src/lib/etap-import/README.md#interpretation-limits)).
The exact Siemens JD6-A 300 A 240 V version 9 TM record SID 785098 is importable. Its
`thermal_magnetic` interpretation uses native rated-current multiples, thermal traces 1/2, no
magnetic trace mapping, fixed Ir 300 A, and the exact native dial pickup (dial 5 is
1959.8999977111816 A). The importer assembles the stored magnetic line parameters and retains a
structured 240 V AC curve condition. Other voltage branches need their own source. See
[the reviewed native mapping](../../../../../src/lib/etap-import/README.md).
Other record types return `NATIVE_ETAP_UNSUPPORTED_RECORD_TYPE`. A collection manifest over the
2 MiB request limit cannot be sent yet; report it instead of trimming the manifest.

For ordinary stored coordinates, the supported units are amperes and seconds. HJD source
inspection accepts the nine three-pole records from 70–250 A, preserving native arrays and
hashes. Configured HJD curves use the `thermal_magnetic` interpretation with
`currentUnit: { kind: "rated_current_multiple", referenceAmps: <exact exported rating> }` and
`thermalPickupAmps` equal to `ratingAmps`; the HJD thermal element is fixed. Each boundary
follows its thermal trace to the selected pickup tolerance, runs vertically there, then keeps
the native magnetic array unchanged. ETAP-rendered 125 A TCCs at the 5x and 10x detents verify
this assembly, and ETAP governs: its library clears in 0.0197 s at 11 times rating where Eaton
SC-4247-87D shows 0.0240 s, and that difference is recorded on the curve, not corrected. The
seven intermediate detents follow the same rule from their stored tolerances and have no
separate ETAP render. Never relabel native multiples as amperes or rewrite the source arrays.

Siemens BLH and QRH2 exports (`kind: "etap_siemens_blh_qrh2_stored_points"`, from each family's
`points_v4/manifest.json`) import three-pole records as fixed thermal-magnetic breakers. Use the
`thermal_magnetic` interpretation with `thermalMapping: { lowerTraceId: "thermal_1",
upperTraceId: "thermal_2" }`, no `magneticMapping`, `thermalPickupAmps` equal to `ratingAmps`, and
`maxTimeSeconds` up to 1,000. ETAP's library is the authority for these releases' instantaneous
pickup: set `instantaneousPickupAmps` to a nominal inside the stored pickup band, 10x In for QRH2
and 11x In for BLH 100 A; BLH 20 A uses 425 A (21.25x In) within its native 350–500 A band.
Never copy a fixed multiple from another handle rating; the band itself sets the curve. QRH2 uses `currentUnit: { kind: "amperes" }` and its complete stored traces.
BLH uses `currentUnit: { kind: "rated_current_multiple", referenceAmps: <rating> }` and
`domain: { maxAmps }`, the reviewed current where the clearing floor ends (22,000 A, the
interrupting rating, for BLH 100 A); the server extends each overload boundary along its last
stored segment to its stored pickup, as ETAP draws it, and says so on the curve. The inspection
reports `fixedRelease`; a BLH record without decoded fixed instantaneous values cannot import.

Review whether traces are a nominal line or lower/upper band, and retain the export's
time/current serialization order. For thermal-magnetic settings and electronic parameter
records, use their explicit interpretation variant only when all required settings, reference
bases, and setting-dependent transformations are established. Parameter-only data is not a
stored-point export. Never invent unexported curves or broaden fixed-setting coverage.

For electronic records, distinguish the native numerical current reference from the breaker's
physical frame size. Cross-check the reference against exported selector labels and manufacturer
settings before supplying `bases.frameAmps`. For example, ETU320 100 A record SID 1250847 labels
0.4 as 40 A and 12 as 1200 A, establishing a 100 A native reference even though the applicable
3VA6110 breaker has a 150 A physical frame. Keep 150 A in the breaker facts; do not use it to scale
these native settings.

The current electronic adapter generates separate unsmoothed function bands. It does not assemble
an LI/LSI operating characteristic or synthesize minimum-time floors and breaker clearing tails.
Check pickup tolerances, delay reference, minimum times and clearing boundaries independently
against the manufacturer drawing. Preserve discrepancies as source differences, not corrections
to ETAP. Do not claim a complete usable breaker curve merely because the function bands import.
If completing the requested import requires extending this behavior or choosing which source
governs a conflicting value, surface that design decision before changing the interpretation.

For the supported Siemens 3VA5440 TM120M1 400 A MCP export, use the `magnetic`
interpretation when importing configured instantaneous protection. Supply the two magnetic trace
IDs and explicit settings: `kind: "magnetic"`, `version: 1`, `ratingAmps`,
`instantaneousPickupAmps`, and `maxTimeSeconds` (at most 1,000). Each interpretation produces
one exact setting variant. Import all requested source-supported detents; do not interpolate
between them. The server checks the selected native pickup and tolerance, clips the timing
boundaries at those pickup limits, and retains the original timing tails. This device has no
thermal pickup or overload protection; the manufacturer's damage curve is not a trip curve.

When only unconfigured timing evidence is wanted, `sourceRole: "magnetic_timing_reference"`
with `curveType: "parameter_space_reference"` retains the original lower/upper timing points.
That reference is excluded from installed profiles. The first timing-domain current is not the
selected pickup. Do not substitute a reference-only import for requested configured curves.

## Import a product line

Send one build per fuse product line: records of one ETAP manufacturer and base model (the model
without its trailing revision), at most 100. For each rating, voltage and speed, pick one record,
normally the newest revision with separate melt and clear curves; a line may mix revisions, such
as FRN-R 30A from `FRN-R (10/08)` and FRN-R 600A from `FRN-R (8/00)`. Two records for the same
rating, voltage and speed are refused, and the error names both.

Put each record in `records` with its `recordFile`, `recordText` and `interpretations`. When all
records come from one model folder, send that folder's `manifest.json` as `manifestText`. When
revisions are mixed, send `manifests` (at most 8), each `{ manifestRef, manifestText }` from a model
folder under `points/FUSE/<Manufacturer>/`, and give every record the `manifestRef` of the manifest
that lists it. Call `inspect_etap_catalog_import` with the same records and manifests first; each
result gives that record's `inspectionFingerprint`, `rating`, `libraryFacts` and trace issues.

- To add curves to an existing record, send `targetCatalogId`, `targetApplicabilityEvidence`, and
  each record's exact `skuId` from `savedSkus` (or `null`).
- With no catalogue record yet, omit the target and name the record (see "Name the record"). EKX
  creates a private record and one part per record with rated current, rated voltage and the AC
  interrupting rating, all labelled as ETAP library facts. Resolve `POSSIBLE_IDENTITY_MATCH` with
  the user; choose the existing record when it is the same product.
- Siemens BLH or QRH2 records without a target create an integrated fixed thermal-magnetic
  breaker record named by the family you state (`QRH2`, `BLH`), with the trip model without ETAP's
  decorations as `tripUnit`, the shared Ii multiple as `magneticFixedMult`, 240 V, three poles, and
  parts such as `QRH2 · 200 A`. Send one build per frame model; all its records must share one Ii
  multiple.

Ratings ending in A, E or C, or bare numbers, import as amperes when the record's rated current
equals that number. R ratings (`9R`) are not amperes and are left out, as are records with damaged
traces or a rated current that differs from the label. Report each left-out record and its reason
from the build's `limitations`. Each curve name keeps its own record's revision, such as
`LPJ-SP · 125 A minimum melt (ETAP 05/02)`. A native build can save up to 200 curves; accept them at
most 20 operations per revision.

## Process and review

1. Call `create_etap_catalog_import`. It returns a sealed build; no PDF upload or upload-ticket
   refresh is needed for this mode.
2. Call `process_catalog_import`, then `get_catalog_import`. Resume only when offered.
3. Resolve the complete current issue set using offered operations. Unsupported or ambiguous source
   semantics remain unresolved; preserve the input and report the specific gap.
4. Call `get_catalog_curve_review` for each distinct graph. Native source images are generated
   evidence cards, not ETAP screenshots or manufacturer pages. Compare each source/overlay pair
   with the original export and the separately downloaded manufacturer tripping document. When
   the manufacturer publishes only an average-melt curve and the record stores separate minimum
   melt and total clear, check that the average curve lies between them and report the currents
   compared at a stated time. The points stay as exported.
5. Submit `accept_curve` through `revise_catalog_import` only after verifying the current fingerprint
   and relevant evidence and getting explicit approval of each curve, from the user or from the
   reviewer they delegated to (SKILL.md, step 7). Reject incorrect
   interpretations; never edit point arrays in a revision or apply an extra tolerance to an already
   bounded band.
6. Verify `get_catalog_import` reports `applied`. Report catalogue links, saved curve count,
   record outcomes, settings coverage and remaining manufacturer-comparison discrepancies.

Import a product line in one build; separate builds that target the same series still run
sequentially.

Native ETAP and PDF inputs are both acceptable, but they have distinct source handling. A failure
to download a manufacturer PDF does not make the ETAP format unsupported. It leaves that independent
comparison unverified. Do not report a facts-only import as completion of a requested ETAP import.
