# Xerces-C++ Input Data-Flow Analysis

This document traces **every kind of input** that flows into Xerces-C++ and
follows each from the outer surface down to where it is consumed. The goal is
to give you a concrete feel for the library's attack/trust surface and its
internal plumbing before you refactor it. Read it alongside `ARCHITECTURE.md`
(which covers layering); this file is specifically about *where bytes and
configuration enter and how they propagate*.

Every path below is annotated with **trust level** — whether the data is
attacker-controllable in a typical deployment (parsing an untrusted document)
— because that determines what a refactor must not weaken.

---

## 0. The input taxonomy at a glance

Xerces takes input through eight distinct channels:

| # | Channel | Enters via | Trust (parsing untrusted XML) |
|---|---------|-----------|-------------------------------|
| 1 | **Primary XML document** | `parse(InputSource/systemId)` | Untrusted |
| 2 | **Referenced entities & sub-resources** (external DTD subset, parameter/general entities, `xsi:schemaLocation`, `xs:import`/`include`, XInclude `href`) | resolved *during* the parse via system ids | Untrusted (the document chooses them) |
| 3 | **Character encoding declaration** (BOM + `<?xml encoding?>`) | sniffed by `XMLReader` | Untrusted (drives transcoder selection) |
| 4 | **Application-supplied handlers/resolvers** | SAX/DOM callback interfaces + `EntityResolver`/`XMLEntityResolver` | Trusted (app code), but they *return* untrusted streams |
| 5 | **Parser configuration** (features, properties, `SecurityManager`, external schema location, grammar caching flags) | setter APIs before `parse()` | Trusted |
| 6 | **Pre-loaded / cached grammars** (binary-serialized `.grammar` blobs, `loadGrammar()`) | `XMLGrammarPool` deserialization | **As trusted as their source** — see §7 |
| 7 | **Network responses** (HTTP/HTTPS/FTP) | `XMLNetAccessor` backends | Untrusted (remote server) |
| 8 | **Process environment** (env vars, locale, filesystem) | `getenv`, file managers, message loaders | Deployment-controlled |

The rest of the document walks each channel.

---

## 1. Channel 1 — The primary XML document

### 1.1 The entry funnel

Everything an application feeds in is normalized to an **`InputSource`**
(`sax/InputSource.hpp`), whose one required job is to hand back a
**`BinInputStream`** — a raw byte source — through the pure-virtual
`makeStream()`:

```
Application call                         Concrete InputSource        Concrete BinInputStream
─────────────────                        ─────────────────────       ───────────────────────
parse("file.xml")           ──┐
parse("http://…/x.xml")     ──┼─► XMLScanner builds a
parse("ftp://…")            ──┘   LocalFileInputSource / URLInputSource
                                  based on the system id

parse(LocalFileInputSource) ────► LocalFileInputSource ─────► BinFileInputStream   (util/FileManagers)
parse(MemBufInputSource)    ────► MemBufInputSource    ─────► BinMemInputStream     (wraps app buffer)
parse(URLInputSource)       ────► URLInputSource       ─────► net accessor stream   (§7) or BinFileInputStream for file://
parse(StdInInputSource)     ────► StdInInputSource     ─────► BinFileInputStream on stdin
DOMLSParser (DOM LS)        ────► Wrapper4DOMLSInput   ─────► wraps a DOMLSInput (byte stream OR in-memory string)
```

Key files:
- `framework/LocalFileInputSource.{hpp,cpp}` → `util/BinFileInputStream`
  (backed by `util/FileManagers/{Posix,Windows}FileMgr`).
- `framework/MemBufInputSource.{hpp,cpp}` → `util/BinMemInputStream`. The
  buffer may be *adopted* or *borrowed* (`fAdoptBuffer`) — an ownership
  subtlety that matters if you refactor lifetime handling.
- `framework/URLInputSource.{hpp,cpp}` → dispatches on protocol (§7).
- `framework/StdInInputSource.{hpp,cpp}`.
- `framework/Wrapper4InputSource` / `Wrapper4DOMLSInput` — adapters so the
  DOM-LS `DOMLSInput` surface (`dom/DOMLSInput.hpp`: `getByteStream()`,
  `getStringData()`, `getEncoding()`, `getSystemId()`) folds into the same
  `InputSource`/`BinInputStream` machinery.

**`BinInputStream`** (`util/BinInputStream.hpp`) is the universal narrow
waist: a `readBytes(buf, max)` + `curPos()` interface. Every byte the parser
will ever see passes through some `BinInputStream::readBytes`. This is the
single best instrumentation/choke point in the whole library.

