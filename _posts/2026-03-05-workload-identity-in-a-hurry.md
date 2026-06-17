---
title: "Workload Identity in a Hurry: SPIFFE, SPIRE & TPM"
categories: Learn in a hurry
author:
- Yuva Peavler
excerpt: "How machines prove their identity without passwords. Covers SPIFFE IDs, SPIRE architecture, X.509 SVIDs, mTLS between workloads, and TPM-based attestation."
---

If you need a networking primer before diving in, check out [Networking in a Hurry]({{ site.baseurl }}{% post_url 2026-01-01-networking-in-a-hurry %}).
If you need a Kubernetes primer, check out [Kubernetes in a Hurry]({{ site.baseurl }}{% post_url 2026-01-02-kubernetes-in-a-hurry %}).
If you need a PKI and TLS primer, check out [Cryptographic Basics in a Hurry]({{ site.baseurl }}{% post_url 2026-03-05-cryptographic-basics-in-a-hurry %}).


> **Analogy key**
>
> Throughout this post, a consistent **passport/border control** analogy is used.

| Concept | Analogy |
| --- | --- |
| Trust domain | A country |
| SPIFFE ID | Passport number |
| SVID (X.509 cert) | The passport booklet |
| Root CA cert | The government's master signing authority |
| Intermediate CA cert | A regional passport office |
| Leaf cert / SVID | The individual passport issued to a person |
| Trust bundle | The border agent's reference book of accepted foreign passports |
| SPIRE Server | The passport office |
| SPIRE Agent | The border control officer at the gate |
| Node attestation | Proving citizenship to get a first passport |
| Workload attestation | The border officer checking face, fingerprints, and visa |
| Federation | A bilateral agreement - country A agrees to accept country B's passports |

---

## Part 1: Workload Identity

---

<a id="secret-zero"></a>

#### Q: What is the "secret zero" problem?

Imagine hiring a new employee and needing to give them a building access badge. But to get the badge, they need to already be inside the building to fill out the form. That's the secret zero problem.

In software: to give a workload a secret (like a database password or an API key), that secret must first get onto the workload securely. But how do you do that without already having a secure channel? Every traditional approach — baking secrets into images, injecting them via environment variables, using a bootstrap token — just pushes the problem one level earlier. The first secret is always the hardest.

---

<a id="workload-identity"></a>

#### Q: What does "workload identity" actually mean?

A workload identity is a cryptographically verifiable name for a piece of software, tied to *what it is* (its code, its execution context, its platform) rather than *where it is* (its IP, its hostname).

A workload identity looks like this:

```
spiffe://global.trust.example/platform/unix/workload/crowdstrike-agent
```

That URI is the identity. The cryptographic proof of that identity is an X.509 certificate whose Subject Alternative Name contains that URI — for a refresher on X.509 certificates, certificate chains, and CAs, see [Cryptographic Basics in a Hurry]({{ site.baseurl }}{% post_url 2026-03-05-cryptographic-basics-in-a-hurry %}).

---

<a id="attestation"></a>

#### Q: What is attestation, and what does "attestation-based identity" mean?

**Attestation** is the process of proving identity by presenting verifiable evidence about *what you are*, rather than by presenting a pre-shared secret. The evidence comes from properties of the environment that are hard or impossible to forge — hardware measurements, platform metadata, or kernel-visible process attributes.

- Traditional identity: "Here is the password I was given. Trust me."
- Attestation-based identity: "Here is cryptographic evidence of what I am. Verify it yourself."

In SPIFFE/SPIRE this happens at two levels:

**Node attestation** — when a SPIRE agent starts on a new machine, it must prove to the SPIRE server that the machine is what it claims to be. On bare metal or VMs, this is done using a TPM (Trusted Platform Module): the TPM holds a hardware-burned key that cannot be extracted, and the agent uses it to produce a signed proof. On cloud VMs, the cloud provider's instance identity document plays the same role.

