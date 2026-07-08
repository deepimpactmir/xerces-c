# Xerces-C++ Architecture Guide

This document describes the architecture, design, and implementation of this
codebase (Xerces-C++ 4.0.0, C++17) for engineers preparing to refactor it,
remove technical debt, and optimize it. It covers the layering of the library,
the data flow of a parse, the major subsystems, the pervasive idioms you must
understand before touching anything, and a catalog of known technical debt
with concrete pointers into the source.

---

## 1. What this library is

Xerces-C++ is a validating XML parser library. It provides:

- **SAX 1 and SAX 2** event-based (push) parsing APIs.
- **DOM Level 3** tree-based parsing, manipulation, and serialization
  (including the DOM Load/Save — "LS" — API).
- **Progressive (pull-ish) parsing** via `parseFirst()`/`parseNext()` tokens.
- **Validation** against DTDs and W3C XML Schema 1.0, including the full
  schema datatype library, identity constraints, and PSVI (Post-Schema
  Validation Infoset).
- **Supporting infrastructure**: Unicode transcoding, URL/network access,
  XInclude, XPath (the small subset needed for schema identity constraints),
  a regular-expression engine (for schema `pattern` facets), Base64/HexBin,
  and binary grammar serialization for grammar caching.

Everything lives in the `xercesc` namespace (see §5.1 for the macro behind
it) and is built as a single shared/static library from `src/xercesc/`.

## 2. Source tree map

```
src/xercesc/
├── util/          Foundation layer: platform abstraction, strings, containers,
│   │              memory management, transcoding, exceptions, regex, mutexes
│   ├── FileManagers/     Posix / Windows file I/O backends
│   ├── MutexManagers/    Std / Posix / Windows / NoThread mutex backends
│   ├── Transcoders/      ICU / Iconv / IconvGNU / MacOS / Win32 encoding backends
│   ├── NetAccessors/     Curl / Socket / WinSock / MacOS HTTP backends
│   ├── MsgLoaders/       ICU / InMemory / MsgCatalog diagnostic-text backends
│   └── regx/             Regular-expression engine (schema pattern facets)
├── framework/     Public parser-framework types: XMLDocumentHandler,
│   │              XMLAttr, XMLBuffer, InputSources, FormatTargets,
│   │              XMLValidator, XMLGrammarPool, MemoryManager
│   └── psvi/      PSVI + XSModel (schema component model) public API
├── internal/      The scanning engine: XMLScanner hierarchy, XMLReader,
│                  ReaderMgr, ElemStack, grammar (de)serialization
├── parsers/       User-facing parser classes gluing scanner to SAX/DOM:
│                  SAXParser, SAX2XMLReaderImpl, XercesDOMParser,
│                  AbstractDOMParser, DOMLSParserImpl
├── sax/, sax2/    SAX interface definitions (mostly pure-virtual headers)
├── dom/           DOM interface headers (pure virtual, W3C IDL-derived)
│   └── impl/      DOM implementation classes (DOMDocumentImpl etc.)
├── validators/
│   ├── common/    Grammar abstraction, content models (DFA etc.), GrammarResolver
│   ├── DTD/       DTD grammar, scanner, validator
│   ├── schema/    Schema grammar, TraverseSchema, SchemaValidator
│   │   └── identity/  xs:key / keyref / unique via XPath matching
│   └── datatype/  ~30 DatatypeValidator classes for XSD simple types
├── xinclude/      XInclude processing (DOM post-processing pass)
└── NLS/           Message catalogs (English source; XMLErrList.dtd)
```

Other top-level directories:

- `samples/src/` — 16 sample programs (DOMCount, SAX2Print, PParse, …); the
  fastest way to see each API in use.
- `tests/src/` — test harnesses (DOM tests, ThreadTest, XSTSHarness for the
  W3C schema test suite, XSerializerTest, …). These are *programs*, not a
  unit-test suite; most require external test-data files and manual
  invocation. There is no modern automated unit-test framework.
- `CMakeLists.txt` + `cmake/` — primary build system (CMake ≥ 3.12,
  `CMAKE_CXX_STANDARD 17`).