### 1.2 From bytes to characters — the `XMLReader`

Raw bytes never reach the scanner. `ReaderMgr::createReader`
(`internal/ReaderMgr.cpp`) wraps each `BinInputStream` in an
**`XMLReader`** (`internal/XMLReader.{hpp,cpp}`), which is where the most
security-relevant transformation happens. Buffer geometry
(`internal/XMLReader.hpp`):

```
BinInputStream ──readBytes──►  fRawByteBuf[48 KB]  ──transcode──►  fCharBuf[16 KB XMLCh]
                               (raw bytes)            (UTF-16)       + fCharSizeBuf / fCharOfsBuf
                                                                     (per-char byte width & offset,
                                                                      for accurate error columns)
```

- `refreshRawBuffer()` pulls the next raw chunk; `xcodeMoreChars()` runs the
  transcoder to refill `fCharBuf`.
- The scanner pulls one logical character at a time from the `XMLReader`;
  character classification (name-start, whitespace, etc.) is table-driven
  (`internal/CharTypeTables.hpp`).
- `fCharSizeBuf`/`fCharOfsBuf` preserve source byte positions so line/column
  in errors are correct even after transcoding — don't discard these in a
  refactor of the reader.

### 1.3 Channel 3 — encoding detection (inside the XMLReader)

Encoding selection is a small state machine and is **input-driven**, so it is
part of the untrusted surface:

1. `framework/XMLRecognizer.cpp::basicEncodingProbe` inspects the first bytes
   for a BOM / the ASCII/UCS byte pattern of `<?xml`. The coarse result is
   one of `XMLRecognizer::Encodings` — `EBCDIC, UCS_4B, UCS_4L, US_ASCII,
   UTF_8, UTF_16B, UTF_16L, XERCES_XMLCH` (`framework/XMLRecognizer.hpp:57`).
2. Using that provisional decoding, the reader parses the actual
   `<?xml version="…" encoding="…"?>` text declaration.
3. The declared encoding name is resolved to an `XMLTranscoder` via
   `XMLPlatformUtils::fgTransService` (ICU / Iconv / Win32 / intrinsic —
   see `ARCHITECTURE.md §4.2`). An explicit `InputSource::setEncoding()`
   from the application **overrides** auto-detection.
4. An unknown/unsupported encoding → fatal error. A declared encoding that
   contradicts the BOM is handled per XML rules.

Refactor note: the probe + text-decl parse + transcoder handoff are spread
across `XMLReader.cpp`, `XMLRecognizer.cpp`, and `TransService.cpp`; this is
a coherent extractable unit ("encoding negotiation") and a good early
refactoring target, but it is directly attacker-reachable — cover it with
tests using deliberately malformed/mislabeled encodings first.

### 1.4 The entity/reader stack

A document is rarely one stream. Each entity that gets expanded (the document
entity, the external DTD subset, every external and internal parsed entity)
becomes **another `XMLReader` pushed onto `ReaderMgr`'s stack**. The scanner
always reads from the top; when a reader hits EOF, `EndOfEntityException`
unwinds back to the enclosing reader. This is how one logical document tree
is assembled from many byte sources, and it is the mechanism that entity
expansion attacks abuse (§2, §6).

### 1.5 Progressive (pull) input

The same funnel serves the progressive API: `parseFirst()` returns an
`XMLPScanToken`, and repeated `parseNext(token)` (`internal/XMLScanner.hpp`,
`scanNext`) drives the scanner incrementally over the *same* reader stack.
No separate input path — just a different way of clocking it.

---

## 2. Channel 2 — Referenced entities and sub-resources

Once scanning starts, **the document itself names more inputs**, and Xerces
fetches them. This is the richest and most dangerous part of the input graph
because the *document* (untrusted) chooses the *resources*.

### 2.1 What can pull in another resource

| Construct | Where handled | Resource named by |
|-----------|---------------|-------------------|
| External DTD subset (`<!DOCTYPE x SYSTEM "…">`) | `validators/DTD/DTDScanner.cpp` | system/public id |
| External parameter entity (`%pe;`) | DTD scanner + `ReaderMgr` | system id |
| External general entity (`&ge;`) | scanner content path | system id |
| `xsi:schemaLocation` / `xsi:noNamespaceSchemaLocation` | schema scanners (`SGXMLScanner`/`IGXMLScanner`) | URI list in the instance |
| `xs:import` / `xs:include` / `xs:redefine` | `validators/schema/TraverseSchema.cpp` | `schemaLocation` attr |
| XInclude `<xi:include href="…">` | `xinclude/XIncludeUtils.cpp` | href (post-DOM pass) |