**Workload attestation** — when a workload calls the SPIRE agent's local socket to request an SVID, the agent interrogates the kernel: what is the PID of this caller? What namespace, service account, or binary is it running as? These kernel-visible facts are the workload's attestation evidence. The agent matches them against registered selectors and, if they match, issues the SVID. The workload never presented a password — the agent observed it.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    A["Machine boots"] --> B["SPIRE agent presents TPM proof\nto SPIRE server"]
    B -->|"node attestation"| C["Server issues node SVID to agent"]
    C --> D["Workload calls agent socket"]
    D --> E["Agent checks kernel:\n'who is this process?'"]
    E -->|"workload attestation"| F["Agent issues workload SVID"]
</div>

This is why attestation-based identity solves the secret zero problem: there is no first secret to protect. The identity is earned from evidence that already exists in the environment.

---

<a id="what-is-spiffe"></a>

#### Q: What is SPIFFE?

SPIFFE (Secure Production Identity Framework for Everyone) is an open standard — think of it like HTTP, but for workload identity. It defines:

1. What a workload identity looks like (the SPIFFE ID)
2. How to cryptographically prove that identity (the SVID)
3. How to verify identities across organizational boundaries (federation)
4. The API a workload uses to get its identity (the Workload API)

SPIFFE itself is just a spec. SPIRE is the reference implementation, the same way Apache or nginx implement HTTP.

