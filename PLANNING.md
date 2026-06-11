# Planning Notebook: draft-westerlund-moq-overview

## Document Purpose

An **informational** IETF draft providing a high-level overview of the
Media over QUIC (MoQ) protocol suite. The target audience is someone who
wants to understand how the pieces fit together *before* diving into the
normative specifications. It should:

1. Explain the problem space and motivation
2. Introduce the data model and terminology
3. Show how the protocol components relate
4. Describe how interoperability is achieved for specific use cases
5. Cover security at a conceptual level
6. Point readers to the right normative spec for each topic

It is NOT a tutorial, NOT a requirements doc, and does NOT define new
protocol behaviour.

---

## Housekeeping Issues in Current Draft

| Issue | Fix |
|-------|-----|
| `cat: std` in YAML front matter | Change to `cat: info` |
| References `I-D.ietf-moq-warp` | Replace with `I-D.ietf-moq-msf` (draft-ietf-moq-msf-01) |
| References `I-D.jennings-moq-secure-objects` | Replace with `I-D.ietf-moq-secure-objects` (now a WG item) |
| Missing references | Add `I-D.ietf-moq-cmsf`, `I-D.englishm-moq-relay-dos` |
| Remove `I-D.cenzano-moq-media-interop` | LOC/codec details now live in moq-msf and moq-loc |

---

## Resolved Document Structure

```
1. Introduction
   1.1. Motivation
   1.2. Document Scope and Non-Goals
   1.3. Terminology

2. Use Cases
   2.1. Live Streaming Media Delivery
   2.2. Real-Time Conferencing
   2.3. Content Contribution and Primary Distribution (Ingest)

3. Architecture and Nodes
   3.1. Roles: Publisher, Subscriber, Original Publisher, End Subscriber
   3.2. Relays
        - Content-agnostic but not application-agnostic
        - Fan-out, caching, topology
        - Trust and operational considerations (brief, point to relay-dos)
   3.3. Deployment Topologies (point-to-point, single relay, relay tree)

4. Data Model
   4.1. Tracks, Groups, Subgroups, Objects
   4.2. Track Naming (Namespace + Track Name)
   4.3. Properties (Track and Object)
   4.4. Object Immutability and Cacheability

5. MOQT: The Transport Protocol
   5.1. Session Establishment (QUIC, WebTransport, SETUP)
   5.2. Publishing and Subscribing
        (SUBSCRIBE, PUBLISH, FETCH, namespace discovery)
   5.3. Priorities and Delivery
        (scheduling, timeouts, partial reliability)
   5.4. Extensibility and Session Management
        (options, parameters, properties, GOAWAY)

6. Streaming Formats
   6.1. Role of a Streaming Format
   6.2. MSF: MOQT Streaming Format (LOC-based)
        - Catalog, Media Timeline, ABR, Event Timelines
   6.3. CMSF: CMAF-compliant Streaming Format
   6.4. Relationship between MSF, LOC, and Applications

7. Security Architecture
   7.1. Hop-by-Hop Security (TLS/QUIC)
   7.2. End-to-End Object Encryption (MoQ Secure Objects / SFrame)
   7.3. Authorization Model (tokens, relay verification)
   7.4. Privacy Considerations

8. Achieving Interoperability
   8.1. What Defines an Interoperability Point
   8.2. MSF as the WG Interoperability Point
   8.3. Extending for New Use Cases
   8.4. Reading Guide (table mapping goals → specs)

9. Security Considerations
   (residual risks, threat summary, pointers to §7)

10. IANA Considerations (None)
```

### Design Decisions (resolved)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Data Model vs Architecture ordering | Architecture first (§3→§4) | Establish actors before what they exchange |
| Transport section depth | Slim (4 subsections) | Overview, not condensed restatement |
| Relay Considerations | Folded into Architecture §3.2 | Key insight (content-agnostic, not app-agnostic) fits there; operational detail out of scope |
| Security treatment | Option C: §7 architecture + §9 residual risks | Clean separation of mechanisms vs threats |
| Use Cases placement | §2 (before technical detail) | Motivational framing |
| Capstone section | "Achieving Interoperability" with reading-guide table | Shows how pieces compose; practical for readers |

---

## Section-by-Section Content Guidance

### 1. Introduction

**Key messages:**
- MoQ is a publish/subscribe media delivery protocol suite built on QUIC
- Designed to replace fragmented ingest+distribution stacks (RTMP→HLS/DASH)
  with a single protocol from contribution to consumption
- Relays are first-class, enabling CDN-scale distribution
- Content-agnostic transport; media formats defined separately

**Sources:** moq-transport-18 §1 (Introduction, Motivation)

### 1.1 Motivation