### 2.2 The resolution pipeline (system id → InputSource)

All of the above converge on the scanner's entity-resolution code
(`internal/XMLScanner.cpp`):

```
system id (relative or absolute)
   │
   ├─► expandSystemId()  ── resolve against the base URI of the *current* entity
   │        (XMLEntityHandler::expandSystemId, framework/XMLEntityHandler.hpp)
   │
   ├─► application resolver hook (Channel 4), if installed:
   │        fEntityHandler->resolveEntity(&resourceIdentifier)   [XMLScanner.cpp:1687]
   │        (SAX EntityResolver OR the richer XMLEntityResolver with an
   │         XMLResourceIdentifier describing what/why is being resolved)
   │        └─ returns an InputSource, OR null to fall through
   │
   └─► default:  resolveSystemId(sysId, pubId)   [XMLScanner.cpp:1840]
            └─ build LocalFileInputSource / URLInputSource from the resolved URI
```

The returned `InputSource` re-enters the **exact funnel from §1** — new
`BinInputStream`, new `XMLReader`, pushed on the stack. So sub-resources get
the same encoding detection, the same buffering, recursively.

### 2.3 Security controls on this channel (do not regress)

This channel is the classic XXE / SSRF / billion-laughs surface. The
existing mitigations, which any refactor must preserve:

- **`SecurityManager`** (`util/SecurityManager.hpp`) caps entity expansion:
  `ENTITY_EXPANSION_LIMIT = 50000` default (`SecurityManager.hpp:55`),
  settable via `setEntityExpansionLimit()`. The scanner counts expansions
  and fatals out past the cap — the billion-laughs defense.
- **DOCTYPE / DTD disabling**: recent commits added programmatic
  `disallow-doctype`-style controls to the SAX parsers (and the
  `XERCES_DISABLE_DTD` env var short-circuit at `XMLScanner.cpp:1279`).
  Disabling DTDs is the recommended XXE hardening.
- **Application resolver veto**: an `XMLEntityResolver` receives an
  `XMLResourceIdentifier` (kind = external entity / schema / DTD / etc.)
  *before* fetching, so app code can block or redirect any external fetch.
  This is the primary user-facing XXE control — keep the hook and the
  metadata it passes intact.
- The **CVE-2018-1311** use-after-free in external-DTD scanning was fixed in
  this tree; the entity-resolution refactor must not reintroduce a dangling
  `InputSource`/reader.

### 2.4 XInclude specifics

XInclude runs as a **post-parse DOM pass** (`xinclude/`), not through the
streaming scanner. `XIncludeLocation` resolves `href` against the including
document's base URI and reads the target via the same `InputSource`
machinery; `XIncludeUtils` handles fallback, `xpointer` (limited), and
recursion detection. Because it is a separate pass, its resource fetching is
governed by whatever `InputSource`/net setup the DOM parser uses, not by the
scanner's `SecurityManager` counters — worth noting for a threat model.

---

## 3. Channel 4 — Application handlers and resolvers

These are **trusted inbound interfaces** (the app implements them), but they
are load-bearing for the data flow because they inject or gate other inputs:

- **Content sinks** (data flows *out* to them, but they can throw back in):
  SAX `ContentHandler`/`DocumentHandler`, `DTDHandler`, `LexicalHandler`,
  `DeclHandler`; DOM building in `AbstractDOMParser`. Exceptions thrown from
  handlers propagate up through the scanner and must leave it consistent.
- **`ErrorHandler`** — receives `SAXParseException`s; may escalate a warning
  to fatal by throwing.
- **`EntityResolver` / `XMLEntityResolver`** — the resource gate from §2.2.
- **`GrammarPool`** — supplies cached grammars (§6).
- **`MemoryManager`** — every allocation is routed to the app-supplied one
  (`ARCHITECTURE.md §5.2`); a custom manager sees every input-sized
  allocation.

The important dataflow property: **handler callbacks carry pointers into
scanner-owned scratch buffers** (e.g. attribute lists, character runs in
`XMLBuffer`). Those are valid only for the duration of the callback. Any
refactor changing buffer lifetimes has to preserve this contract or silently
corrupt user code.

---