- `configure.ac`, `Makefile.am`, `m4/` — parallel autotools build.
- `doc/` — Doxygen/StyleBook documentation sources.
- `tools/` — code generators for message catalogs and character tables.

## 3. Layered architecture

The dependency direction is strictly upward; `util` depends on nothing else,
and nothing in `util` may reach into higher layers.

```
┌────────────────────────────────────────────────────────────┐
│ Application                                                │
├────────────────────────────────────────────────────────────┤
│ parsers/         SAXParser, SAX2XMLReaderImpl,             │
│                  XercesDOMParser, DOMLSParserImpl          │
├──────────────────────────────┬─────────────────────────────┤
│ sax/, sax2/ (interfaces)     │ dom/ (interfaces) dom/impl/ │
├──────────────────────────────┴─────────────────────────────┤
│ internal/        XMLScanner + 4 concrete scanners,         │
│                  XMLReader, ReaderMgr, ElemStack           │
├────────────────────────────────────────────────────────────┤
│ validators/      Grammar, GrammarResolver, content models, │
│                  DTDValidator, SchemaValidator, datatypes  │
├────────────────────────────────────────────────────────────┤
│ framework/       XMLDocumentHandler, XMLValidator,         │
│                  XMLAttr/XMLBuffer, InputSource, PSVI      │
├────────────────────────────────────────────────────────────┤
│ util/            PlatformUtils, transcoding, containers,   │
│                  strings, memory, mutexes, net, regex      │
└────────────────────────────────────────────────────────────┘
```

### 3.1 The core data flow of a parse

1. The application constructs a parser (e.g. `SAX2XMLReaderImpl` or
   `XercesDOMParser`) and calls `parse(...)` with a system id or an
   `InputSource` (`framework/`), which supplies a `BinInputStream` of raw
   bytes.
2. The parser owns an **`XMLScanner`** (`internal/`). Scanner selection is
   dynamic: `XMLScannerResolver` picks one of four concrete scanners based
   on the validation scheme and whether namespaces/schemas are enabled
   (see §4.3).
3. The scanner asks **`ReaderMgr`** for characters. `ReaderMgr` maintains a
   *stack* of **`XMLReader`** objects — one per open entity (the document
   entity, external DTD subset, each external/internal entity being
   expanded). `XMLReader` performs encoding auto-detection (BOM sniffing,
   `<?xml encoding=...?>`), transcodes raw bytes to `XMLCh` (UTF-16) using a
   transcoder from `XMLTransService`, and classifies characters via lookup
   tables (`CharTypeTables.hpp`).
4. The scanner tokenizes markup, maintains element nesting in **`ElemStack`**
   (which also tracks namespace bindings), buffers text via the
   **`XMLBufferMgr`** buffer pool, and drives validation *incrementally* by
   calling into an **`XMLValidator`** (DTD or Schema) which checks content
   models (`validators/common/*ContentModel*`) as elements open and close.
5. Grammars are looked up/created through **`GrammarResolver`** and cached in
   an **`XMLGrammarPool`**. Schema grammars are built by **`TraverseSchema`**,
   which itself parses `.xsd` files into a DOM using an internal
   **`XSDDOMParser`** and walks that DOM to build `SchemaGrammar` objects.
6. Scanner events are emitted through the abstract **`XMLDocumentHandler`**
   / `DocTypeHandler` / `XMLEntityHandler` / `XMLErrorReporter` interfaces
   (`framework/`). The `parsers/` classes implement these interfaces and
   translate the events into either SAX callbacks or DOM node construction.
   This is the key internal seam: **the scanner knows nothing about SAX or
   DOM**; both are adapters over the same event stream.

### 3.2 The two halves of DOM

- `dom/*.hpp` are abstract interfaces matching the W3C DOM spec (pure
  virtual, returned to users as `DOMElement*` etc.).
- `dom/impl/*` are the concrete classes (`DOMElementImpl`,
  `DOMDocumentImpl`, …). A node's storage is composed from small embedded
  structs (`DOMNodeImpl`, `DOMParentNode`, `DOMChildNode`) rather than a
  deep inheritance chain; implementation code recovers them via
  `dom/impl/DOMCasts.hpp` helpers (now implemented with `dynamic_cast`
  through `HasDOMNodeImpl`-style capability interfaces — this used to be
  offset-arithmetic casting and has already been made safe).