Summarise the four pillars from moq-transport §1.1:
- Latency (avoid HoL blocking, rapid congestion response)
- Leveraging QUIC (parallel streams, partial reliability)
- Convergence (single protocol ingest-to-distribution)
- Relays (scale via intermediaries without accessing media payload)

### 1.2 Document Scope and Non-Goals

State explicitly:
- This doc does NOT define protocol behaviour
- It provides orientation for implementers and reviewers
- It maps each topic to the relevant normative draft

### 1.3 Terminology

Pull the key terms from moq-transport §1.2:
- Client, Server, Endpoint, Peer
- Publisher, Subscriber, Original Publisher, End Subscriber
- Relay, Upstream, Downstream
- Transport Session, Track, Group, Object, Subgroup

---

### 2. Use Cases

For each use case, describe:
- The topology (who publishes, who subscribes, how many relays)
- The latency regime (interactive <500ms, live <5s, VOD)
- The key protocol features exercised

**2.1 Live Streaming:**
- One original publisher → relay tree → many end subscribers
- Target latency 1-5 seconds
- ABR switching, catalog-driven track selection
- Source: moq-msf §1 (scope), moq-transport §1.1.4

**2.2 Real-Time Conferencing:**
- Many publishers, many subscribers, possibly mesh of relays
- Target latency <500ms
- Each participant publishes audio+video tracks
- Subscribers select which participants to receive
- May use SVC/simulcast via dependencies in catalog

**2.3 Ingest/Contribution:**
- Professional encoder → CDN ingress relay
- High quality, possibly redundant paths
- The CDN relay becomes the "original publisher" for downstream distribution
  after transcoding

---

### 3. Architecture and Nodes

**3.1 Roles:**
Brief definition of each (1-2 sentences). Note that a single endpoint
can be both publisher and subscriber (e.g., in conferencing).

**3.2 Relays:**
- Terminate transport sessions (not transparent proxies)
- Can cache objects (identified by Full Track Name + Group + Object)
- Fan-out: single upstream subscription serves many downstream subscribers
- Do NOT access media payload (opaque); only see metadata/properties
- Source: moq-transport §9

**3.3 Deployment Topologies:**

Describe with ASCII-art:
```
Point-to-point:    Publisher ←→ Subscriber

Single relay:      Publisher ←→ Relay ←→ Subscriber(s)

Relay tree:        Publisher ←→ Origin Relay ←→ Edge Relay(s) ←→ Subscriber(s)
```

Mention that relay chains can be longer, and that GOAWAY enables migration.

---

### 4. Data Model

This is the conceptual heart of the overview. Source: moq-transport §2.

**4.1 Hierarchy:**
```
Track
  └── Group (join point; independently decodable)
        └── Subgroup (stream-mapped; dependency/priority unit)
              └── Object (smallest addressable payload)
```

Key points:
- Objects are immutable once published
- Groups are random-access points (subscriber can join at group boundary)
- Subgroups map to QUIC streams; objects in a subgroup share priority

**4.2 Track Naming:**
- Full Track Name = Track Namespace (ordered tuple of fields) + Track Name
- Namespace enables prefix-based routing at relays
- Example: (conference.example.com, meeting123, alice) + "video-hd"

**4.3 Properties:**
- Track Properties: relay-visible metadata (priority, timeouts, cache duration)
- Object Properties: per-object metadata (gap indicators, timestamps via LOC)
- Immutable Properties: authenticated E2E, cannot be altered by relays

**4.4 Object Immutability:**
- Same (namespace, name, group, object) → same bytes forever
- Enables caching and deduplication at relays
- Status signals: END_OF_GROUP, END_OF_TRACK

---

### 5. MOQT: The Transport Protocol

**5.1 Session Establishment:**
- `moqt://` URI scheme
- Native QUIC (ALPN `moqt`) or WebTransport over HTTP/3
- SETUP message exchange on control streams
- Extension negotiation via Setup Options

**5.2 Publish/Subscribe:**
- SUBSCRIBE: "send me new objects for this track as they appear"
  - Filters: Largest Object, Next Group, AbsoluteStart, AbsoluteRange
- PUBLISH: publisher-initiated subscription (push model)
- FETCH: "send me this specific range of past objects"
- Joining Fetch: contiguous with a subscription (fill playback buffer)
- Subscription state machine: Idle → Pending → Established → Terminated

**5.3 Namespace Discovery:**
- PUBLISH_NAMESPACE: "I have tracks under this namespace"
- SUBSCRIBE_NAMESPACE: "tell me about namespaces matching this prefix"
- SUBSCRIBE_TRACKS: "send me PUBLISH messages for tracks in this namespace"
- Relays match prefixes to route subscriptions to publishers