## 4. Channel 5 — Parser configuration input

Set before `parse()` on the façade classes (`parsers/*`) and forwarded into
the scanner. These are pure configuration but they *reshape the data flow*:

- **Validation scheme / namespaces / schema** — selects which of the four
  scanners runs (`XMLScannerResolver`; see `ARCHITECTURE.md §4.3`) and
  therefore which sub-resource fetches (§2) can happen at all.
- **`setExternalSchemaLocation` / `setExternalNoNamespaceSchemaLocation`**
  (`XMLScanner.hpp:374`) — injects schema URIs *from the application* that
  are treated like an instance-supplied `schemaLocation`. Note these strings
  are `replicate`d/`transcode`d and then resolved through the same
  sub-resource pipeline (§2).
- **SAX/DOM features & properties** — large flat sets of getters/setters
  (`SAX2XMLReaderImpl::setFeature/setProperty`, `DOMConfiguration` for
  DOM-LS). This includes the security knobs from §2.3.
- **`SecurityManager`, `LowWaterMark`, buffer sizes** — throttle inputs.

Trust: trusted, but note that `schemaLocation`-style config causes outbound
fetches, so in a hostile-network deployment even "config" can be an SSRF
lever if wired to untrusted values.

---

## 5. Channel 8 — Process environment inputs

Easy to overlook; all read via `getenv`/locale/filesystem:

- `XERCES_DISABLE_DTD` — global DTD kill switch (`XMLScanner.cpp:1279`).
- `XERCESC_NLS_HOME` / `XERCESCROOT` — where the ICU/MsgCatalog message
  loaders find diagnostic text (`util/MsgLoaders/*/*.cpp`).
- `LC_ALL` / `LC_CTYPE` / `LANG` — locale for the IconvGNU transcoder
  (`IconvGNUTransService.cpp:421`).
- The filesystem itself, via `FileManagers`, for every `file:`/local path.

Trust: deployment-controlled. In a hardened service you generally want DTDs
off and message/locale env pinned.

---

## 6. Channel 7 — Network responses in detail

When a system id resolves to `HTTP`/`HTTPS`/`FTP` (`XMLURL::Protocols`,
`util/XMLURL.hpp:44`), `URLInputSource` delegates to the build-selected
`XMLNetAccessor` backend (`util/NetAccessors/`):

| Backend | Files |
|---------|-------|
| libcurl | `NetAccessors/Curl/CurlURLInputStream.cpp` |
| Raw sockets | `NetAccessors/Socket/` |
| WinSock | `NetAccessors/WinSock/` |
| macOS CFURL | `NetAccessors/MacOSURLAccessCF/` |
| shared HTTP logic | `NetAccessors/BinHTTPInputStreamCommon.cpp` (`sendRequest`, status handling) |

Data-flow facts worth knowing before refactoring:

- The curl backend **follows redirects** (`CURLOPT_FOLLOWLOCATION`, up to
  `CURLOPT_MAXREDIRS = 6`, `CurlURLInputStream.cpp:88`) — a redirect can
  move a fetch to a different host/scheme; relevant to SSRF reasoning.
- **Credentials in URLs**: `XMLURL::getUser()/getPassword()` are extracted
  and passed to curl (`CURLOPT_HTTPAUTH`, `CURLOPT_USERPWD`,
  `CurlURLInputStream.cpp:92-103`). Document-supplied URLs can therefore
  carry credentials.
- The network stream is a plain `BinInputStream`, so once bytes arrive they
  rejoin the §1.2 reader/transcode path — no special trust handling
  downstream. Whatever a remote server returns is parsed like a local file.
- Backend selection is build-time; there is no per-parse "no network" flag
  short of installing an `XMLEntityResolver` that refuses remote ids or
  building without a net accessor. (A candidate improvement: a first-class
  runtime "disable network" switch.)

---

## 7. Channel 6 — Binary grammar deserialization (the subtle one)

Xerces can **serialize compiled grammars** to a binary blob and reload them
to skip schema compilation (`framework/XMLGrammarPoolImpl.hpp`:
`serializeGrammars()` / `deserializeGrammars()`, format versioned by
`GRAMMAR_SERIALIZATION_LEVEL=7`). This is a *separate input channel with its
own parser* — `internal/XSerializeEngine.cpp` — that reads sizes, object
tags (`XProtoType`), strings, and reconstructs the full grammar/validator
object graph (`XTemplateSerializer` knows every serializable type).

Why it matters for this analysis:

- The deserializer reads length-prefixed data (`readSize`, `read(...)` in
  `XSerializeEngine.hpp:272-522`) and allocates/constructs objects from it.
  **A malformed or malicious grammar blob is as dangerous as a malicious
  document — arguably more, since it drives object construction directly.**
  Treat cached-grammar input as trusted only if its source is trusted.
- Any change to data members of a serializable grammar/validator/datatype
  class silently changes this binary format. Refactors here must bump the
  serialization level and/or keep read/write in lockstep, or previously
  cached grammars will deserialize into corrupt objects.
- `loadGrammar()` on the parsers is the other way grammars enter: it parses a
  `.dtd`/`.xsd` (via the §1/§2 funnel) and *caches* the result in the
  `GrammarPool` for reuse across documents.

---

## 8. Consolidated data-flow diagram

```
                        ┌─────────────────────────── Application ───────────────────────────┐
                        │ parse(id/InputSource)   setFeature/Property   EntityResolver        │
                        │ loadGrammar()           SecurityManager       GrammarPool           │
                        └───────┬───────────────────────┬───────────────────┬────────────────┘
                                │ (Ch.1)                 │ (Ch.5)            │ (Ch.4/6)
                                ▼                         ▼                   │
                        ┌───────────────┐        parser façade (parsers/)    │
   env vars (Ch.8) ────►│  InputSource  │◄───────── config forwarded ────────┤
   getenv/locale        │  makeStream() │                                     │
                        └──────┬────────┘                                     │
                               ▼                                              │
   HTTP/FTP (Ch.7) ───► BinInputStream ◄─── file / mem / stdin                │
   NetAccessors          (util/*InputStream)                                  │
                               │                                              │
                               ▼                                              │
                    ┌──────────────────────┐   encoding probe + <?xml?>       │
   Ch.3 encoding ──►│  XMLReader (48K raw   │   (XMLRecognizer / TransService) │
                    │  → 16K XMLCh UTF-16)  │                                  │
                    └──────────┬───────────┘                                  │
                               ▼                                              │
                    ┌──────────────────────┐                                  │
                    │  ReaderMgr  (stack)   │◄── pushes a new XMLReader per    │
                    │                       │    entity/sub-resource (Ch.2) ───┤ resolveEntity()
                    └──────────┬───────────┘                                  │  (XMLScanner.cpp:1687/1840)
                               ▼                                              │
                    ┌──────────────────────┐                                  │
                    │   XMLScanner (1 of 4) │── validation ──► DTD/Schema ◄────┘ grammar
                    │   ElemStack, XMLBuffer│                  validators       (Ch.6 deserialize
                    └──────────┬───────────┘                  (validators/)     via XSerializeEngine)
                               ▼
                    XMLDocumentHandler events ──► SAX callbacks  /  DOM tree  /  PSVI
```

---

## 9. Refactoring implications specific to input handling

1. **`BinInputStream::readBytes` and `XMLReader` are the two universal choke
   points.** If you want metrics, sandboxing, byte-limits, or a clean seam
   for testing malformed input, add it there — every channel funnels through
   them.
2. **Encoding negotiation (§1.3) is an extractable, high-value, untrusted
   unit.** Good first refactor, but gate it behind fuzz/malformed-encoding
   tests.
3. **Entity/sub-resource resolution (§2) is the security-critical seam.**
   Consolidating the four scanners (see `ARCHITECTURE.md §7`) must keep the
   `SecurityManager` counting, DTD-disable, and resolver-veto behavior
   identical across all paths — today they are re-implemented per scanner and
   have drifted.
4. **A first-class "offline / no external access" mode** would be a genuine
   improvement: today it requires either a custom `XMLEntityResolver`, DTD
   disabling, and/or a net-accessor-less build. The data-flow shows there is
   no single switch, which is a usability and security gap.
5. **Grammar deserialization (§7) is a parser in its own right** and should
   be threat-modeled and fuzzed as one; it is easy to forget because it
   doesn't look like "XML input."
6. **Buffer-lifetime contracts** (handler callbacks borrowing scanner
   buffers, `MemBufInputSource` adopt-vs-borrow) are implicit and undocumented
   at the type level; formalizing them (spans/owned types) is a safe
   modernization that also removes a class of use-after-free risk.

---
*Generated July 2026 against the 4.0.0 development tree on this fork. Line
references (`file:line`) were verified against the current source at
generation time; treat them as starting points, not guarantees after future
edits.*