- **DOM memory model**: all nodes and strings of a document are
  *sub-allocated from per-document memory blocks* owned by
  `DOMDocumentImpl` (`allocate()` in `DOMDocumentImpl.hpp`; blocks are only
  freed when the whole document is released). Individual node destructors
  are never run for heap reclamation. `operator new(size_t, DOMDocument*)`
  is overloaded to allocate nodes inside their document. Consequences:
  deleting/replacing many nodes in a long-lived document *grows* memory
  monotonically; `DOMDocument::release()` frees everything at once. Any
  refactor of DOM internals must preserve or consciously replace this arena
  scheme — it is a deliberate performance/design decision, not an accident.

### 3.3 Parser façade classes (`parsers/`)

- `SAXParser` — SAX1 (deprecated in spirit; kept for compatibility).
- `SAX2XMLReaderImpl` — SAX2 `XMLReader`; `SAX2XMLFilterImpl` chains readers.
- `AbstractDOMParser` — implements `XMLDocumentHandler` by building a DOM
  tree; shared base of:
  - `XercesDOMParser` — classic Xerces DOM parser API.
  - `DOMLSParserImpl` — W3C DOM LS `DOMLSParser` (with filters, async modes,
    `DOMConfiguration`).

All façades expose largely overlapping feature/property sets (validation
scheme, namespaces, schema checking, entity limits, DOCTYPE disallowing,
etc.) with per-class getter/setter boilerplate — a notable duplication site.

## 4. Major subsystems in detail

### 4.1 Platform abstraction & initialization (`util/`)

- **`XMLPlatformUtils`** (`util/PlatformUtils.hpp`) is the global service
  locator. `XMLPlatformUtils::Initialize()` must be called before any other
  API and `Terminate()` after; calls are reference-counted. It owns
  process-wide singletons as *public static members*:
  `fgTransService` (transcoding), `fgNetAccessor` (HTTP),
  `fgMemoryManager` (global allocator), `fgUserPanicHandler`,
  mutex/file managers, and `fgAtomicMutex`.
- Backend selection (which transcoder, net accessor, file manager, mutex
  manager, message loader) happens at **build time** via CMake modules
  (`cmake/Xerces*Selection.cmake`) / autotools macros, compiled in with
  `#if` blocks inside `PlatformUtils.cpp`.
- **Threading model**: the library is thread-safe in the "one parser
  instance per thread" sense. There is no fine-grained concurrency;
  `XMLMutex`/`XMLMutexLock` guard the few shared structures (grammar pools,
  static registries like `RangeTokenMap`, DOM string pools during lazy
  init). `MutexManagers/StdMutexMgr` wraps `std::mutex`; the older
  Posix/Windows managers remain as alternates.

### 4.2 Strings, characters, and transcoding

- **`XMLCh`** is the pervasive character type — a typedef chosen at
  configure time (`XERCES_XMLCH_T`, usually `char16_t`; see
  `util/XercesDefs.hpp:71` and `cmake/XercesXMLCh.cmake`). All internal
  text is UTF-16.
- **`XMLString`** (`util/XMLString.hpp`, ~1600 lines, ~96 static methods) is
  a C-style function bucket over raw `XMLCh*` buffers: `copyString`,
  `transcode`, `stringLen`, `catString`, tokenizing, number parsing.
  There is **no owning string class** in general use; ownership is by
  convention plus `ArrayJanitor<XMLCh>` guards, or via `XMLBuffer` (a
  growable scratch buffer, pooled by `XMLBufferMgr`) inside the scanner.
- **`XMLTransService` / `XMLTranscoder`** abstract encoding conversion; the
  five backends live in `util/Transcoders/`. Intrinsic decoders for
  UTF-8/UTF-16/ASCII/Latin1/EBCDIC variants live beside them (`XMLUTF8Transcoder`
  etc. in `util/`). `TransService.cpp` also hosts normalization glue.
- **String pooling**: `XMLStringPool` / `StringPool` intern strings to ids;
  the scanner and grammars key most lookups on pooled ids (`unsigned int`)
  rather than string compares. `QName` (`util/QName.hpp`) carries
  prefix/localpart/URI-id triples.

