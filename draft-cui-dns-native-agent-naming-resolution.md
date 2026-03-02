---
title: "DNS-Native AI Agent Naming and Resolution"
abbrev: "DN-ANR"
category: info

docname: draft-cui-dns-native-agent-naming-resolution-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications"
workgroup: "Domain Name System"
keyword:
 - AI agent
 - DNS
 - SVCB
 - service resolution
 - agent protocol
venue:
  group: "Domain Name System"
  type: "Working Group"
  mail: "namedroppers@nic.ddn.mil"
  arch: "nicfs.nic.ddn.mil:~/namedroppers/*.Z"
  github: "nobrowning/dns-native-agent-naming-resolution"
  latest: "https://nobrowning.github.io/dns-native-agent-naming-resolution/draft-cui-dns-native-agent-naming-resolution.html"

author:
- role:  # remove if not true
  fullname: Yong Cui
  organization: Tsinghua University
  email: cuiyong@tsinghua.edu.cn
  region: Beijing # not always available
  code: 100084
  country: China # use TLD (except UK) or country name
  phone:
  email: cuiyong@tsinghua.edu.cn
  uri: http://www.cuiyong.net/

normative:
  RFC1035:
  RFC9460:
  RFC9461:
  RFC8446:
  RFC4033:
  RFC9525:
  RFC4648:
  RFC8785:

informative:
  A2A:
    title: "Agent2Agent Protocol (A2A)"
    target: https://google.github.io/A2A/
    author:
      - org: Google
    date: 2025
  ANP:
    title: "Agent Network Protocol (ANP)"
    target: https://agent-network-protocol.com/
    author:
      - org: ANP Community
    date: 2025


...

--- abstract

This document specifies DNS-Native Agent Naming and Resolution (DN-ANR) for AI agents. DN-ANR has three goals: (1) use domain names (FQDNs) as stable Agent Identifiers, (2) resolve Agent Identifiers to verifiable endpoints and supported protocol/version information with a cryptographic integrity chain (DNSSEC preferred), and (3) provide only minimal and stable pointer/index capabilities that can be referenced by upper-layer discovery systems. DN-ANR intentionally does not carry heavy semantic metadata in DNS, and does not define semantic discovery, ranking, or routing decisions.


--- middle

# Introduction

The emergence of AI agents as autonomous software entities creates concrete requirements for naming, trusted resolution, and endpoint verification. Existing deployments often mix discovery, semantic matching, and resolution into one control plane, which increases coupling and weakens interoperability.

This document defines DN-ANR as a DNS-native resolution layer built on {{RFC1035}} and Service Binding (SVCB/HTTPS RRs, {{RFC9460}}, {{RFC9461}}). The design objective is strict scope control: discovery systems produce candidate Agent Identifiers, while DN-ANR securely resolves a chosen Agent Identifier into connection material.

## Goals

DN-ANR goals are:

1. **Identity naming**: use domain names/FQDNs as administratively managed Agent Identifiers.
2. **Trusted resolution and connection guidance**: resolve an Agent Identifier to endpoint(s), protocol/version declarations, and verifiable integrity material.
3. **Foundational support for discovery systems**: expose only minimal stable pointers/indexes that upper-layer discovery systems MAY reference.

## Non-Goals

DN-ANR non-goals are:

- DN-ANR does not provide cross-domain agent discovery by semantics or capability.
- DN-ANR does not provide semantic matching, capability ranking, or task-routing decisions.
- DN-ANR does not standardize heavy capability metadata schemas inside DNS.
- DN-ANR only specifies how to securely and deterministically connect after an Agent Identifier has been selected.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

Agent:
: An autonomous software entity capable of communicating with other agents or humans using defined protocols.

Agent Identifier:
: A Fully Qualified Domain Name (FQDN) that uniquely identifies an agent.

Agent Protocol:
: The application-layer protocol used for agent-to-agent communication (e.g., {{A2A}}, {{ANP}}).


# Design Principles

This specification follows five core principles:

| Principle | Description |
|-----------|-------------|
| DNS-First | DNS is the authoritative source for Agent Identifier resolution; HTTP serves only as a fallback mirror |
| Layered Scope | Discovery and semantic selection are out of scope; DN-ANR resolves selected identifiers |
| Path-Independent | Version and endpoint selection are controlled by DNS, not URL paths |
| Protocol Autonomy | Agent interaction protocols are decoupled from transport |
| Default Availability | A/AAAA records guarantee minimum connectivity; enhanced features are optional. Default version is provided when not specified |