**5.4 Priority and Congestion:**
- Two-level priority: Subscriber Priority (per-request) + Publisher Priority (per-object)
- Group Order: ascending or descending (newest-first or oldest-first)
- Under congestion, lower-priority streams are starved or dropped
- Delivery timeouts (OBJECT_DELIVERY_TIMEOUT, SUBGROUP_DELIVERY_TIMEOUT)

**5.5 Partial Reliability:**
- Objects can be sent as datagrams (unreliable) or streams (reliable)
- Timeouts cause stream resets → controlled data loss
- Publisher can drop objects that expire before transmission

**5.6 Extensibility:**
- Setup Options: negotiated per-session extensions
- Message Parameters: per-message metadata (not forwarded by relays)
- Properties: per-track/object metadata (forwarded and cached by relays)
- IANA registries for all code points; GREASE for forward compatibility

**5.7 Session Migration:**
- GOAWAY with optional new URI and timeout
- Enables server restart without disrupting subscribers
- Client re-establishes subscriptions on new session

---

### 6. Streaming Formats

**6.1 Role of a Streaming Format:**
- Defines how media is packaged into MOQT objects
- Defines a catalog format for track discovery and selection
- Defines ABR switching rules, timeline mapping, event metadata
- MOQT is content-agnostic; streaming format provides the media semantics

**6.2 MSF (MOQT Streaming Format):**
- Uses LOC (Low Overhead Container) packaging: one sample per object
- JSON catalog track (named "catalog") describing all available tracks
  - codec, bitrate, resolution, framerate, language, etc.
  - renderGroup (tracks to play together), altGroup (ABR alternatives)
  - Delta updates for adding/removing tracks
- Media Timeline track: maps Group/Object IDs to media PTS and wallclock
- Event Timeline tracks: synchronized metadata (scores, GPS, SCTE-35)
- Publish tracks: subscriber can publish logs/metrics back
- Variable substitution for personalized delivery
- Authorization signalling (authInfo field)
- Source: moq-msf-01

**6.3 CMSF (CMAF-compliant MSF):**
- Extends MSF for CMAF/ISO-BMFF packaged media
- Each MOQT Object = one or more CMAF Chunks (moof+mdat)
- Initialization segments carried in catalog (base64 inline)
- Common Encryption (CENC) for DRM integration (Widevine, PlayReady, FairPlay)
- SAP-type event timeline for seeking
- Source: moq-cmsf-01

**6.4 Relationship:**
```
Application
    ↓ uses
Streaming Format (MSF or CMSF)
    ↓ defines packaging for
LOC / CMAF container
    ↓ carried by
MOQT (transport)
    ↓ over
QUIC / WebTransport
```

---

### 7. End-to-End Security

**7.1 Hop-by-Hop:**
- TLS 1.3 in QUIC provides confidentiality and integrity per hop
- Relays terminate TLS → they can see MOQT headers and object metadata
- Object payloads are opaque to relays but NOT encrypted E2E by default

**7.2 E2E Object Encryption (MoQ Secure Objects):**
- Based on SFrame (RFC 9605)
- Encrypts object payload + optional "encrypted properties"
- Authenticates: Group ID, Object ID, Track Name, Immutable Properties
- Key derivation: HKDF from (track_base_key, Key ID, Full Track Name)
- Nonce: GroupID(64bit) || ObjectID(32bit) XOR salt
- Relays cannot decrypt; can still route based on unencrypted metadata
- Source: moq-secure-objects-00

**7.3 Authorization:**
- AUTHORIZATION TOKEN parameter in control messages
- Token aliasing for efficiency (register once, reference by alias)
- Schemes: Privacy Pass, Common Access Tokens (CAT)
- Relay verifies tokens before forwarding content
- Source: moq-transport §10.2.2, moq-msf §11.4

**7.4 Privacy:**
- Track names and namespace visible to relays (traffic analysis risk)
- Object sizes reveal content characteristics
- MOQT_IMPLEMENTATION option enables fingerprinting
- Padding streams/datagrams for traffic shaping

---

### 8. Relay Considerations

**8.1 First-Class Citizen:**
- Designed into the protocol, not bolted on
- Relay implements ALL MOQT messages
- Terminates sessions, manages subscriptions independently on each hop

**8.2 Caching and Forwarding:**
- Cache key: Full Track Name + Group ID + Object ID
- MUST NOT modify object payloads
- MAY aggregate subscriptions (one upstream, many downstream)
- MAX_CACHE_DURATION property controls cache lifetime

**8.3 Multiple Publishers:**
- Relay can receive same track from multiple publishers
- MUST deduplicate objects before forwarding
- Handles publisher failover transparently