See: [spiffe.io/docs/latest/spiffe-about/overview](https://spiffe.io/docs/latest/spiffe-about/overview/)

---

<a id="spiffe-id"></a>

#### Q: What is a SPIFFE ID?

A SPIFFE ID is a URI of the form `spiffe://<trust-domain>/<path>`. The trust domain is the country that issued it; the path is the unique identifier within that country.

Examples:
- Machine: `spiffe://global.trust.example/provider/aws/region/us-east-1/host/aws-node-1.example.com`
- Workload: `spiffe://global.trust.example/platform/unix/workload/crowdstrike-agent`

A SPIFFE ID is **not a secret**. It's a name. What makes it trustworthy is the X.509 certificate (SVID) attached to it — signed by the trust domain's CA.

---

<a id="what-is-svid"></a>

#### Q: What is an SVID?

An SVID (SPIFFE Verifiable Identity Document) is the cryptographic proof of a SPIFFE ID. It is a leaf X.509 certificate where the Subject Alternative Name (SAN) field contains the SPIFFE ID URI.

The SVID is the passport booklet. The SPIFFE ID is the passport number printed inside it. The CA's signature is the holographic seal.

SVIDs also come in a JWT form for HTTP/REST APIs, but X.509 SVIDs are the primary form used for mTLS between services. They are **short-lived** and **auto-rotated** by the SPIRE agent — the workload never manages renewal.

---

<a id="why-short-lived"></a>

#### Q: Why are SVIDs short-lived?

Short-lived certificates are a deliberate security design:

- **Revocation is hard** — CRL/OCSP is slow and unreliable. Short TTLs make revocation unnecessary: a compromised cert expires on its own.
- **Rotation is automatic** — SPIRE agents rotate SVIDs before they expire, transparently. The workload never manages renewal.
- **Blast radius is limited** — a leaked SVID is useful to an attacker for minutes.

Passports that expired every 4 hours and were automatically renewed while you slept. A stolen passport would be useless by morning. SPIRE SVIDs default to a 1-hour TTL and are rotated at the 50% mark (30 minutes).

---

<a id="trust-bundle"></a>

#### Q: What is a trust bundle?

A trust bundle is the set of root CA certificates trusted by a workload — the anchors used to verify any certificate chain. When a service presents its leaf cert, the chain is walked up to a root and checked: "is this root in my trust bundle?"

What a workload actually receives via the Workload API:
- A set of **X.509 CA certificates** — used to verify X.509 SVIDs from that trust domain
- A set of **JWT public keys** — used to verify JWT SVIDs from that trust domain

The border agent's reference book: it contains the official seal patterns for every country whose passports have been agreed to accept. The critical rule: a bundle must always be stored alongside the name of the trust domain it represents. When verifying an SVID, the bundle for *that specific trust domain* must be used, not a pool of all bundles mixed together.

---

<a id="trust-domain"></a>

#### Q: What is a trust domain?

A SPIFFE trust domain is an identity namespace backed by an issuing authority with a set of cryptographic keys. Those keys serve as the cryptographic anchor for all identities in the domain.

The spec defines a **1:N relationship** between a trust domain and its keys — a single trust domain can be backed by multiple keys and key types. This is necessary for root rotation (two keys are valid simultaneously during the transition) and multi-protocol support (separate keys for X.509 SVIDs and JWT SVIDs).

`global.trust.example` is one example trust domain. Any two workloads in the same trust domain can verify each other's SVIDs without additional configuration — they both trust the same set of keys. To trust SVIDs from another trust domain, federation is needed.

The spec strongly advises against sharing cryptographic keys across trust domains. If the same root key backs both `staging.example.com` and `prod.example.com`, a naïve verifier that trusts the key without checking the trust domain name could accept a staging identity as a production identity. The safe practice is one root key per trust domain.

---

<a id="bundle-format"></a>

#### Q: The spec says SPIFFE bundles are "not meant for direct consumption by workloads." Does that mean workloads don't get the bundle?

No — workloads absolutely do receive bundle contents. That sentence is about the *format*, not the *distribution*.

The SPIFFE bundle format (JWK Set) is a control-plane wire format — designed for SPIRE servers exchanging bundles with each other. The spec is saying: don't expect workloads to parse raw JWK Sets themselves.

What actually happens is a two-layer system:

```
SPIFFE Bundle (JWK Set)          ← control plane format
        ↓
  SPIRE Agent                    ← translates it
        ↓
  Workload API (gRPC stream)     ← delivers X.509 certs and JWT keys
        ↓
  go-spiffe / spiffe-helper      ← presents it as a standard tls.Config / x509.CertPool
        ↓
  workload process               ← never sees a JWK Set
```

The SPIRE agent converts the `x509-svid` JWK entries into `*x509.Certificate` objects and delivers them to the workload as a proper X.509 trust bundle. The workload plugs those directly into its TLS configuration.

---

## Part 2: mTLS — How Two Services Actually Authenticate Each Other

---

<a id="what-is-tls"></a>

#### Q: What is TLS and what does it normally protect?

TLS (Transport Layer Security) is the protocol that encrypts traffic between two parties.

In standard TLS:
- The **server** presents a certificate proving its identity
- The **client** verifies that certificate against a trusted CA
- The client is **anonymous** — the server doesn't know who the client is

This is fine for a browser talking to a website. It's not fine for a microservice talking to another microservice.

---

<a id="what-is-mtls"></a>

#### Q: What is mTLS?

mTLS (mutual TLS) is TLS where **both sides** present and verify certificates.

Standard TLS is like showing a passport at a hotel check-in — only one party proves who they are. mTLS is two diplomats meeting — both show their passports, both verify the other's seal before talking.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant A as service-A
    participant B as service-B

    A->>B: ClientHello
    B->>A: ServerHello + Server Certificate<br/>(proves: "I am service-B,<br/>signed by global.trust.example CA")
    A->>B: Client Certificate<br/>(proves: "I am service-A,<br/>signed by global.trust.example CA")
    A->>B: Finished (encrypted)
    B->>A: Finished (encrypted)
    A->>B: Encrypted application data
    B->>A: Encrypted application data
</div>

After the handshake, both sides have verified the other's identity, established a shared encryption key, and have a fully encrypted channel.

---

<a id="svid-verification"></a>

#### Q: How does a service verify the other side's SVID during mTLS?

When service-A receives service-B's SVID during the TLS handshake, it does three things:

1. **Verify the certificate chain** — walk from the leaf cert up through the intermediate CA to the root CA, verifying each signature. If any link is broken, reject.
2. **Check the trust bundle** — is the root CA in the trust bundle? If not, reject. This is why distributing the trust bundle to all workloads matters — without it, the chain has nothing to anchor to.
3. **Check the SPIFFE ID** — extract the SAN URI from the leaf cert. Is it a valid SPIFFE ID from a recognized trust domain? Does it match the service expected?

---

<a id="spiffe-vs-traditional-mtls"></a>

#### Q: What makes mTLS with SPIFFE SVIDs better than mTLS with traditional certificates?

| Property | Traditional mTLS | SPIFFE/SPIRE mTLS |
|---|---|---|
| Certificate issuance | Manual CSR, human approval, days | Automatic, seconds |
| Certificate lifetime | 1–2 years | 1 hour |
| Rotation | Manual, often skipped | Automatic, transparent |
| Identity tied to | Hostname / IP | What the workload *is* (selectors) |
| Works across clouds | Requires shared PKI setup | Yes, via federation |
| Revocation | CRL/OCSP, slow and unreliable | TTL expiry, no revocation needed |
| Secret zero problem | Still exists | Solved via attestation |

With SPIFFE, the certificate is issued *because the workload proved what it is*, not because a human approved a request. The attestation is the bootstrap.

---

<a id="spire-vs-cert-manager"></a>

#### Q: Does SPIRE make cert-manager redundant?

No — they solve different problems and are commonly used together in the same cluster.

**SPIRE** issues **SVIDs** — short-lived X.509 certificates with a SPIFFE URI in the SAN field. These are used for **workload-to-workload mTLS**. SPIRE handles issuance, rotation, and delivery entirely through its own Workload API socket. cert-manager is not involved.

**cert-manager** handles everything else in Kubernetes that needs a certificate:

| Use case | Who handles it |
|---|---|
| Ingress TLS (HTTPS termination at the edge) | cert-manager |
| Kubernetes webhook TLS (admission controllers) | cert-manager |
| etcd client/server TLS | cert-manager |
| Internal service TLS for non-SPIFFE workloads | cert-manager |
| Workload-to-workload mTLS identity | SPIRE (SVIDs) |

**Vault sits behind both.** cert-manager uses Vault as an issuer to get certs signed by the internal CA. SPIRE uses Vault's PKI engine to obtain its intermediate CA certificate at startup.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    V["Vault\n(Root CA + PKI engine)"]
    CM["cert-manager\n(Kubernetes cert lifecycle)"]
    SP["SPIRE\n(Workload attestation + SVID issuance)"]
    ING["Ingress / Webhook TLS certs\n(stored in K8s Secrets)"]
    SVID["SVIDs\n(delivered via Workload API socket)"]

    V -- "signs intermediate CA for" --> SP
    V -- "issues certs via vault-issuer" --> CM
    CM -- "manages" --> ING
    SP -- "issues" --> SVID
</div>

Running SPIRE for mTLS still requires cert-manager for ingress TLS, webhook TLS, and any other Kubernetes infrastructure that needs certificates.

---

## Part 3: X.509 Certificates and SVIDs

An SVID is a leaf X.509 certificate where the Subject Alternative Name (SAN) field contains a SPIFFE ID URI instead of a hostname:

```
Leaf cert / SVID
  Subject:          CN=crowdstrike-agent
  SAN URI:          spiffe://global.trust.example/platform/unix/workload/crowdstrike-agent
  Issuer:           SPIRE intermediate CA
  Validity:         1 hour
  Signature:        signed by SPIRE's intermediate CA key
```

When a service presents its SVID during an mTLS handshake, the verifier walks this chain upward:
1. Is the leaf cert's signature valid? (Was it signed by the intermediate?)
2. Is the intermediate cert's signature valid? (Was it signed by the root?)
3. Is the root in the trust bundle?

If all three checks pass, the identity is verified.

### Multiple intermediates

The chain can be arbitrarily deep — the verifier keeps walking upward until it hits a cert it trusts. In practice, chains deeper than root → one intermediate → leaf are rare. Every extra intermediate is one more cert that has to be transmitted and verified on every TLS connection.

The most common reason to add a second intermediate is the **online/offline split**:

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    R["Root CA\n(air-gapped, touched once a year)"]
    P["Policy Intermediate\n(HSM, rarely touched)"]
    I["Issuing Intermediate\n(online, rotated frequently)"]
    L["Leaf certs"]

    R -- "signs" --> P -- "signs" --> I -- "signs" --> L
</div>

The root CA is kept air-gapped and never touches the network. It signs a long-lived policy intermediate. That policy intermediate signs short-lived issuing intermediates that are the only CAs actively online. If an issuing intermediate is compromised, revoke it and mint a new one from the policy intermediate — the root never has to come online.

This is the pattern used by public CAs like DigiCert and Let's Encrypt. A future deep-dive post will cover how SPIRE gets its intermediate CA from Vault and how that pattern plays out in practice.

---

## Part 4: SPIRE — The Implementation

---

<a id="spire-architecture"></a>

#### Q: How is SPIRE architected?

SPIRE has one **centralized server** per trust domain and one **distributed agent** per node. This split is deliberate.

**The server is centralized** because identity decisions need a single source of truth. The registration entries, the CA that signs SVIDs, and the node attestation logic all live on the server.

**The agents are distributed** — one per node — because workload attestation must happen locally. The agent inspects the calling process (PID, UID, binary hash, cgroup) using OS kernel APIs. That inspection only works on the same machine as the workload.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
graph TD
    S["SPIRE Server\n(one per trust domain)\n─────────────────\n● CA — signs SVIDs\n● Registration entry DB\n● Node attestation verifier"]

    A1["SPIRE Agent · node-1\n─────────────────\n● workload attestation\n● SVID cache\n● unix socket"]
    A2["SPIRE Agent · node-2\n─────────────────\n● workload attestation\n● SVID cache\n● unix socket"]
    A3["SPIRE Agent · node-3\n─────────────────\n● workload attestation\n● SVID cache\n● unix socket"]

    S -->|"mTLS (agent uses its node SVID)"| A1
    S -->|"mTLS (agent uses its node SVID)"| A2
    S -->|"mTLS (agent uses its node SVID)"| A3
</div>

The agent acts as a **local proxy and cache**. Once attested and with registration entries received from the server, it can issue SVIDs to local workloads without a server round-trip on every request. The server is only in the critical path for the agent's own bootstrap, periodic node SVID renewal, and when registration entries change.

---

<a id="spire-agent-day-to-day"></a>

#### Q: What does the SPIRE agent actually do day-to-day?

Node attestation is just the bootstrap. After the agent has its node SVID, it does five things continuously:

**1. Syncs registration entries from the server**

The agent maintains a local copy of all registration entries that apply to workloads on its node. It polls the server for changes. When a new workload entry is created, the agent picks it up on the next sync — no restart required.

**2. Performs workload attestation on every Workload API call**

When a process connects to the unix socket, the agent inspects it using OS kernel APIs:
- On Linux: reads `/proc/<pid>/` for binary path, UID, GID, cgroup
- In Kubernetes: queries the kubelet API to map the calling process's cgroup to a pod, then reads the pod's namespace and service account

The agent matches the observed properties against its local registration entries. If a match is found, it serves the SVID from its local cache.

The agent **cannot sign SVIDs itself** — the signing key lives on the SPIRE server. The agent generates a keypair and a CSR for each entry it is responsible for, sends those CSRs to the server, and caches the signed SVIDs it gets back. This happens proactively when entries are first synced, so by the time a workload calls the socket the SVID is already in cache — no server round-trip on the hot path.

**3. Issues SVIDs and keeps them fresh**

The agent tracks expiry for every SVID it has issued and proactively renews them before they expire (at the 50% TTL mark by default).

The application pod doesn't do any of this directly. The Workload API is a gRPC streaming API — `FetchX509SVIDs` keeps a long-lived stream open. The SVID-aware library in the app (go-spiffe, spiffe-helper, or the Istio sidecar) opens that stream once and listens on it. When the agent renews an SVID it pushes the new cert down the existing stream. The library swaps it into the `tls.Config` transparently. The application code never sees a rotation event.

**4. Distributes trust bundles**

Along with each SVID, the agent delivers the trust bundle for the workload's trust domain and any federated trust domains configured on the registration entry.

**5. Serves as the only runtime contact point for workloads**

Workloads never talk to the SPIRE server directly. The agent is the sole interface — a local, low-latency unix socket.

```
Server interaction (infrequent):
  Agent startup          → node attestation, receive node SVID
  Every ~30 min          → renew node SVID
  On entry change        → sync updated registration entries
  On CA rotation         → receive new trust bundle

Local work (every Workload API call):
  Process connects       → inspect PID/UID/cgroup via kernel
  Match against entries  → local lookup, no network
  Issue SVID             → request signing from server, cache result
  Stream updates         → push renewed SVIDs and bundles to open connections
```

---

<a id="registration-entry"></a>

#### Q: What is a registration entry, and how are they structured?

A registration entry is a record in the SPIRE server that says: "any workload matching these selectors should receive this SPIFFE ID, parented to this node identity."

The only structural rule SPIRE enforces is that every entry has a **parent**. The root of the tree is always `/spire/server`. Below that, the depth is entirely up to the deployment.

```
/spire/server
    └── Node entry   (attested by the agent)
            └── Workload entry   (minimum viable hierarchy)
```

In practice most deployments add at least one intermediate level for organizational reasons:

```
/spire/server
    └── Machine entry   (hardware attestation)
            └── Cluster entry   (scopes to a specific k8s cluster)
                    └── Workload entry   (the pod's identity)
```

The invariant that matters: **every ancestor in the chain must be currently attested**. If any parent entry fails, all its descendants stop receiving SVIDs — the trust chain breaks at the point of failure.

---

<a id="workload-api"></a>

#### Q: What is the Workload API?

The Workload API is a gRPC service served by the SPIRE Agent over a unix socket. A workload calls it to get its SVID and trust bundles.

The socket path is configured per trust domain, for example:
```
unix:///run/spire-agent/global.trust.example/public/api.sock
```

**The workload does not authenticate to the agent.** The agent authenticates the workload by inspecting the calling process (its UID, binary path, SHA-256 hash, Kubernetes service account, etc.) and matching it against registered selectors.

No secret is passed. The workload walks up to the border desk. The officer checks the face, the fingerprints, the visa stamp. If everything matches, the passport is handed over.

---

<a id="selectors"></a>

#### Q: What are selectors, and how do they differ by platform?

Selectors are the observable properties of a workload that the SPIRE agent uses to decide whether to issue an SVID. All selectors on a registration entry must match — they are ANDed together.

| Platform | Selector | Example |
|---|---|---|
| Unix process | `unix:uid` | `unix:uid:1000` |
| Unix process | `unix:sha256` | `unix:sha256:abc123...` |
| Systemd | `systemd:id` | `systemd:id:crowdstrike.service` |
| Kubernetes | `k8s:ns` | `k8s:ns:monitoring` |
| Kubernetes | `k8s:sa` | `k8s:sa:prometheus` |
| Kubernetes (PSAT) | `k8s_psat:cluster` | `k8s_psat:cluster:prod-cluster-1` |
| TPM (on-prem node) | `tpm:pub_hash` | `tpm:pub_hash:deadbeef...` |
| AWS node | `aws_iid:instance:id` | `aws_iid:instance:id:i-0abc123` |
| Azure node | `azure_msi:resource-group` | `azure_msi:resource-group:my-rg` |

---

<a id="node-vs-workload-attestation"></a>

#### Q: What is the difference between node attestation and workload attestation?

**Node attestation** — "Is this a legitimate node that should run a SPIRE agent?"

Happens once at agent startup. The agent proves to the server it's running on a real, authorized machine — using TPM, cloud instance metadata, or a join token. If it passes, the agent gets a node SVID and the machine entry in SPIRE is considered attested.

Proving citizenship before the passport office will issue a passport. Show the birth certificate, national ID, biometrics. This happens once.

**Workload attestation** — "Is this the right process on this node?"

Happens every time a workload calls the Workload API. The agent inspects the calling process and matches its properties against registered selectors. If it matches, the workload gets a workload SVID.

The border officer checking face, fingerprints, and visa at the gate. This happens every time a border is crossed.

The node SVID is the *parent* of all workload SVIDs on that node. If node attestation fails, no workloads on that node get SVIDs — the entire chain collapses.

---

## Part 5: Federation

---

<a id="spiffe-federation"></a>

#### Q: What is SPIFFE federation?

Federation is how two separate trust domains establish mutual trust so their workloads can verify each other's SVIDs.

Two countries sign a bilateral visa agreement. Citizens of country A can enter country B because B trusts A's passport office. Neither country shares their CA private key — they only exchange trust bundles. Border agents use the other country's bundle to verify incoming passports.

A workload can declare `federated_trust_domains: ["prod.trust.example"]`. SPIRE includes the `prod.trust.example` trust bundle in the SVID response so the workload can verify SVIDs from that domain during mTLS.

---

<a id="cross-domain-mtls"></a>

#### Q: What exactly happens during an mTLS handshake when a workload crosses trust domains without federation configured?

`service-A` lives in `global.trust.example` and wants to call `service-B` in `prod.trust.example`. Federation has **not** been set up.

service-A walks the chain to the root and asks: *"Is this root CA in my trust bundle?"*

service-A's trust bundle contains only `global.trust.example root CA`. The chain terminates at an unknown anchor.

**TLS error: `certificate signed by unknown authority`**

The handshake fails. service-A never even sends its own certificate. The failure is symmetric — even if service-A got past step 1, service-B would face the same problem in the other direction.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart LR
    subgraph no ["Without federation"]
        direction LR
        bA1["service-A trust bundle\n─────────────────\nglobal.trust.example root CA"]
        bB1["service-B trust bundle\n─────────────────\nprod.trust.example root CA"]
        bB1 -->|"presents prod.trust.example leaf\n→ root CA not in service-A bundle"| X1["❌ REJECTED"]
        bA1 -->|"presents global.trust.example leaf\n→ root CA not in service-B bundle"| X2["❌ REJECTED"]
    end

    subgraph yes ["With federation"]
        direction LR
        bA2["service-A trust bundle\n─────────────────\nglobal.trust.example root CA\nprod.trust.example root CA ← added"]
        bB2["service-B trust bundle\n─────────────────\nprod.trust.example root CA\nglobal.trust.example root CA ← added"]
        bB2 -->|"presents prod.trust.example leaf\n→ root CA IS in service-A bundle"| Y1["✅ ACCEPTED"]
        bA2 -->|"presents global.trust.example leaf\n→ root CA IS in service-B bundle"| Y2["✅ ACCEPTED"]
    end
</div>

service-A arrives at the border of country Prod. The border officer opens the reference book. Country Global is not in it. service-A is turned away — not because the passport is fake, but because country Prod has never agreed to accept it. Federation is the bilateral agreement that gets country Global added to the book.

---

<a id="federated-x509pop"></a>

#### Q: What is the federated-x509pop-nodeattestor?

X.509 PoP (Proof of Possession) is a node attestation mechanism where the SPIRE agent proves it already holds a valid X.509 certificate from a trusted source.

This is used for nodes that are already part of a federated trust domain — they use their existing X.509 SVID to attest to a new SPIRE server, bootstrapping trust across domains without a separate attestation mechanism.

Using an existing passport from country A to get a visa stamp from country B's consulate, rather than starting the identity verification process from scratch.

---

*Continue reading:*
- *Future deep dives: TPM bootstrap, service mesh, pod SVID lifecycle*
- *Future deep dives: Vault CA, federation internals, Workload API, CSI driver*

---

<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>
  mermaid.initialize({
    startOnLoad: true,
    theme: 'base',
    themeVariables: {
      'darkMode': true,
      'background': '#1c2128',
      'primaryColor': '#2d333b',
      'primaryTextColor': '#adbac7',
      'primaryBorderColor': '#444c56',
      'lineColor': '#444c56',
      'secondaryColor': '#316dca',
      'tertiaryColor': '#1c2128'
    }
  });
</script>