# Architecture Overview

DN-ANR is a resolution-layer specification inside a three-layer architecture:

| Layer | Examples | Scope |
|-------|----------|-------|
| Discovery Layer | Web registry, agent gateway, search engine, semantic router, DNS-SD/mDNS | OUT OF SCOPE |
| Resolution Layer | Agent Identifier (FQDN) -> endpoint, protocol/version, integrity material | DN-ANR scope |
| Connection Layer | A2A, MCP, HTTPS, gRPC, other application protocols | OUT OF SCOPE |

Interface boundary:

- Discovery Layer outputs one or more candidate Agent Identifiers (FQDNs).
- DN-ANR takes one selected Agent Identifier and resolves it into verifiable connection guidance.
- Connection Layer consumes DN-ANR outputs and executes protocol-specific session logic.


# Naming and Resource Location

## Domain Name as Identity

Each agent is uniquely identified by a stable Fully Qualified Domain Name (FQDN). Domain ownership combined with TLS certificates forms the foundation of agent identity.

### Naming Rules

- Use domain names or subdomains owned by the organization
- Agent version changes do not introduce new identities
- No registration with any central authority is required

### Naming Examples

~~~
# Recommended: dedicated subdomains
translator.agents.example.com
assistant.ai.company.com
agent123.agents.example.com
~~~

## Resource Location via DNS

This specification does not use URL paths for version expression. All version and endpoint selection is controlled by DNS records:

1. Obtain (from Discovery Layer or local policy) a candidate Agent Identifier (FQDN)
2. Query DNS SVCB records -> obtain version, endpoint, and protocol information
3. If SVCB is unavailable, use A/AAAA resolution of the Agent Identifier as the default Agent Endpoint
4. Query DNS TXT records (if present) -> obtain optional application-layer security and descriptor metadata
5. Apply local security policy (e.g., DNSSEC validation and/or TXT signature validation)
6. Connect to the selected endpoint and interact according to the selected protocol specification

DN-ANR provides only deterministic resolution and verification for an already-selected identifier; it does not perform semantic discovery or ranking.

# DNS Record Design

This specification keeps DNS payloads minimal and operationally stable. DNS data is classified as MUST/SHOULD/MAY to separate core resolution from optional optimization.

## Mandatory DNS Data (MUST)

- **A/AAAA** {{RFC1035}}: provide baseline reachability and interoperability for resolvers and clients.

Rationale: A/AAAA guarantees minimum connectability and provides a default Agent Endpoint when no SVCB policy is available.

## Recommended DNS Data (SHOULD)

- **SVCB** {{RFC9460}} {{RFC9461}} with endpoint parameters and address hints (`ipv4hint`, `ipv6hint`): provide deterministic endpoint selection, protocol/version signaling, and reduced lookup latency.
- **DNSSEC** {{RFC4033}}: provide origin authentication and integrity for DNS RRsets.
- **TXT identity anchor** {{RFC1035}}: publish optional application-layer security metadata and optional descriptor pointers.

Rationale: SVCB and DNSSEC substantially improve determinism, performance, and security. TXT metadata supports alternative or additional security models and descriptor linkage when needed.

## Optional DNS Data (MAY)

- **TXT signature fields** (`alg`, `pk`, `sig`): used when signature-based verification is enabled.
- **TXT SVCB integrity digest** (`svcb-digest`): optional integrity cross-check material, especially for HTTPS fallback workflows.
- **TXT descriptor pointer fields** (`agent-desc`, `agent-desc-sha256`): pointer + digest for heavy external metadata.

Rationale: Heavy metadata evolves quickly and can grow large; keeping it out of DNS preserves DNS efficiency while retaining verifiable linkage.

## TXT Record: Identity Anchor (Conditional Metadata)

TXT records {{RFC1035}} provide optional application-layer metadata. Their responsibilities are strictly limited to:

1. Declare identity metadata (e.g., `v`, `kid`)
2. Optionally publish key/signature material (`alg`, `pk`, `sig`) for signature-based security
3. Optionally publish SVCB digest (`svcb-digest`) for integrity cross-check, especially with HTTPS fallback
4. Optionally publish external descriptor pointer metadata (`agent-desc`, `agent-desc-sha256`)

### TXT Record Format

