---
name: Fehler in einer Publikation
about: Fehler in einer DATEX-II-Publikation melden, z. B. Schema-Validierungsfehler oder unplausible Daten
title: '<Herausgeber>: <Publikation> – <Kategorie, z. B. Schema-Validierungsfehler>'
---

<!--
Titel: kurz halten, nur Kategorie statt einzelner Felder oder Fehler, z. B.
"Stadt Erfurt: Parkierungseinrichtungen – Schema-Validierungsfehler" oder
"Bosch: Secure Truck Parking – Koordinaten vertauscht".

Anleitung zum Prüfen und Validieren: docs/validating-publications.md
-->

| | |
|---|---|
| Herausgeber | <Herausgeber laut Angebot> |
| Angebote | statisch [`<Angebots-ID>`](https://mobilithek.info/offers/<Angebots-ID>) · Echtzeit [`<Angebots-ID>`](https://mobilithek.info/offers/<Angebots-ID>) |
| Format | <z. B. DATEX II 2.3, ITP-Profil (`ITP_Deutschland_ParkingTable.xsd` / `ITP_Deutschland_ParkingStatus.xsd`)> |
| Geprüft | `publicationTime` <statisch …, Echtzeit …> |
| Nutzbar | <❌ Nein / ⚠️ Eingeschränkt / ✅ Ja>: <ein Satz, warum> |

## Probleme

<!--
Eine Zeile pro Problem, wichtigstes zuerst. Schweregrade:
- kritisch: Daten sind nicht nutzbar
- Schema:   Dokument ist ungültig gegen das Schema
- Profil:   falsches oder fehlendes Profil bzw. Schemaangabe
- prüfen:   unplausibel, sollte vom Herausgeber geprüft werden
- Hinweis:  Verbesserungsvorschlag
Behobene Probleme werden abgehakt.
-->

- [ ] **P1 · <Schweregrad>** · <Problem in einem Satz, mit Anzahl betroffener Einträge, z. B. (13 von 13)>
- [ ] **P2 · <Schweregrad>** · <Problem in einem Satz>

<!--
Ein <details>-Block pro Problem. Keine vollständigen Listen aller betroffenen Einträge:
Anzahl nennen und ein Beispiel zeigen, die Prüfung findet alle weiteren.
-->

<details>
<summary><b>P1</b> · <Kurztitel></summary>

**Wo:** <XPath bzw. Elementpfad, Dokument (statisch / Echtzeit), Anzahl betroffener Einträge>

**Warum:** <Was ist falsch: Schemaregel oder Plausibilitätsgrund>

```
<xmllint-Fehlermeldung, falls vorhanden>
```

**Beispiel:** <Eintrag, ggf. mit Adresse oder Kontext>

```xml
<wenige relevante Zeilen aus der Publikation>
```

**Lösung:**

```xml
<korrigierte Zeilen>
```

**Prüfung:** erwartet <leer / 0 / true / false>, aktuell <Ergebnis>

```xpath
<XPath-Ausdruck mit local-name()>
```

</details>

<details>
<summary><b>P2</b> · <Kurztitel></summary>

**Wo:** …

**Warum:** …

**Lösung:** …

**Prüfung:** …

</details>

---
<sub>Validiert mit `xmllint` gegen <Schema mit Link>. <Bei mehreren Fehlern: xmllint meldet je Inhaltsmodell nur den ersten Fehler, daher wurde nach jeder Korrektur erneut validiert. Mit den Korrekturen P… sind die Dokumente gültig.> Anleitung: [Validating DATEX II publications](https://github.com/datex2-tools/datex2-publications-issues/blob/main/docs/validating-publications.md)</sub>
