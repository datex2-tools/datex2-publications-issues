# Validating DATEX II publications

This guide explains how to reproduce the checks documented in the issues of this repository: schema validation with
`xmllint` and the XPath expressions listed under **Prüfung** / **Automatisierte Prüfung**. It focuses on publications
from the [Mobilithek](https://mobilithek.info), the German national access point, but the commands work for any DATEX II
document.

## Prerequisites

- `xmllint` from libxml2 (Debian / Ubuntu: `apt install libxml2-utils`, macOS: preinstalled)
- `curl`
- Python 3 (only for downloading schemas that consist of several files, see below)

## 1. Get the publication

### Mobilithek subscription

Most Mobilithek offers require a subscription, even when they are open data. Once the subscription is approved, the
data is fetched with the client certificate of your Mobilithek organisation. Note that the URL uses the **subscription
ID**, not the offer ID:

```bash
SUB=1234567890

# DATEX II 2.x
curl --compressed --cert cert.pem --key key.pem -o static.xml \
  "https://mobilithek.info:8443/mobilithek/api/v1.0/subscription/$SUB/clientPullService?subscriptionID=$SUB"

# DATEX II 3.x
curl --compressed --cert cert.pem --key key.pem -o static.xml \
  "https://mobilithek.info:8443/mobilithek/api/v1.0/subscription/datexv3?subscriptionID=$SUB"

# Mobilithek container format
curl --compressed --cert cert.pem --key key.pem -o container.xml \
  "https://mobilithek.info:8443/mobilithek/api/v1.0/container/subscription?subscriptionID=$SUB"
```

Note the `publicationTime` of the document you checked, so that others can tell whether the data has changed since:

```bash
xmllint --xpath "string(//*[local-name()='publicationTime'])" static.xml
```

### Unwrap container documents

In the container format the DATEX II document is the single child of `container/body/xml`, and the `schema` attribute
of the `xml` element names the declared schema:

```bash
xmllint --xpath "string(//*[local-name()='body']/*[local-name()='xml']/@schema)" container.xml
xmllint --xpath "/*[local-name()='container']/*[local-name()='body']/*[local-name()='xml']/*" container.xml > static.xml
```

Validate the unwrapped `static.xml`, not the container. If the unwrapped document uses a namespace prefix that was
only declared in the container (error `Namespace prefix ... is not defined`), add the declaration to its root element.

### Make sure it is XML at all

Some offers declared as DATEX II XML deliver something else, e.g. JSON. `xmllint` then fails before any validation:

```bash
xmllint --noout static.xml
# static.xml:1: parser error : Start tag expected, '<' not found
head -c 200 static.xml
```

### Format the document

Many publications are written on a single line, so every error reports line 1. Format the document first to get useful
line numbers:

```bash
xmllint --format static.xml > static-formatted.xml
```

## 2. Find the right schema

A document can be checked against up to three schema declarations. They should match, and the issues report when they
don't:

| Source | How to read it |
|---|---|
| Mobilithek offer | `curl -s https://mobilithek.info/mdp-api/mdp-msa-metadata/v2/offers/<offer-id>`, fields `contentData[].masterSchema.accessURL` (entry point) and `contentData[].schema[].accessURL` (further files). The paths are relative to `https://mobilithek.info/mdp-api/files/aux`. Only works anonymously for public offers. |
| Document | `xmllint --xpath "string(/*/@*[local-name()='schemaLocation'])" static.xml`, plus `extensionName` / `extensionVersion` on `d2LogicalModel` for MDM profiles |
| Container | `schema` attribute of `container/body/xml`, see above |

If none of them is given or they contradict the content, use the schema that matches the root and publication elements,
see the table below.

### Schemas used in the issues

All URLs are relative to `https://mobilithek.info/mdp-api/files/aux/`.

| Profile | Content | Entry point |
|---|---|---|
| DATEX II 2.3 | any 2.x publication | `schemas/DATEX_2/Version_2.3/DATEXIISchema_2_2_3.xsd` |
| ITP Lkw-Parken, static | `parkingTablePublication` | `schemas/DATEX_2/Profile_Intelligent_Truck_Parking_Site/ITP_Deutschland_ParkingTable.xsd` |
| ITP Lkw-Parken, realtime | `parkingStatusPublication` | `schemas/DATEX_2/Profile_Intelligent_Truck_Parking_Status/ITP_Deutschland_ParkingStatus.xsd` |
| MDM ParkingFacility, static | `parkingFacilityTablePublication` | `110000000002025002/MDM-Profile_ParkingFacilityTable.xsd` |
| MDM ParkingFacility, realtime | `parkingFacilityTableStatusPublication` | `110000000002025001/MDM-Profile_ParkingFacilityStatus.xsd` |
| Parkraummanagement DATEX II v3, Grunddaten | `payload` with `xsi:type="ParkingTablePublication"` | `schemas/DATEX_2_V3/Parkraummanagement_DATEX_II_v3_Grunddaten/DATEXII_3_D2Payload.xsd` |
| Parkraummanagement DATEX II v3, Belegungsdaten | `payload` with `xsi:type="ParkingStatusPublication"` | `schemas/DATEX_2_V3/Parkraummanagement_DATEX_II_v3_Belegungsdaten/DATEXII_3_D2Payload.xsd` |

The MDM profiles have no central location in the Mobilithek. The ones listed are the copies uploaded with the offers of
the city of Wuppertal, which are publicly accessible. Copies at other offers are often not accessible (HTTP 403 or
empty).

Publishers may also upload their own schemas with an offer, see `contentData[].schema` in the offer metadata. If they
are used, say so in the issue.

## 3. Download the schemas

`xmllint` cannot load schemas over HTTPS, so download them first. Single-file schemas (DATEX II 2.x, ITP, MDM) can be
fetched directly:

```bash
mkdir -p schemas/mdm
curl -s -o schemas/mdm/MDM-Profile_ParkingFacilityTable.xsd \
  https://mobilithek.info/mdp-api/files/aux/110000000002025002/MDM-Profile_ParkingFacilityTable.xsd
```

DATEX II 3.x schemas consist of several files that import each other (`DATEXII_3_D2Payload.xsd` imports
`DATEXII_3_Parking.xsd`, `DATEXII_3_Common.xsd`, …). The files of a profile must be taken from the same directory,
because each profile contains a reduced version of the shared files. This script downloads an entry point and,
recursively, all relatively imported files into one directory:

```python
#!/usr/bin/env python3
"""
Download an XSD and, recursively, everything it imports or includes via a relative schemaLocation.
Usage: fetch_xsd.py <url> <target-dir>
"""

import sys
import urllib.request
import xml.etree.ElementTree as ET
from pathlib import Path
from urllib.parse import urljoin

XS = '{http://www.w3.org/2001/XMLSchema}'


def fetch(url: str, base_url: str, target: Path, seen: set[str]) -> None:
    if url in seen:
        return
    seen.add(url)
    path = target / url.removeprefix(base_url)
    path.parent.mkdir(parents=True, exist_ok=True)
    with urllib.request.urlopen(url) as response:
        path.write_bytes(response.read())
    print(f'{url} -> {path}')
    root = ET.parse(path).getroot()
    for element in root.iter():
        location = element.get('schemaLocation')
        if element.tag in (f'{XS}import', f'{XS}include', f'{XS}redefine') and location:
            if location.startswith(('http://', 'https://')):
                print(f'  absolute import, not fetched: {location}')
                continue
            fetch(urljoin(url, location), base_url, target, seen)


if __name__ == '__main__':
    entry_url = sys.argv[1]
    fetch(entry_url, entry_url.rsplit('/', 1)[0] + '/', Path(sys.argv[2]), set())
```

```bash
python3 fetch_xsd.py \
  https://mobilithek.info/mdp-api/files/aux/schemas/DATEX_2_V3/Parkraummanagement_DATEX_II_v3_Grunddaten/DATEXII_3_D2Payload.xsd \
  schemas/pr-grunddaten
```

If the script reports an absolute import, download that file as well and change the `schemaLocation` in the importing
file to the local path.

## 4. Validate with xmllint

```bash
xmllint --noout --nonet --schema schemas/mdm/MDM-Profile_ParkingFacilityTable.xsd static-formatted.xml
```

- `--noout` suppresses the document output, `--nonet` prevents any network access.
- `--schema` overrides the document's `xsi:schemaLocation`, so you always validate against the file you choose.
- Exit code `0`: `static-formatted.xml validates`. Exit code `3`: `fails to validate`. Exit code `5`: the schema
  itself could not be loaded or compiled, e.g. a missing imported file.

Example output:

```
static-formatted.xml:3: element payloadPublication: Schemas validity error : Element '{http://datex2.eu/schema/2/2_0}payloadPublication': This element is not expected. Expected is ( {http://datex2.eu/schema/2/2_0}exchange ).
static-formatted.xml fails to validate
```

### Find all errors

`xmllint` reports only the **first** error per content model. Once an element does not match, its siblings and
descendants are no longer checked, so a document with seven errors may show only one. To get the complete list, as
documented in the issues:

1. Copy the document.
2. Fix the reported error in the copy (add the missing element, remove the unexpected one, …).
3. Validate the copy again and repeat until it validates.

The list of fixes you needed is the list of errors. It also proves that the suggested solution in the issue results in
a valid document.

## 5. Run the XPath checks

The issues list XPath expressions for each problem, together with the expected and the current result. Run them with
`xmllint --xpath`:

```bash
xmllint --xpath "//*[local-name()='parkingFacility'][not(@version)]" static.xml
```

The expressions use `local-name()` instead of namespace prefixes, because `xmllint --xpath` cannot register
namespaces. They therefore work for DATEX II 2.x and 3.x regardless of the prefixes used in the document.

How to read the result:

| Expected in the issue | Command | Result if fixed |
|---|---|---|
| "erwartet leer" (node set) | `xmllint --xpath "<expr>" file.xml` | `XPath set is empty`, exit code `10` |
| number of hits | `xmllint --xpath "count(<expr>)" file.xml` | `0` |
| `true` / `false` | `xmllint --xpath "<expr>" file.xml` | the expected boolean |

For node sets, wrap the expression in `count()` to get the number of affected elements instead of the elements
themselves:

```bash
xmllint --xpath "count(//*[local-name()='pointCoordinates'][*[local-name()='latitude'] < 47 or *[local-name()='latitude'] > 56])" static.xml
# 26
```

Compare the number with the issue: it should be the "aktuell" count while the problem exists and drop to `0` once it is
fixed.

To run all checks of an issue against both documents:

```bash
for expr in \
  "//*[local-name()='parkingFacility'][not(@version)]" \
  "//*[local-name()='parkingFacilityReference'][not(@targetClass)]"
do
  for file in static.xml realtime.xml; do
    echo "$file: $(xmllint --xpath "count($expr)" "$file") × $expr"
  done
done
```

## 6. Writing a new check

When you report a new problem, add a check that others can run without additional tools:

- Prefer XPath 1.0 with `local-name()`, so it runs with plain `xmllint`.
- Phrase it so that the expected result is empty, `0` or a fixed boolean, and state the current result.
- Test it against the current publication before posting, including the count you report.
- For plausibility problems that the schema allows (e.g. swapped coordinates, a coordinate range is just a `Float` in
  DATEX II 2.x), use a range that clearly separates correct from wrong values, e.g. latitude 47 … 56 for Germany.