~~~
_agent.translator.example.com. IN TXT (
  "v=1;"
  "kid=key-2025-01;"
  "alg=Ed25519;"                                       ; OPTIONAL
  "pk=base64-encoded-public-key;"                      ; OPTIONAL
  "sig=base64-encoded-signature;"                      ; OPTIONAL
  "svcb-digest=base64-encoded-sha256-digest;"          ; OPTIONAL
  "agent-desc=https://translator.example.com/.well-known/agent-descriptor.json;" ; OPTIONAL
  "agent-desc-sha256=x48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE="                 ; OPTIONAL
)
~~~

### TXT Field Descriptions

| Field | Description |
|-------|-------------|
| v | Version identifier, fixed as 1 |
| kid | Key identifier, used for key rotation |
| alg | Signature algorithm: Ed25519 (RECOMMENDED) or ES256 (OPTIONAL; REQUIRED when `sig` is present) |
| pk | Base64-encoded public key (OPTIONAL; REQUIRED when `sig` is present) |
| sig | Signature over selected TXT content (OPTIONAL; used in signature-based security mode) |
| svcb-digest | Base64-encoded SHA-256 digest of canonicalized SVCB records (OPTIONAL; useful for HTTPS fallback integrity cross-check) |
| agent-desc | Descriptor URI for external heavy metadata (OPTIONAL) |
| agent-desc-sha256 | Base64-encoded SHA-256 digest of descriptor content (OPTIONAL; RECOMMENDED when `agent-desc` is present) |

## SVCB Record: Version Distribution and Protocol Negotiation

SVCB (Service Binding) records {{RFC9460}} are the core resolution mechanism, serving the following responsibilities:

| Level | SVCB Role |
|-------|-----------|
| Service Location | TargetName + port specify the service endpoint |
| Version Distribution | Private SvcParam declares agent version |
| Protocol Negotiation | Private parameters declare supported agent protocols |
| Performance Optimization | `ipv4hint` / `ipv6hint` reduce additional address lookups |

### SVCB Record Example

~~~
# Complete SVCB record example
_agent.translator.example.com. IN SVCB 1 agent-v3.example.com. (
  alpn=h2
  port=443
  ipv4hint=203.0.113.50
  ipv6hint=2001:db8::50
  key65480="v3"              ; Agent version
  key65481="a2a,anp"         ; Supported agent protocols
)

# v2 version (lower priority)
_agent.translator.example.com. IN SVCB 2 agent-v2.example.com. (
  alpn=h2
  port=443
  ipv4hint=203.0.113.51
  key65480="v2"
  key65481="a2a"
)
~~~


## Version and Protocol Resolution

### SVCB Private Parameters

This specification introduces private SVCB parameters (SvcParam) as defined in {{RFC9460}}:

| Parameter | Semantics | Example |
|-----------|-----------|---------|
| key65480 | Agent version | "v3", "v2.1.0" |
| key65481 | Agent protocols | "a2a", "a2a,anp" |

#### Version Selection Behavior

Clients can:

- **Default selection**: When version is not specified, the highest priority version based on SVCB priority is used
- **Specific selection**: Specify key65480 value to select a particular version
- **Protocol filtering**: Select only versions supporting specific protocols (key65481)

### ALPN Usage

ALPN is used for TLS-layer protocol negotiation (e.g., h2, h3). Agent interaction protocols ({{A2A}}, {{ANP}}) are declared via SVCB private parameters (key65481), not ALPN values. This ensures compatibility with existing TLS ecosystems and reserves space for future IANA registration.

### Relationship Between Version and Protocol

This specification clearly distinguishes two layers:

| Layer | Declaration Location | Example |
|-------|---------------------|---------|
| Agent Version | key65480 | v3, v2.1.0 |
| Agent Protocol | key65481 | a2a, anp |

### External Descriptor Locator and Digest in TXT (Optional)

DN-ANR supports optional linkage to heavy external metadata while keeping DNS payloads minimal:

- `agent-desc` in TXT contains an absolute URI that identifies a descriptor resource.
- `agent-desc-sha256` in TXT contains the SHA-256 digest of the descriptor in Base64 encoding.

DN-ANR standardizes only:

- URI syntax and transport locator semantics.
- Digest algorithm (SHA-256) and digest encoding.
- Client verification flow (fetch descriptor -> compute digest -> compare -> consume).

DN-ANR does not standardize descriptor content schema (capability model, OpenAPI, model card, I/O schema, etc.).

Descriptor digest computation rules:

1. Fetch descriptor bytes from the URI in `agent-desc`.
2. If the descriptor media type is JSON, canonicalize using JCS {{RFC8785}} before hashing.
3. For non-JSON media types, hash the raw octet stream as retrieved.
4. Compute SHA-256 and Base64-encode the result.
5. Compare with `agent-desc-sha256`; mismatch MUST be treated as verification failure.

