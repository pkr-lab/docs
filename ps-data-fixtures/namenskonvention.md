# Dateibenennungskonvention - IT-Ausbildung SharePoint

## Format

```
[Kürzel]_[Präfix]_[Suffix]_[Nummer].[Dateiformat]
```

### Komponenten

| Komponente | Beschreibung | Beispiel |
|------------|-------------|----------|
| **Kürzel** | Schülerkürzel (1. Vorname + 1. Nachname + 3. Nachname) | `PKE`, `XBu`, `LKc` |
| **Präfix** | Themenbereich/Programmiersprache | `Java`, `Python`, `Datenbanken` |
| **Suffix** | Spezifisches Konzept/Thema | `ArrayList`, `Dict`, `Query` |
| **Nummer** | 3-stellige laufende Nummer | `001`, `002`, `003` |
| **Dateiformat** | Dateityp | `pdf`, `docx` |

### Beispiele für Schülerkürzel

| Schüler/in | Kürzel | Erklärung |
|-----------|--------|----------|
| Peter Kretzer | PKE | **P**eter, **K**retzer, Kr**e**tzer |
| Xenia Bauer | XBu | **X**enia, **B**auer, Ba**u**er |
| Liam Koch | LKc | **L**iam, **K**och, Ko**c**h |
| Anna Müller | AMl | **A**nna, **M**üller, Mü**l**ler |
| Noah Bauer | NBu | **N**oah, **B**auer, Ba**u**er |

## Alle verfügbaren Präfixe und Suffixe

### 1. Java
**Suffixe:** ArrayList, HashMap, Collections, Streams, Lambda, Threads, JPA, Generics, Exceptions, Reflection

**Beispiele:**
- `PKE_Java_ArrayList_001.pdf`
- `XBu_Java_Lambda_002.docx`
- `LKc_Java_Threads_003.pdf`

---

### 2. Python
**Suffixe:** List, Dict, Tuple, Classes, Decorators, Modules, Generators, Context_Manager, Comprehensions, AsyncIO

**Beispiele:**
- `PKE_Python_List_001.docx`
- `XBu_Python_Dict_002.pdf`
- `LKc_Python_Classes_003.pdf`

---

### 3. Datenbanken
**Suffixe:** Query, Index, Trigger, View, Normalisierung, Constraints, Joins, Transactions, Backup, Replikation

**Beispiele:**
- `PKE_Datenbanken_Query_001.pdf`
- `XBu_Datenbanken_Joins_002.docx`
- `LKc_Datenbanken_Transactions_003.pdf`

---

### 4. Netzwerktechnik
**Suffixe:** TCP, UDP, DNS, DHCP, Firewall, VPN, Routing, SMTP, HTTP, SSL_TLS

**Beispiele:**
- `PKE_Netzwerktechnik_TCP_001.docx`
- `XBu_Netzwerktechnik_VPN_002.pdf`
- `LKc_Netzwerktechnik_Firewall_003.pdf`

---

### 5. Zahlensysteme
**Suffixe:** Binär, Hexadezimal, Oktal, Konvertierung, Bitoperationen, Overflow, Kodierung, Komplement, BCD, Floating_Point

**Beispiele:**
- `PKE_Zahlensysteme_Binär_001.pdf`
- `XBu_Zahlensysteme_Hexadezimal_002.docx`
- `LKc_Zahlensysteme_Bitoperationen_003.pdf`

---

### 6. PowerShell
**Suffixe:** Cmdlets, Loops, Functions, ErrorHandling, Modules, Registry, WMI, Remoting, JSON, Pipeline

**Beispiele:**
- `PKE_PowerShell_Cmdlets_001.pdf`
- `XBu_PowerShell_Functions_002.docx`
- `LKc_PowerShell_Pipeline_003.pdf`

---

## Regeln

1. ✅ **Präfix ist immer erforderlich** - Muss einer der 6 genannten Präfixe sein
2. ✅ **Suffix muss zum Präfix passen** - Verwende nur die festgelegten Suffixe für jeden Präfix
3. ✅ **Nummer ist 3-stellig** - Immer 001, 002, 003... (mit führenden Nullen)
4. ✅ **Dateiformat** - Nur `.pdf` oder `.docx`
5. ✅ **Groß-/Kleinschreibung** - Präfix mit Großbuchstaben anfangen, Suffixe wie aufgelistet (mit Unterstrichen bei Multi-Wort-Suffixen)
6. ✅ **Keine Leerzeichen** - Verwende Unterstriche als Trennzeichen

## Gültige Beispiele

| Beispiel | ✅ |
|----------|-----|
| `PKE_Java_ArrayList_001.pdf` | ✅ |
| `XBu_Python_Dict_002.docx` | ✅ |
| `LKc_Datenbanken_Query_003.pdf` | ✅ |
| `NBu_Netzwerktechnik_TCP_004.docx` | ✅ |
| `AMl_Zahlensysteme_Binär_005.pdf` | ✅ |
| `OZm_PowerShell_Cmdlets_006.docx` | ✅ |
| `Java_ArrayList_001.pdf` | ❌ (fehlendes Kürzel) |
| `PKE_Java_Context_Manager_007.pdf` | ❌ (falscher Suffix für Java) |
| `pke_java_arraylist_001.pdf` | ❌ (falscher Case) |
| `PKE_Java_ArrayList_1.pdf` | ❌ (Nummer nicht 3-stellig) |

## Zusammenfassung

- **6 Präfixe** verfügbar
- **10 Suffixe** pro Präfix
- **Unbegrenzte Nummernvergabe** (001, 002, ... 999+)
- **2 Dateiformate** (.pdf, .docx)
- **Maximale Klarheit und Konsistenz** in der Ablagestruktur

