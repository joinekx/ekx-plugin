# Download Siemens characteristics PDFs

Use this for public Siemens Industry Online Support document links. It uses ordinary public HTTP
requests only: no login, API key, browser session, or browser impersonation. If Siemens refuses a
request (HTTP 403, "Access Denied", a bot check, or a sign-in page), stop and ask the user to
download the PDF in their browser and attach it. Do not try to get around the refusal.

## Retrieve the real attachment

Take the numeric document ID from the user's URL. In one HTTP session that keeps cookies, open the
document page, initialize the public application, then request its document metadata. Use the
returned `PdfLink`, not a guessed filename based on the title. The filename can include a different
identifier, device details and a timestamp.

This sequence worked for document `109824381`. Adapt the document ID and working directory. Do not
print or save cookies or headers. On Windows, run `curl.exe`, not the PowerShell `curl` alias.

```sh
doc=109824381
origin=https://support.industry.siemens.com
jar=siemens-cookies.txt
page="$origin/cs/document/$doc?lc=en-WW"
xhr=(-H "Referer: $page" -H "X-Requested-With: XMLHttpRequest" -H "Accept: application/json")

curl --fail --silent --show-error --max-time 30 -c "$jar" -b "$jar" "$page" -o /dev/null
curl --fail --silent --show-error --max-time 30 -c "$jar" -b "$jar" "${xhr[@]}" \
  "$origin/webbackend/api/Initialization/InitializeClient?region=ww&language=en&networks=Internet" -o /dev/null
curl --fail --silent --show-error --max-time 30 -c "$jar" -b "$jar" "${xhr[@]}" \
  "$origin/webbackend/api/DocumentContents/DetailedDocument?localeGroupId=$doc&language=en&networks=Internet&region=ca" \
  -o metadata.json
```

Read `PdfLink`, `FileName`, `FileSizeInBytes`, and `Title` from `metadata.json`. Resolve `PdfLink`
against the origin, confirm it is an `https://support.industry.siemens.com/` URL, then download it
with the same cookie jar:

```sh
curl --fail --silent --show-error --max-time 60 -c "$jar" -b "$jar" "<resolved PdfLink>" -o "<FileName>"
```

Check that the file starts with `%PDF-` and that its byte count equals `FileSizeInBytes`. Then read
its page count, render and inspect it. Confirm the drawing number, model, rating, trip unit, pole
count and applicable conditions before using it as evidence. HTTP 200 or an application/pdf header
alone is insufficient. Upload the original unchanged bytes to EKX when the task calls for importing
curves; retrieving or inspecting a PDF does not accept its curves.

## Find a document from a model number

When no support document ID is known, initialize the same public session, then GET
`/webbackend/api/GlobalSearch/SearchDocuments` with `language=en`, `region=ca`,
`networks=Internet`, `$search=<model prefix>`, and `$top=100`. Inspect `Documents` entries with
`Type=Characteristic`: `Title` distinguishes tripping, let-through and I²t plots, while
`ForProductsText` gives applicability. Use the selected `LocaleGroupId` with `DetailedDocument`
as above, then verify the actual PDF. Do not treat `HitCount` as the number of returned entries.

For example, searching `3VA5240` found the three/four-pole TM210 40 A tripping drawing
L1V30040333872A DS03, document `109773560`. The original PDF confirms 40 A rated current and
400 A instantaneous pickup. Separate two-pole, let-through and I²t documents also appeared; they
are not substitutes for the requested three-pole time-current curve.

## Limits

- Preserve `networks=Internet` capitalization. The sequence uses worldwide English for
  initialization and Canadian English for document metadata.
- A title-derived attachment URL can return 410 even when the real PDF is current. Rediscover
  `PdfLink` through metadata. A SIMARIS exact-product 404 is a separate lookup failure.
- Keep TLS verification on. Never use `curl -k` or disable certificate checks.
- If a step fails, report the failing step and HTTP status and ask the user for the PDF. Do not
  loop, and do not work around a sign-in, entitlement, or bot-protection response.

## Verified example

Document `109824381`, drawing `L1V30040333880A DS01`, returned:

`/cs/attachments/109824381/880aUD_01_I-t_3VA5235_TM210_LI_35A_3pol_202309131047131146.pdf`

The original was one page, 297,669 bytes, SHA-256
`0490e9afda4c33f31a2ad303958ec55a3965ec5e2f32dcb0ccab27ae62b5ae57`.
It explicitly labels a 35 A rating and 400 A instantaneous pickup with ±20% pickup tolerance.
These are facts for that drawing, not defaults for other Siemens breakers. When discussing a
conflict, distinguish rated current from instantaneous pickup and frame size.

## Verified instantaneous-only example

Searching `3VA5440` returned both document `109773565` (UL amperes-axis plot, drawing
`L1V30040333913ADS01`) and `109770981` (multiplier-axis plot, drawing `L1V30040332913A DS02`) for
`3VA5440- . MH`, TM120M1. The UL PDF explicitly identifies three poles, a 600 A frame, 400 A trip
rating, HLAP/CLAP applicability, and instantaneous pickup 2400–4800 A in 400 A steps with ±20%
tolerance. Its gray instantaneous band distinguishes minimum tripping time from maximum clearing
time. The descending high-time trace is explicitly a **damage curve**, requiring upstream overload
protection; it must not become a thermal trip curve merely because the document title says
tripping characteristics.

The original UL PDF was 332,422 bytes, SHA-256
`9c3ca6c2e93cfc03db55327b281553dea47c7fd571e9dce7b413c61dbdb270ae`, returned as:

`/cs/attachments/109773565/pdf_913rUD_01_I-t_3VA5440_TM120M1_I_400A_201912031332243302.pdf`