### Interoperability Gating for Descriptor-Dependent Clients

- A client that depends on descriptor data MUST require `agent-desc`; otherwise it MUST treat descriptor-based logic as unavailable.
- A client that requires descriptor-integrity verification MUST require both `agent-desc` and `agent-desc-sha256`.
- A publisher that wants interoperable descriptor verification SHOULD publish both TXT fields together.

# Performance and Determinism

## Why Address Hints

SVCB `ipv4hint` and `ipv6hint` improve resolution behavior by:

- reducing extra A/AAAA lookup round-trips;
- improving first-connection determinism;
- reducing resolver-path jitter under recursive caching variance.

## Recommended SVCB Publication Strategy

- Publishers SHOULD keep each SVCB RRSet compact and avoid excessive per-version record expansion.
- When many versions exist, publishers SHOULD keep only stable externally supported versions in DNS and move detailed capability/version matrices to external descriptors (`agent-desc` + `agent-desc-sha256` in TXT).
- Publishers SHOULD keep endpoint migration agility by using shorter TTLs for SVCB than TXT.

## TTL Guidance

- Identity anchors (TXT) SHOULD use relatively longer TTL values.
- Endpoint/control-plane records (SVCB) SHOULD use relatively shorter TTL values to support endpoint migration and rapid rollback.


# HTTPS Fallback Mechanism

To ensure "works by default" behavior, this specification introduces an optional but strongly recommended fallback mechanism.

## agent-dns.json

For clients that do not support SVCB queries, agents can publish a JSON mirror of DNS records at an HTTPS endpoint:

~~~
https://{agent-id}/.well-known/agent-dns.json
~~~

### Media Type

The agent-dns.json file MUST be served with the following HTTP headers:

~~~
Content-Type: application/json; charset=utf-8
Cache-Control: max-age=300
~~~

Servers SHOULD set an appropriate Cache-Control header. A value between 300 seconds (5 minutes) and 3600 seconds (1 hour) is RECOMMENDED.

### JSON Signature

The JSON file MUST include a signature for integrity protection. Unlike the TXT record signature which covers only TXT fields, the JSON signature covers the complete service binding information.

The signature is computed over the canonical JSON representation of the document (excluding the sig field) as defined in {{RFC8785}} (JSON Canonicalization Scheme).

### JSON Schema Definition

The agent-dns.json file MUST conform to the following JSON Schema:

~~~json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/agent-dns.schema.json",
  "title": "Agent DNS JSON",
  "description": "Mirror of DNS records for agent resolution",
  "type": "object",
  "required": ["agentId", "txt", "sig"],
  "properties": {
    "agentId": {
      "type": "string",
      "description": "The FQDN identifying the agent",
      "pattern": "^[a-zA-Z0-9]([a-zA-Z0-9-]*[a-zA-Z0-9])?
                  (\\.[a-zA-Z0-9]([a-zA-Z0-9-]*[a-zA-Z0-9])?)*$"
    },
    "txt": {
      "type": "object",
      "description": "Core identity fields from DNS TXT record",
      "required": ["v", "kid"],
      "properties": {
        "v": {
          "type": "string",
          "const": "1"
        },
        "kid": {
          "type": "string",
          "description": "Key identifier"
        },
        "alg": {
          "type": "string",
          "enum": ["ES256", "Ed25519"],
          "description": "Signature algorithm"
        },
        "pk": {
          "type": "string",
          "description": "Base64-encoded TLS certificate public key"
        }
      }
    },
    "svcb": {
      "type": "array",
      "description": "Mirror of DNS SVCB records",
      "items": {
        "type": "object",
        "required": ["priority", "target", "port"],
        "properties": {
          "priority": {
            "type": "integer",
            "minimum": 1,
            "maximum": 65535
          },
          "target": {
            "type": "string",
            "description": "Target hostname"
          },
          "port": {
            "type": "integer",
            "minimum": 1,
            "maximum": 65535
          },
          "alpn": {
            "type": "array",
            "items": {
              "type": "string"
            }
          },
          "agentVersion": {
            "type": "string",
            "description": "Agent version (mirrors key65480)"
          },
          "agentProtocols": {
            "type": "array",
            "items": {
              "type": "string"
            },
            "description": "Supported protocols (mirrors key65481)"
          }
        }
      }
    },
    "sig": {
      "type": "string",
      "description": "Base64-encoded signature over canonical JSON
                                           (excluding sig field)"
    }
  }
}
~~~