### 4.3 The scanner family (`internal/`)

`XMLScanner` (abstract, `internal/XMLScanner.hpp/.cpp`, ~2400 lines of
shared logic) has four concrete subclasses selected at parse time by
`XMLScannerResolver`:

| Scanner | File(s) | Purpose |
|---|---|---|
| `WFXMLScanner` | `WFXMLScanner.cpp` (2k lines) | Well-formedness only, no grammar |
| `DGXMLScanner` | `DGXMLScanner.cpp` (3.6k) | DTD validation only |
| `SGXMLScanner` | `SGXMLScanner.cpp` (5k) | Schema validation only |
| `IGXMLScanner` | `IGXMLScanner.cpp` + `IGXMLScanner2.cpp` (6.8k combined) | Integrated DTD + Schema (the default) |

These four contain **substantial copy-paste-diverged duplication** of the
core scanning loops (`scanStartTag`, `scanContent`, attribute processing,
entity handling). Historically the split was a performance optimization to
strip validation branches from the hot path. This is the single largest
refactoring surface in the codebase — but be aware the files have diverged
in subtle, behavior-affecting ways (bug fixes applied to one and not
another), so unification requires careful differential review.

Also in `internal/`:

- `ReaderMgr` + `XMLReader` — the entity/reader stack described in §3.1.
  `XMLReader` owns the character-classification tables and the
  encoding-detection state machine; `EndOfEntityException` implements
  non-local exit when an entity ends mid-construct.
- `ElemStack` — element nesting + namespace-scope stack.
- `VecAttrListImpl` / `VecAttributesImpl` — SAX1/SAX2 attribute list
  adapters over the scanner's `XMLAttr` vector.
- `XSerializeEngine`, `XProtoType`, `XTemplateSerializer` — hand-rolled
  binary object serialization used to cache compiled grammars
  (`GRAMMAR_SERIALIZATION_LEVEL=7` in `configure.ac`; every serializable
  class implements `IS_SERIALIZABLE` macros). Touching data members of any
  grammar/validator class can break serialized-grammar compatibility.
- `MemoryManagerImpl` — default `MemoryManager` (thin `new[]` wrapper).

**Security posture** (recent work visible in git history): the scanner layer
enforces `SecurityManager` limits (entity expansion count, `ENTITY_EXPANSION_LIMIT`
default in `util/SecurityManager.hpp`), supports programmatic DOCTYPE/DTD
disallowing (recent commits added this to SAX parsers), and contains the
CVE-2018-1311 use-after-free fix in external-DTD scanning. Preserve these
behaviors during refactoring; regressions here are CVEs.

### 4.4 Validation (`validators/`)

- **`Grammar`** (`validators/common/Grammar.hpp`) abstracts DTD vs Schema
  grammars: pools of element decls (`XMLElementDecl`), attribute defs
  (`XMLAttDef`), entities, notations. `GrammarResolver` maps
  namespace/system ids to grammars and fronts the (optional, user-supplied)
  `XMLGrammarPool` for cross-parse caching.
- **Content models** (`validators/common/`): element content is compiled
  into one of `SimpleContentModel` (trivial patterns), `MixedContentModel`,
  `AllContentModel` (xs:all), or `DFAContentModel` — a classic
  NFA→DFA construction from the `ContentSpecNode` tree, with `CMStateSet`
  bitsets. DFA construction cost/size is a known hotspot for pathological
  schemas.
- **DTD side** (`validators/DTD/`): `DTDScanner` parses internal/external
  subsets directly (character-level scanning, separate from XMLScanner),
  building `DTDGrammar`; `DTDValidator` enforces it.
- **Schema side** (`validators/schema/`):
  - `TraverseSchema` (**9,466 lines**, the largest file in the codebase)
    walks the XSD DOM and builds `SchemaGrammar`, `ComplexTypeInfo`,
    `SchemaElementDecl`, attribute groups, wildcards, substitution groups.
    It is a god-class handling parsing, resolution, redefinition, import /
    include semantics, and error reporting in one pass structure.
  - `SchemaValidator` performs per-element validation during scanning;
    identity constraints (`identity/`) evaluate a restricted XPath subset
    (`XercesXPath`, `XPathMatcher`) via `FieldActivator`/`ValueStore`.
  - `GeneralAttributeCheck` validates attributes *of schema documents
    themselves* against built-in metadata.