**8.4 DoS Resilience:**
- Amplification risk: one SUBSCRIBE triggers unbounded data flow
- Control plane flooding (PUBLISH_NAMESPACE, rapid subscribe/cancel)
- Slow subscriber causing buffer accumulation
- Mitigations: rate limiting, per-session resource limits, EXCESSIVE_LOAD error
- Source: moq-relay-dos-00

---

### 9. Achieving Interoperability

**9.1 What Defines an Interop Point:**
An interoperability point specifies:
1. MOQT version (ALPN)
2. Streaming format (MSF, CMSF, or custom)
3. Supported codecs and container format
4. Security requirements (E2E encryption scheme, auth scheme)
5. Any mandatory extensions

**9.2 MSF as the WG Interop Point:**
- MSF version 1 + MOQT draft-18 + LOC packaging
- Catalog format defines track discovery
- Time-alignment rules enable ABR switching
- Authorization via CAT or Privacy Pass

**9.3 Extending for New Use Cases:**
- Define new packaging type in catalog
- Register new event timeline types
- Add Properties via IANA registry
- Use Mandatory Track Properties (0x4000-0x7FFF) for required extensions

---

### 10. Security Considerations

Brief summary pointing to §7. Highlight:
- The trust boundary is at the relay (it sees metadata but not payload if E2E encrypted)
- Authorization tokens prove subscriber's right to access
- Cache poisoning risk if relays don't verify publisher identity
- Replay protection delegated to token scheme

### 11. IANA Considerations

None (informational document).

---

## Key Figures to Include

1. **Protocol stack diagram** (§1 or §5.1):
   ```
   +------------------+
   | Application      |
   +------------------+
   | Streaming Format |
   | (MSF / CMSF)    |
   +------------------+
   | MOQT             |
   +------------------+
   | QUIC / WebTrans  |
   +------------------+
   | UDP / IP         |
   +------------------+
   ```

2. **Data model hierarchy** (§4.1) — as shown above

3. **Relay topology** (§3.3) — publisher → relay tree → subscribers

4. **Subscription state machine** (§5.2) — simplified from moq-transport

5. **Security layers** (§7):
   ```
   E2E encrypted payload   ← MoQ Secure Objects (Original Publisher ↔ End Subscriber)
   MOQT object metadata    ← Visible to Relays
   TLS/QUIC               ← Hop-by-hop confidentiality
   ```

---

## References to Update

### Normative (should be empty for an informational overview)
None.

### Informative
| Current ref | Replace with |
|-------------|-------------|
| `I-D.ietf-moq-warp` | `I-D.ietf-moq-msf` |
| `I-D.jennings-moq-secure-objects` | `I-D.ietf-moq-secure-objects` |
| (add) | `I-D.ietf-moq-cmsf` |
| (add) | `I-D.englishm-moq-relay-dos` |
| (add) | `I-D.ietf-moq-loc` (already present) |
| `I-D.cenzano-moq-media-interop` | Remove (content now in MSF/LOC) |
| `RFC9605` | Keep (SFrame) |
| `RFC9000` | Keep (QUIC) |
| `I-D.ietf-webtrans-overview` | Keep (WebTransport) |
| `I-D.ietf-moq-transport` | Keep |

---

## Writing Style Guidelines

- Use present tense ("MOQT defines..." not "MOQT will define...")
- Avoid normative language (MUST/SHOULD/MAY) — this is informational
- Keep paragraphs short; use bullet lists for enumerations
- Every concept introduced should have a forward pointer to the
  relevant normative spec section
- ASCII art for diagrams (kramdown-rfc supports it)
- Target length: 25-35 pages rendered (current specs are 140+ pages;
  this should be a 20-minute read that orients someone)

---

## Open Questions for Authors

1. Should the draft cover moq-loc separately from MSF, or treat LOC as
   an implementation detail of MSF? (Recommendation: mention it as the
   packaging layer MSF uses, with a brief description, but don't
   dedicate a full section to LOC wire format.)

2. How much detail on CMSF? It's a separate WG item targeting existing
   CMAF/DASH workflows. (Recommendation: one subsection showing it
   exists and how it differs from base MSF.)

3. Should the draft discuss the LOC Object Properties (TIMESTAMP,
   VIDEO_FRAME_MARKING, etc.)? (Recommendation: mention them in §4.3
   as examples of object properties, don't enumerate all of them.)

4. How to handle the relay DoS draft (individual, not WG)? 
   (Recommendation: reference it in §8.4 as complementary analysis,
   note it's an individual submission.)

5. Should there be a section on "how to read the other specs"? A reading
   guide mapping reader goals to specs. (Recommendation: yes, as a short
   appendix or subsection of §9.)