### File Structure Example

~~~json
{
  "agentId": "translator.example.com",
  "txt": {
    "v": "1",
    "kid": "key-2025-01",
    "alg": "ES256",
    "pk": "MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE..."
  },
  "svcb": [
    {
      "priority": 1,
      "target": "agent-v3.example.com",
      "port": 443,
      "alpn": ["h2"],
      "agentVersion": "v3",
      "agentProtocols": ["a2a", "anp"]
    },
    {
      "priority": 2,
      "target": "agent-v2.example.com",
      "port": 443,
      "alpn": ["h2"],
      "agentVersion": "v2",
      "agentProtocols": ["a2a"]
    }
  ],
  "sig": "MEUCIQC7..."
}
~~~

### JSON Signature Computation {#json-sig}

The signature over the JSON file is computed as follows:

1. Construct the JSON object without the sig field.
2. Serialize using JSON Canonicalization Scheme (JCS) as defined in {{RFC8785}}.
3. Compute the signature using the TLS private key.
4. Encode the signature using Base64.

~~~
json_without_sig = { agentId, txt, svcb }
canonical_json = JCS(json_without_sig)
signature = Sign(TLS_private_key, UTF-8(canonical_json))
sig = Base64Encode(signature)
~~~

### JSON Signature Verification

Clients MUST verify the JSON signature:

1. Fetch the JSON file over HTTPS.
2. Extract the sig field and remove it from the object.
3. Serialize the remaining object using JCS.
4. Obtain the public key from the txt.pk field in the JSON.
5. Verify the signature.
6. (RECOMMENDED for TLS-based signing) Verify that txt.pk matches the TLS certificate's public key.

If verification fails, the client MUST reject the JSON file.

Note: When agent providers use separate key pairs (not TLS-based), the verification in step 6 is not applicable. In such cases, the integrity of the JSON file depends on the authenticity of the public key in txt.pk, which has the same trust anchor limitations as described in the Security Model Overview.

## Design Principles

- **Mirror, not addition**: JSON only mirrors information already in DNS; it does not introduce content absent from DNS
- **DNS remains authoritative**: HTTPS JSON is only a "readable mirror", not a new authoritative source
- **Signature required**: JSON files MUST be signed for integrity protection
- **Schema validated**: Clients SHOULD validate JSON against the defined schema

## Applicable Scenarios

- Clients that do not support SVCB queries
- Browsers / debugging tools
- Early ecosystem transition period
- Environments where DNS resolution is limited


# Security {#security}

This section defines the security mechanisms for ensuring the integrity and authenticity of agent resolution data.

## Security Model Overview {#security-model}

This specification provides two complementary mechanisms for ensuring integrity and authenticity of agent resolution data:

1. **DNSSEC (RECOMMENDED for Internet-facing deployments)**: Protocol-level cryptographic authentication of DNS data
2. **Signature-based Security (OPTIONAL)**: TXT key/signature validation (`pk`, `sig`) with optional digest cross-checks (`svcb-digest`); if HTTPS fallback JSON is used, fallback signature validation is REQUIRED

| Mechanism | Protection Scope | Trust Anchor |
|-----------|-----------------|--------------|
| DNSSEC | All DNS records (TXT, SVCB, A/AAAA) | DNS root zone |
| Signature-based Security | TXT signed fields, optional SVCB digest consistency, JSON fallback signature (when fallback is used) | Web PKI (when using TLS keys) or self-declared (when using separate keys) |

### DNSSEC-based Security (RECOMMENDED)

DNSSEC {{RFC4033}} provides cryptographic authentication of DNS data at the protocol level.

### DNSSEC Deployment Recommendations

- For publicly reachable agents, the authoritative zone SHOULD deploy DNSSEC.
- When DNSSEC validation is available and the SVCB RRSet (or TXT RRSet, when used) validates as **bogus**, clients MUST treat resolution as failure (fail-closed) and MUST NOT use that endpoint.
- Clients SHOULD apply stricter fail-closed behavior at least to SVCB and TXT (when TXT is part of the selected trust path).
- In enterprise/private networks where DNSSEC is not deployed, operators MAY rely on TXT signatures and TLS certificate binding as a minimum trust baseline.

When DNSSEC is enabled:

- All DNS records are signed by the zone's DNSSEC keys.
- Clients with DNSSEC validation can verify record authenticity.
- Application-layer signatures remain useful for defense in depth and for JSON fallback integrity.