- **Datatypes** (`validators/datatype/`): ~30 `DatatypeValidator`
  subclasses (one per XSD built-in), created through
  `DatatypeValidatorFactory`, layered by inheritance
  (`AbstractStringValidator`, `AbstractNumericValidator`, …). Facet
  checking (pattern → `util/regx`, enumeration, min/max, etc.) happens in
  `validate()`. Numeric types use arbitrary-precision helpers
  (`util/XMLBigDecimal`, `XMLBigInteger`, `XMLAbstractDoubleFloat`,
  `XMLDateTime`).
- **PSVI / XSModel** (`framework/psvi/`): a read-only reflection of the
  compiled schema components (`XSModel`, `XSElementDeclaration`,
  `XSComplexTypeDefinition`, …) plus per-node PSVI results
  (`PSVIElement`, `PSVIAttribute`) delivered through `PSVIHandler`. These
  are built lazily from grammar internals and hold many cross-pointers into
  them — changing grammar data structures usually means updating both the
  XSModel builders and the `XTemplateSerializer`.

### 4.5 Regular expressions (`util/regx/`)

A self-contained port of Jakarta/Xerces-J's regex engine: `RegxParser` /
`ParserForXMLSchema` build a `Token` tree, `RegularExpression` compiles it
to an `Op` graph and interprets it (backtracking matcher), with `BMPattern`
(Boyer-Moore) fast paths and `RangeToken`/`RangeTokenMap` for Unicode
category ranges. Used by schema `pattern` facets and public API. It
predates `std::regex` (and intentionally implements *XML Schema* regex
semantics, which `std::regex` does not support — do not naively replace it).

### 4.6 XInclude (`xinclude/`)

`XIncludeUtils` + `XIncludeDOMDocumentProcessor` implement XInclude as a
**DOM post-processing pass** (used by `XIncludeParserConfiguration`-style
setups and the `tests/src/xinclude` harness). It is not integrated into the
streaming scanner path.

### 4.7 Serialization / output

`DOMLSSerializerImpl` (`dom/impl/`) writes DOM trees through an
`XMLFormatter` (`framework/XMLFormatter.hpp`) which handles output
transcoding and escaping, into `XMLFormatTarget` sinks
(`LocalFileFormatTarget`, `MemBufFormatTarget`, `StdOutFormatTarget`).

## 5. Pervasive idioms you must know before editing

### 5.1 Namespace and export macros

- `namespace XERCES_CPP_NAMESPACE { ... }` wraps every file; the macro
  expands to a versioned name (e.g. `xercesc_4_0`) with
  `namespace xercesc = XERCES_CPP_NAMESPACE;` as the user-facing alias
  (`util/XercesDefs.hpp:144`). Older code used
  `XERCES_CPP_NAMESPACE_BEGIN/END` macros; both appear in the tree.
- `XMLUTIL_EXPORT`, `XMLPARSER_EXPORT`, `SAX_EXPORT`, `CDOM_EXPORT` etc. are
  DLL export/import macros (`util/XercesDefs.hpp`, per-platform config
  headers). Every public class needs the right one.

### 5.2 Memory management: `MemoryManager` everywhere

- Nearly every allocating class takes a `MemoryManager* manager =
  XMLPlatformUtils::fgMemoryManager` constructor parameter, stores it, and
  routes all allocation through it. Most heap classes derive from
  **`XMemory`** (`util/XMemory.hpp`), whose overloaded `operator new`
  routes through a `MemoryManager`.
- **Janitors** (`util/Janitor.hpp`, `ArrayJanitor`, `JanitorMemFunCall`,
  `FlagJanitor`) are the in-house RAII guards used for exception safety —
  the codebase predates `std::unique_ptr` and does not use it. Raw
  owning pointers + explicit `delete` in destructors are the norm.