### Signature-based Security (OPTIONAL but RECOMMENDED)

This specification defines an optional but recommended signing mechanism for integrity protection. Agent providers have two options for key management:

#### Option 1: TLS Certificate Keys (RECOMMENDED)

Using the domain's TLS certificate keys provides a complete trust chain:

- Uses the domain's TLS certificate private key for signing
- Public key is published in the TXT record (pk field)
- Enables verification through the established Web PKI trust chain
- Clients can verify that pk matches the TLS certificate presented during HTTPS connection

When TLS-based signing is used:

1. The TXT record contains the TLS certificate's public key
2. A signature covers the selected TXT fields
3. `svcb-digest` MAY be included as optional integrity cross-check material (especially for HTTPS fallback consistency checks)
4. Clients can verify the signature using the public key from the TLS certificate chain

#### Option 2: Separate Key Pair

Agent providers MAY use a separate key pair (not derived from TLS certificates) for signing:

- Agent provider generates and manages their own key pair
- Public key is published in the TXT record (pk field)
- Signature is computed using the corresponding private key

**Trust Anchor Limitation**: When using separate keys, the trust anchor is limited to the TXT record itself. If the TXT record is tampered with (e.g., via DNS spoofing or cache poisoning), an attacker could replace both the public key and signature, rendering the integrity protection ineffective. This is because:

- The public key in the TXT record is self-declared without external verification
- Clients have no independent trust anchor to verify the authenticity of the public key
- SVCB records and agent-dns.json cannot be reliably verified if the TXT record is compromised

For this reason, when using separate keys:

- DNSSEC deployment becomes more important to protect the TXT record itself
- Clients SHOULD treat records from non-DNSSEC zones with appropriate caution
- Out-of-band key distribution mechanisms MAY be used to establish trust

### Choosing a Security Mechanism

| Scenario | Recommended Approach |
|----------|---------------------|
| DNSSEC fully deployed | DNSSEC alone is sufficient |
| DNSSEC not available | Use TLS-based signing (Option 1) |
| High security requirements | Use both DNSSEC and signing (defense-in-depth) |
| HTTPS fallback required | JSON signing and/or `svcb-digest` consistency checks are recommended |
| Separate keys without DNSSEC | Limited trust; consider additional verification mechanisms |

## SVCB Integrity Digest (Optional) {#svcb-integrity}

When `svcb-digest` is present in TXT, SVCB records can be cross-checked for integrity (for example, during HTTPS fallback reconciliation). This section defines the canonicalization and digest computation procedures.

### SVCB Canonicalization

To compute the svcb-digest, SVCB records MUST be canonicalized as follows:

#### Step 1: Collect and Sort

1. Collect all SVCB records for the agent's _agent prefix.
2. Exclude AliasMode records (priority = 0).
3. Sort records by priority in ascending order (lowest first).
4. If priorities are equal, sort by TargetName lexicographically.

#### Step 2: Normalize Each Record

For each SVCB record, construct a canonical string in the following format:

~~~
<priority> <target> <params>
~~~

Where:
- priority: Decimal integer with no leading zeros
- target: Fully qualified domain name in lowercase, with trailing dot removed
- params: SvcParams in sorted order by key number, formatted as key=value

#### Step 3: SvcParam Normalization

SvcParams MUST be normalized as follows:

1. Sort by SvcParamKey number (ascending).
2. Format each parameter as: `key<number>=<value>`
3. String values are enclosed in double quotes.
4. List values (e.g., alpn) use comma separation with no spaces.
5. Separate parameters with a single space.

#### Canonical Format Example

~~~
# Original SVCB records:
_agent.translator.example.com. IN SVCB 2 agent-v2.example.com. (
  alpn=h2 port=443 key65480="v2" key65481="a2a"
)
_agent.translator.example.com. IN SVCB 1 agent-v3.example.com. (
  alpn=h2 port=443 key65480="v3" key65481="a2a,anp"
)

# Canonical representation (sorted by priority):
1 agent-v3.example.com key1=h2 key3=443 key65480="v3" key65481="a2a,anp"
2 agent-v2.example.com key1=h2 key3=443 key65480="v2" key65481="a2a"
~~~

Note: alpn is SvcParamKey 1, port is SvcParamKey 3 as defined in {{RFC9460}}.

### Digest Computation

~~~
canonical_svcb = <line1> + "\n" + <line2> + "\n" + ...
digest_bytes = SHA-256(UTF-8(canonical_svcb))
svcb-digest = Base64Encode(digest_bytes)
~~~

The resulting svcb-digest is approximately 44 characters (32 bytes encoded in Base64).


## Signature Specification {#signing-spec}

This section defines the signature mechanism when signature-based security is used.

### Public Key Requirements

The pk field contains the public key used for signature verification. There are two options:

#### When Using TLS Certificate Keys (RECOMMENDED)

The pk field MUST contain the public key from the domain's TLS certificate:

1. Extract the SubjectPublicKeyInfo from the TLS certificate.
2. Encode using Base64 {{RFC4648}}.
3. The certificate MUST be valid for the agent's domain name.

#### When Using Separate Key Pair

The pk field contains the agent provider's self-managed public key:

1. Generate a key pair using a supported algorithm.
2. Extract the public key in SubjectPublicKeyInfo format.
3. Encode using Base64 {{RFC4648}}.

Note: When using separate keys, the public key is self-declared and lacks an independent trust anchor. See Security Model Overview for implications.

#### Supported Key Types

- EC P-256 (for ES256 algorithm) - RECOMMENDED
- Ed25519 (for Ed25519 algorithm)

### Signature Input Construction

When signature-based TXT validation is used, the signature input MUST be constructed from TXT fields as follows:

1. Include required fields in this exact order: `v`, `kid`, `alg`, `pk`.
2. If present, append optional fields in this exact order: `svcb-digest`, `agent-desc`, `agent-desc-sha256`.
3. Use `key=value` pairs separated by semicolons, with no trailing semicolon.

~~~
signing_input = "v=" + v + ";kid=" + kid + ";alg=" + alg + ";pk=" + pk
if svcb-digest present: signing_input += ";svcb-digest=" + svcb-digest
if agent-desc present: signing_input += ";agent-desc=" + agent-desc
if agent-desc-sha256 present: signing_input += ";agent-desc-sha256=" + agent-desc-sha256
~~~

Example:
~~~
v=1;kid=key-2025-01;alg=ES256;pk=MFkwEwYHKoZI...;agent-desc=https://translator.example.com/.well-known/agent-descriptor.json
~~~

### Signature Generation

~~~
signature_bytes = Sign(private_key, UTF-8(signing_input))
sig = Base64Encode(signature_bytes)
~~~

Where private_key is either:
- The TLS certificate's private key (Option 1, RECOMMENDED), or
- The agent provider's separately managed private key (Option 2)

For ES256: signature is 64 bytes (r || s format), resulting in 88 Base64 characters.
For Ed25519: signature is 64 bytes, resulting in 88 Base64 characters.

### Signature Verification Procedure

Clients MUST perform the following steps:

1. Parse TXT record and extract `v`, `kid`, `alg`, `pk`, `sig`, and any optional signed fields present.
2. Reconstruct `signing_input` using the required/optional field ordering defined above.
3. Decode pk from Base64 to obtain the public key.
4. Decode sig from Base64 to obtain the signature bytes.
5. Verify the signature using the specified algorithm.
6. (RECOMMENDED for Option 1) Verify that pk matches the TLS certificate presented during connection.

If verification fails, the client MUST reject the TXT record.

When using Option 2 (separate key pair), clients should be aware that the signature only proves consistency between the TXT record content and the private key holder. Without DNSSEC or TLS binding, there is no external trust anchor to verify the key's authenticity.

### TLS Certificate Binding Verification (Option 1 Only)

When TLS certificate keys are used (Option 1), clients SHOULD verify that the pk in the TXT record matches the server's TLS certificate:

1. Establish TLS connection to the agent's domain.
2. Extract the public key from the server's certificate.
3. Compare with the pk field in the TXT record.
4. If mismatch, treat as verification failure.

This binding ensures that the entity controlling the TLS private key is the same entity that published the DNS records.

Note: This verification is not applicable when separate key pairs are used (Option 2), as the pk in the TXT record will not match the TLS certificate.


# Implementation Checklist

## For Agent Publishers

1. Prepare domain name, configure HTTPS and TLS certificate
2. Configure DNS A/AAAA records (basic connectivity)
3. (OPTIONAL) Configure DNS TXT record (_agent.xxx) for signature metadata (`alg`/`pk`/`sig`), `svcb-digest`, and/or descriptor pointer fields
4. Configure DNS SVCB records with endpoint, protocol/version, and (SHOULD) address hints
5. (OPTIONAL) Publish descriptor URI + digest in TXT (`agent-desc`, `agent-desc-sha256`) for heavy metadata externalization
6. (RECOMMENDED) Publish /.well-known/agent-dns.json fallback file
7. (RECOMMENDED for public deployments) Enable DNSSEC