- Any refactor introducing standard containers/smart pointers must decide
  how to honor the pluggable-`MemoryManager` contract (it is public API —
  users pass custom managers e.g. for pool allocation and OOM handling).

### 5.3 The `.c` template-include idiom

Generic containers are split as `Foo.hpp` (declaration) + `Foo.c`
(definitions), with the `.hpp` doing `#include <xercesc/util/Foo.c>`
(see `util/BaseRefVectorOf.hpp:155`). The `.c` files are C++ template
implementation files, not C. There are ~21 of them in `util/` plus
`XSNamedMap.c`, `DOMDeepNodeListPool.c`. This confuses tooling and build
glob rules; renaming to `.icc`/`.tcc` or merging into headers is a safe,
mechanical cleanup.

### 5.4 The home-grown container zoo

`util/` contains a full pre-STL container library, all used heavily:

- Vectors: `ValueVectorOf`, `RefVectorOf`, `RefArrayVectorOf`, `BaseRefVectorOf`
- Hash maps: `RefHashTableOf`, `ValueHashTableOf`, `RefHash2KeysTableOf`,
  `RefHash3KeysIdPool`, `Hash2KeysSetOf` (+ `Hashers.hpp`)
- Stacks: `ValueStackOf`, `RefStackOf`; arrays: `ValueArrayOf`, `RefArrayOf`
- Pools: `NameIdPool`, `XMLStringPool`; misc: `BitSet`, `KVStringPair`,
  `CountedPointer`

"Ref" containers optionally *own and delete* their elements (`adoptElems`
flags); enumerator classes (`*Enumerator`) provide iteration. These types
appear in public headers and in serialized grammar formats
(`XTemplateSerializer` knows how to serialize each), so replacing them with
STL is a large, API- and format-breaking project — plan it as such.

### 5.5 Error reporting and message loading

Errors are emitted by numeric code through `XMLErrorReporter` /
`emitError(...)` with per-domain code enums generated into
`framework/XMLErrorCodes.hpp`, `XMLValidityCodes.hpp`, etc. Message *text*
is resolved at runtime by the configured `MsgLoader`
(`util/MsgLoaders/InMemory` by default — generated arrays in
`XercesMessages_en_US.hpp`), sourced from `NLS/` catalogs via `tools/`
generators. Adding an error message means touching the catalog + regenerated
headers, not just a string literal.

### 5.6 Exceptions

`XMLException` (util) is the base of a wide family (`RuntimeException`,
`TranscodingException`, …) created via `THROW_FROM_CODE`-style macros
carrying message codes; DOM has its own `DOMException`/`DOMLSException`;
SAX has `SAXException`. `OutOfMemoryException` is thrown (not
`std::bad_alloc`) and several call sites catch it specially. Exception
classes carry a `MemoryManager` for message allocation.

## 6. Build systems and configuration

- **CMake is primary** (`CMakeLists.txt` + 20 modules under `cmake/`):
  selects transcoder/netaccessor/filemgr/mutexmgr/msgloader backends,
  detects `XMLCh` type, SSE2, large-file support, generates
  `config.h` and `Xerces_autoconf_config.hpp`. Tests/samples build via
  CMake with `RunTest.cmake`.
- **Autotools is maintained in parallel** (`configure.ac`, `Makefile.am`
  per directory, `m4/`). Every file addition/removal must be reflected in
  *both* `CMakeLists.txt` (or its source lists) and `Makefile.am`, or one
  build silently breaks. This duplication is itself standing tech debt;
  check with stakeholders whether autotools can be dropped before investing
  in it.
- Library versioning: package 4.0.0, `INTERFACE_VERSION=4.0`; grammar
  serialization format is versioned independently
  (`GRAMMAR_SERIALIZATION_LEVEL=7`).

## 7. Technical-debt catalog (refactoring targets, prioritized)

Ordered roughly by value-to-risk ratio. In all cases the binding
constraints are: (a) the public C++ API/ABI is consumed by many downstream
projects, (b) the binary grammar-serialization format must stay readable or
be explicitly version-bumped, and (c) security fixes in the scanner/entity
path must not regress.