## For Client Developers

1. Query SVCB records, parse version and endpoint information
2. If SVCB is unavailable, use A/AAAA of the Agent Identifier as the default endpoint
3. Query TXT records (if present), parse optional fields (`pk`, `sig`, `svcb-digest`, `agent-desc`, `agent-desc-sha256`)
4. If descriptor fields are present and required by local policy, fetch descriptor and verify digest before use
5. (Fallback) If SVCB unavailable, fetch agent-dns.json when needed
6. Connect to endpoint per agent protocol (key65481) specification
7. Validate DNSSEC when present, and fail closed for bogus SVCB/TXT results that are part of the selected trust path

## DNS Record Configuration Example

~~~
; Basic connectivity
translator.example.com.    IN A     203.0.113.50
translator.example.com.    IN AAAA  2001:db8::50

; Optional TXT identity/security/descriptor metadata
_agent.translator.example.com. IN TXT "v=1;kid=key-2025-01;
                    alg=Ed25519;pk=...;sig=...;
                    svcb-digest=...;
                    agent-desc=https://translator.example.com/.well-known/agent-descriptor.json;
                    agent-desc-sha256=x48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE="

; Version resolution (SVCB)
_agent.translator.example.com. IN SVCB 1 agent-v3.example.com. (
  alpn=h2 port=443
  ipv4hint=203.0.113.50 ipv6hint=2001:db8::50
  key65480="v3" key65481="a2a,anp"
)
_agent.translator.example.com. IN SVCB 2 agent-v2.example.com. (
  alpn=h2 port=443 ipv4hint=203.0.113.51 key65480="v2" key65481="a2a"
)
~~~


# Security Considerations

This specification uses DNS as the authoritative source for agent resolution and identity information. Its security objectives are to ensure the authenticity, integrity, and verifiability of resolution results, rather than evaluating agent service quality or behavioral trustworthiness.

## Threat Model

This specification primarily considers the following threats:

- DNS poisoning or cache pollution leading to incorrect endpoint resolution
- Tampering with resolution results to redirect clients to unintended endpoints
- Downgrade attacks inducing clients to use older versions or weaker protocols
- Trust violations caused by expired or replaced identity declarations

## Mandatory Security Requirements

To address the above threats, this specification mandates:

- Clients MUST establish at least one validated integrity path before endpoint use: DNSSEC validation, or TXT signature verification when TXT signing fields are used
- Clients MUST perform TXT-SVCB consistency checks when `svcb-digest` is present and selected by local policy
- Clients MUST use TLS {{RFC8446}} and verify server certificates {{RFC9525}}
- Clients MUST NOT use endpoints that fail verification
- Agents that publish `svcb-digest` or TXT signatures over endpoint-related metadata MUST synchronously update TXT and SVCB information when versions or endpoints change

## Deployment Recommendations

- For Internet-facing agent domains, authoritative operators SHOULD enable DNSSEC {{RFC4033}}.
- If DNSSEC data is present and validates as bogus for SVCB (or TXT, when TXT is part of the selected trust path), clients MUST fail closed for that endpoint.
- For private/enterprise deployments without DNSSEC, clients SHOULD require TXT signature verification and TLS certificate validation as minimum controls.
- This specification does not require DNSSEC as the only trust mechanism; deployments MAY combine DNSSEC and signature-based protections.

## Specification Scope

This specification guarantees the following properties:

- Verifiability of agent identity
- Integrity and consistency of resolution results
- Encryption and tamper-proofing of connections

This specification does NOT attempt to address:

- Agent capability authenticity
- Service quality (SLA) or behavioral compliance
- Agent reputation or governance issues

These concerns should be handled by upper-layer protocols, operational frameworks, or governance mechanisms.


# IANA Considerations

This document requests IANA registration of the following SVCB SvcParamKeys:

| Number | Name | Meaning | Reference |
|--------|------|---------|-----------|
| 65480 | agent-version | Agent version identifier | This document |
| 65481 | agent-protocols | Comma-separated list of supported agent protocols | This document |

Note: The values 65480-65481 are in the private use range (65280-65534) as defined in {{RFC9460}}. Upon publication, these should be replaced with IANA-assigned values from the Expert Review range.


--- back

# Acknowledgments
{:numbered="false"}