1. **No automated unit-test safety net.** `tests/src` are manual harnesses.
   Before any structural refactor, stand up a CI-runnable test suite
   (wire existing harnesses + the W3C XML/XSTS conformance suites into
   CTest; add targeted unit tests around code you're about to change).
   This is prerequisite work, not optional.
2. **Scanner quadruplication** (`internal/{WF,DG,SG,IG}XMLScanner*.cpp`,
   ~14k lines with heavy copy-paste divergence). Highest-value dedup
   target: extract shared attribute/entity/content scanning into the
   `XMLScanner` base or policy templates. Requires differential analysis
   first — the copies have drifted deliberately and accidentally.
3. **`TraverseSchema` god-class** (9.5k lines): split by concern
   (per-component traversers, resolution phase, error reporting). High
   value for maintainability; moderate risk (dense internal state sharing
   via `SchemaInfo`).
4. **Manual memory management**: raw owning pointers + Janitors throughout.
   Incremental, low-risk modernization: introduce internal smart-pointer
   aliases that respect `MemoryManager`, convert leaf classes first.
   `CountedPointer` and `Janitor` can be thinned to standard equivalents
   internally without changing public API.
5. **Duplicated parser-façade feature plumbing** (`parsers/*`): four
   classes re-implement the same ~40 feature/property switches around one
   `XMLScanner`. Extract a shared feature-state object.
6. **Container zoo + `.c` template files** (§5.3–5.4): mechanical rename of
   `.c` → `.tpp`-style is trivial and safe; wholesale STL replacement is a
   major-version project because containers leak into public headers and
   the serialization format.
7. **`XMLString` static bucket + raw `XMLCh*` ownership-by-convention**:
   introduce an internal owning string type (or `std::u16string` when
   `XMLCh == char16_t`) at module boundaries incrementally.
8. **Dual build systems** (§6): decide, then delete one.
9. **Global mutable singletons** in `XMLPlatformUtils` (public static
   pointers, build-time backend selection): hard to test, order-sensitive
   init. Longer-term: encapsulate behind accessors, allow runtime
   injection (partially exists via `Initialize()` params for panic
   handler/memory manager).
10. **Dead platforms/backends**: OS/390 `#ifdef`s, MacOS pre-OSX
    Carbon transcoder/net accessor (`MacOSUnicodeConverter`,
    `MacOSURLAccessCF`), `NoThreadMutexMgr`, SAX1. Auditing and pruning
    unmaintained backends shrinks the `#if` matrix meaningfully.
11. **Performance opportunities** (measure first — `tests/src/ParserTest`,
    DOMCount/SAXCount with large corpora): DFA content-model build for
    large schemas; transcoder call overhead per-buffer; `XMLBuffer`
    growth policy; hash-table sizing (recent commit added primes to
    `DOMNodeIDMap` — more of the same likely applies); string-pool
    contention under the grammar-pool mutex in multi-threaded reuse.

## 8. Where to start reading code

| Goal | Entry point |
|---|---|
| Follow a SAX2 parse end-to-end | `samples/src/SAX2Count` → `parsers/SAX2XMLReaderImpl.cpp::parse` → `internal/IGXMLScanner.cpp::scanDocument` |
| Understand entity/reader handling | `internal/ReaderMgr.cpp`, `internal/XMLReader.cpp` |
| Understand DOM building | `parsers/AbstractDOMParser.cpp` (esp. `startElement`, `docCharacters`) |
| Understand DOM memory | `dom/impl/DOMDocumentImpl.{hpp,cpp}` (`allocate`, comment block ~line 316) |
| Understand schema compilation | `validators/schema/TraverseSchema.cpp::doTraverseSchema` |
| Understand validation-during-scan | `validators/schema/SchemaValidator.cpp`, `validators/common/DFAContentModel.cpp` |
| Understand grammar caching | `framework/XMLGrammarPoolImpl.cpp`, `internal/XSerializeEngine.cpp` |
| See every public knob | `parsers/SAX2XMLReaderImpl.cpp::setFeature/setProperty` |

---
*Generated July 2026 against the 4.0.0 development tree on this fork
(includes post-3.x hardening: CVE-2018-1311 fix, DOCTYPE-disallow features,
`dynamic_cast`-based DOMCasts).*
