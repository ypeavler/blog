---
title: "PKI Operations in a Hurry: Let's Encrypt, CSRs & Internal PKI"
categories: Learn in a hurry
author:
- Yuva Peavler
excerpt: "The operational side of PKI - generating CSRs, automating certificates with Let's Encrypt and ACME, running internal CAs, and managing certificate lifecycle at scale."
---

For the cryptographic foundations (public key crypto, certificates, TLS), see [Cryptographic Basics in a Hurry]({{ site.baseurl }}{% post_url 2026-03-05-cryptographic-basics-in-a-hurry %}).
For how PKI is applied to workload identity, SPIFFE, and mTLS, see [Workload Identity in a Hurry]({{ site.baseurl }}{% post_url 2026-03-05-workload-identity-in-a-hurry %}).

---

## Part 1: What is PKI?

---

<a id="what-is-pki"></a>

#### Q: What is PKI?

**PKI** (Public Key Infrastructure) is the system of policies, processes, hardware, software, and standards that governs the creation, distribution, use, storage, and revocation of digital certificates.

It's not a single product or protocol — it's the entire trust infrastructure that makes public key cryptography usable at scale.

PKI answers one fundamental question: **how do you know the public key received actually belongs to the entity it claims to?**

| Component | Role |
|---|---|
| **Certificate Authority (CA)** | Issues and signs certificates; the trusted third party |
| **Registration Authority (RA)** | Verifies identity before the CA issues a cert |
| **Certificate** | The signed document binding a public key to an identity |
| **CRL / OCSP** | Mechanism for marking certificates as no longer valid |
| **Trust store** | Pre-installed set of trusted root CA certificates on a device or OS |
| **Certificate policy** | Documented rules governing how the PKI operates |

---

<a id="why-pki-emerged"></a>

#### Q: Why did PKI emerge — what problem does it solve?

Asymmetric encryption was a breakthrough, but it introduced a new problem: the **man-in-the-middle attack**.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant A as Alice
    participant M as Mallory (attacker)
    participant B as Bob

    A->>M: "Hey Bob, send me your public key"
    Note over M: Intercepts the request
    M->>A: Sends Mallory's public key (claiming it's Bob's)
    A->>M: Encrypts message with Mallory's key
    Note over M: Decrypts, reads, re-encrypts with Bob's real key
    M->>B: Forwards re-encrypted message
    Note over A,B: Neither Alice nor Bob knows Mallory is in the middle
</div>

Alice thinks she's communicating securely with Bob. She is — but with Mallory, not Bob.

PKI solves this by introducing a **trusted third party** (the CA) that vouches for the binding between a public key and an identity. Instead of trusting a key directly, trust is placed in a CA that has already verified the key belongs to who it claims.

---

<a id="role-of-digital-certs"></a>

#### Q: What role do digital certificates play in PKI?

A digital certificate is the core artifact of PKI — it's what the CA produces and what everyone else verifies.

Think of PKI as a digital DMV:
- The **DMV** (CA) verifies identity and issues a credential
- The **driver's license** (certificate) is hard to forge, identifies the holder, and expires
- The **police officer** (verifier) checks the license and trusts it because they trust the DMV issued it

A certificate:
- **Identifies** a person, device, domain, or workload
- **Issued by** a trusted third party (the CA)
- **Tamper-resistant** — the CA's signature covers the entire contents; any modification breaks it
- **Expires** — forces regular renewal and limits exposure if compromised

---

<a id="three-waves-of-pki"></a>

#### Q: What are the three waves of PKI adoption?

**Wave 1 (1995–2002): HTTPS for e-commerce**
A small number of high-value certificates, issued manually, costing thousands of dollars each. The goal: the padlock icon. PKI projects took years and millions of dollars.

**Wave 2 (2003–2010): Enterprise identity**
The mobile workforce arrived. Employees needed VPN access from laptops and phones. Organizations started issuing certificates to devices. Internal CAs emerged. Certificate management became a real operational problem.

**Wave 3 (2011–today): Everything gets a certificate**
IoT devices, cloud workloads, containers, microservices. Millions of short-lived certificates issued and rotated automatically. 98% of organizations say they'd rebuild their PKI if they could — legacy systems from Wave 2 can't keep up. Automation (ACME, SPIFFE/SPIRE, cert-manager) is now mandatory, not optional.

The pattern: each wave multiplied the number of certificates by orders of magnitude and reduced acceptable lifetime. Wave 1: thousands of certs, 1–2 year lifetimes. Wave 3: millions of certs, hours or minutes.

---

## Part 2: PKI Operations

---

<a id="pki-lifecycle"></a>

#### Q: What does operating a PKI actually involve?

PKI operations is the day-to-day work of keeping certificates healthy across their full lifecycle:

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart LR
    A["🔑 Issue\nGenerate keypair\nSubmit CSR\nCA verifies identity\nSigns and returns cert"]
    B["🔄 Renew\nBefore expiry\nSame process as issue\nAutomated where possible"]
    C["❌ Revoke\nIf key is compromised\nPublish to CRL / OCSP"]
    D["📦 Distribute\nGet cert to the right place\n(server, device, workload)"]
    E["📊 Monitor\nTrack expiry dates\nAlert before outages\nAudit usage"]

    A --> B --> A
    A --> C
    A --> D
    D --> E
    E --> B
</div>

The hardest part is not issuing certificates — it's renewing them before they expire, revoking them when keys are compromised, and knowing where all certificates are. Certificate expiry outages are common and entirely avoidable; they happen when the monitoring and renewal process breaks down.

---

<a id="what-is-a-csr"></a>

#### Q: What is a CSR?

A CSR (Certificate Signing Request) is the document sent to a CA when requesting a certificate. It contains:
- The public key
- The identity being claimed (domain name, organization, etc.)
- A signature with the private key (proving ownership of the private key in the request)

The CA verifies the identity claim, then signs the CSR's public key and identity together — producing the certificate.

The private key never leaves the server. The CA never sees it. The CA is vouching for the binding between the public key and the identity, not issuing a keypair.

---

<a id="lets-encrypt"></a>

#### Q: How does Let's Encrypt work?

Let's Encrypt uses the ACME protocol (Automated Certificate Management Environment) to issue certificates without human involvement.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant S as Your Server
    participant LE as Let's Encrypt

    Note over S: Generate keypair + CSR
    S->>LE: Request certificate for yourdomain.com
    LE->>S: Issue challenge: prove you control this domain
    Note over S: HTTP-01: place token at<br/>http://yourdomain.com/.well-known/acme-challenge/TOKEN<br/>— OR —<br/>DNS-01: create TXT record at _acme-challenge.yourdomain.com
    LE->>S: Verify challenge ✓
    LE->>S: Return signed certificate (90-day TTL)
    Note over S: ACME client (certbot, cert-manager, Caddy)<br/>handles renewal automatically
</div>

The 90-day TTL is intentional — it forces automation and limits the blast radius of a compromised cert. Let's Encrypt issues ~5 million certificates per day.

---

<a id="why-cert-manager"></a>

#### Q: Why do organizations need a certificate manager — can't they manage certs manually?

At small scale, manual cert management works. At any real scale, it becomes the leading cause of unplanned outages.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    I["Engineer manually requests cert\nfrom CA portal"]
    S["Cert stored in a spreadsheet,\nwiki page, or someone's head"]
    E["Cert expires\n(nobody got the alert, or the alert went to\na person who left the company)"]
    O["Outage\nTLS handshakes fail, services go down"]
    P["Post-mortem:\n'We need to track our certs better'"]

    I --> S --> E --> O --> P --> I
</div>

This is not hypothetical. Some of the largest outages — and breaches — in recent years were caused by expired certificates:
- **Microsoft Teams (2020)** — an expired authentication cert took Teams offline for hours
- **Ericsson (2021)** — an expired cert in network software caused mobile outages across 11 countries
- **LinkedIn (2023)** — cert expiry caused partial service disruption
- **Equifax (2017)** — an expired cert in their network inspection tool meant malicious traffic went undetected for 76 days, contributing to a breach of 147 million records and a $575M FTC settlement

Research from Keyfactor and the Ponemon Institute found organizations have an average of **88,750 certificates and keys** in use on their networks — impossible to track manually.

| Problem | What a cert manager solves |
|---|---|
| "We don't know what certs we have" | Discovery — scans infrastructure and builds a full inventory |
| "A cert expired and nobody noticed" | Expiry monitoring and alerts with enough lead time to act |
| "Renewing a cert takes a ticket and 3 days" | Automation — renewal happens without human involvement |
| "We need to revoke a cert right now" | Centralized revocation across all issuers |
| "We need to migrate from SHA-1 to SHA-256" | Bulk operations — find and replace all certs matching a policy |

**Short-lived certs make this non-optional.** Let's Encrypt's 90-day TTL was controversial when introduced — now it's the norm. SPIFFE SVIDs have TTLs measured in hours. A cert that expires every 24 hours cannot be managed by a human.

---

<a id="hashicorp-vault"></a>

#### Q: What is HashiCorp Vault and what does it do for PKI?

HashiCorp Vault is a secrets management platform with a built-in PKI engine. It can act as a CA — issuing, signing, and revoking certificates — while also storing and controlling access to other secrets (API keys, database passwords, etc.).

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    R["Vault Root CA\n(or an imported external root)"]
    I["Vault Intermediate CA\n(signed by root, used for day-to-day issuance)"]
    W["Workload / Service\nRequests a cert via Vault API"]
    C["Issued Certificate\n(short-lived, auto-renewed)"]

    R -- "signs" --> I
    W -- "POST /pki/issue/{role}" --> I
    I -- "returns" --> C
</div>

| Capability | What it means |
|---|---|
| **Dynamic certificates** | Certs generated on demand, not stored. Every request gets a fresh cert. |
| **Short TTLs** | Vault encourages 1h–24h cert lifetimes. Short-lived certs are safer than revocation. |
| **Role-based issuance** | A Vault role defines what a cert can contain — allowed domains, max TTL, key type. |
| **Audit log** | Every cert issued is logged — who requested it, when, what it contains. |
| **Root CA offline** | Vault's root CA can be kept offline. The intermediate CA does day-to-day issuance. |

---

<a id="cert-manager-k8s"></a>

#### Q: What is cert-manager and how does it fit into Kubernetes?

cert-manager is a Kubernetes-native certificate manager — a controller that automates the full certificate lifecycle for workloads running in Kubernetes: requesting, renewing, and rotating certificates without human intervention.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart LR
    D["Developer\nDefines a Certificate resource\nin Kubernetes YAML"]
    CM["cert-manager\n(watches Certificate resources)"]
    I["Issuer / ClusterIssuer\n(Let's Encrypt, Vault, internal CA)"]
    S["Kubernetes Secret\n(cert + private key stored here)"]
    W["Workload Pod\nmounts the Secret as a volume"]

    D --> CM
    CM -- "requests cert from" --> I
    I -- "returns signed cert" --> CM
    CM -- "stores in" --> S
    S -- "mounted into" --> W
    CM -- "renews automatically\nbefore expiry" --> I
</div>

Declare what's needed — cert-manager handles the rest:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-service-tls
spec:
  secretName: my-service-tls-secret
  duration: 24h
  renewBefore: 1h
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  dnsNames:
    - my-service.prod.svc.cluster.local
```

| Issuer | Use case |
|---|---|
| **Let's Encrypt (ACME)** | Public-facing services needing browser-trusted certs |
| **Vault** | Internal services using your own CA |
| **Self-signed** | Development / testing |
| **CA (internal)** | Sign with a cert stored directly in a Kubernetes Secret |
| **ACME + DNS-01** | Wildcard certs, private clusters not reachable from the internet |

cert-manager treats certificates as Kubernetes-native resources. Expiry is no longer a manual operational concern — it's a control loop.

---

<a id="three-ways-to-run-pki"></a>

#### Q: What are the three ways to run a PKI?

| Approach | Who operates the CA | Who controls issuance policy | Best for |
|---|---|---|---|
| **Public CA** | DigiCert, Let's Encrypt, etc. | The public CA | Public-facing services needing browser/OS trust |
| **Private CA (DIY)** | Your organization | Your organization | Internal services, workload identity, air-gapped environments |
| **Managed PKI (mPKI)** | A third-party PKI vendor | Shared | Organizations that need internal PKI but lack the in-house expertise |

**Public CA** is the default for anything internet-facing. The CA verifies the domain/org, and the cert is automatically trusted by all browsers and OSes.

**Private CA (DIY)** gives full control — your own root, your own policies, your own issuance rates. This is what Vault + SPIRE provides. The trade-off is operational burden: root CA security, HSMs, intermediate rotation, trust bundle distribution.

**Managed PKI (mPKI / PKI-as-a-Service)** is the middle ground. A vendor (Keyfactor, Venafi, DigiCert ONE, AWS Private CA) runs the PKI infrastructure — HSMs, root CA security, availability — while the organization controls certificate profiles and validation policies.

Only 45% of companies have staff dedicated to managing their PKI (Keyfactor/Ponemon Institute). mPKI exists for the other 55%.

---

<a id="why-internal-pki"></a>

#### Q: Why run an internal PKI?

Public internet PKI (DigiCert, Let's Encrypt) solves the problem of strangers trusting each other — both parties trust DigiCert as a shared third party.

**Internal PKI solves a different problem**: services within an organization trusting each other, without depending on a public CA for every certificate.

**Why public CAs don't fit internal services:**

| Concern | Why public CAs don't fit |
|---|---|
| **Service-to-service identity** | Let's Encrypt issues certs for domain names. Internal services have names like `payments.prod.svc.cluster.local`. |
| **Short TTLs at scale** | Let's Encrypt rate-limits issuance. Thousands of 1-hour certs for workloads would hit those limits immediately. |
| **Identity model** | Public certs bind to DNS names. Internal workload identity needs richer identifiers — like a SPIFFE URI. |
| **Audit and control** | A public CA can't enforce internal policies: only certain services can get certs for certain identities. |
| **Air-gapped environments** | Internal services in private networks can't complete Let's Encrypt's HTTP-01 challenge. |

The mechanics are identical — root CA, intermediate CA, leaf certs, chain verification — but the organization owns and operates the entire chain:

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    subgraph Public["Public Internet PKI"]
        PR["DigiCert Root\n(pre-installed in OS/browser trust store)"]
        PI["DigiCert Intermediate"]
        PL["bank.com TLS cert\n(valid for 90 days)"]
        PR -- "signs" --> PI -- "signs" --> PL
    end

    subgraph Internal["Internal PKI"]
        IR["Your Root CA\n(generated internally, air-gapped)"]
        II["Your Intermediate CA\n(online, does day-to-day issuance)"]
        IL["Service leaf cert\n(valid for 24h or less)"]
        IR -- "signs" --> II -- "signs" --> IL
    end
</div>

**What's needed to run internal PKI:**

| Component | What it does |
|---|---|
| **Root CA** | The trust anchor. Generated once, kept offline. Signs intermediate CAs. |
| **Intermediate CA** | Online CA that does day-to-day cert issuance. Can be rotated without touching the root. |
| **Issuance policy** | Rules for what certs can be issued — allowed identities, max TTL, key type. |
| **Certificate manager** | Automates issuance, renewal, and distribution to services. |
| **Trust bundle distribution** | Mechanism to push the root CA cert to all verifiers. |

In a Kubernetes environment, these tools divide the work:

| Tool | Role | What it issues |
|---|---|---|
| **Vault** | CA + issuance policy | Signs intermediate CAs; issues certs on request |
| **cert-manager** | Certificate lifecycle for Kubernetes resources | Ingress TLS, webhook TLS, internal service TLS |
| **SPIRE** | Workload identity + attestation | SVIDs (X.509 certs with SPIFFE URIs) for mTLS between services |

---

<a id="pki-tiers"></a>

#### Q: What are PKI tiers and how many should you have?

A PKI **tier** is a level in the CA hierarchy. The number of tiers determines how many intermediate CAs sit between the root CA and leaf certificates.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    subgraph T1["1-Tier (avoid)"]
        R1["Root CA\n(online — also the issuing CA)"]
        L1["Leaf certs"]
        R1 -- "signs directly" --> L1
    end

    subgraph T2["2-Tier (standard)"]
        R2["Root CA\n(offline)"]
        I2["Issuing CA\n(online)"]
        L2["Leaf certs"]
        R2 -- "signs" --> I2 -- "signs" --> L2
    end

    subgraph T3["3-Tier (high security)"]
        R3["Root CA\n(air-gapped)"]
        P3["Policy Intermediate\n(HSM, rarely touched)"]
        I3["Issuing CA\n(online, rotated frequently)"]
        L3["Leaf certs"]
        R3 -- "signs" --> P3 -- "signs" --> I3 -- "signs" --> L3
    end
</div>

The more degrees of separation between the root CA and leaf certs, the smaller the blast radius if a CA is compromised. If an issuing CA's key is compromised, only the certs it issued need to be revoked. If the root CA's key is compromised, every cert ever issued from that root must be revoked — a full PKI rebuild.

| Tier | Root CA exposure | When to use |
|---|---|---|
| **1-tier** | Root is online and issuing — maximum exposure | Never in production |
| **2-tier** | Root offline, one online issuing CA | Most organizations |
| **3-tier** | Root air-gapped, policy intermediate in HSM, issuing CA online | High-security environments, public CAs |

Most organizations use a **2-tier** hierarchy. The 3-tier pattern is used by public CAs like DigiCert and Let's Encrypt.

---

<a id="what-is-hsm"></a>

#### Q: What is an HSM and why does PKI use one?

An **HSM** (Hardware Security Module) is a physical device purpose-built for generating, storing, and using cryptographic keys — with the keys never leaving the hardware in plaintext.

For PKI, HSMs solve the root CA private key problem: if the root CA private key is stored on a regular server, anyone who compromises that server gets the key and can issue fraudulent certificates trusted by the entire organization. An HSM makes this impossible — the key is generated inside the hardware, and signing happens inside the hardware. The key cannot be exported.

| Storage | Risk |
|---|---|
| Private key on disk (plaintext) | Stolen by anyone with filesystem access |
| Private key encrypted on disk | Stolen if the decryption key is also compromised |
| Private key in an HSM | Cannot be extracted — signing happens inside the device |

**How HSMs are used in PKI:**
- **Root CA**: Key stored in an offline HSM, physically locked away. Brought online only to sign a new intermediate CA (once a year or less).
- **Intermediate CA**: Key stored in an online HSM (e.g. AWS CloudHSM, Azure Dedicated HSM) for day-to-day cert signing.
- **Vault**: HashiCorp Vault can be configured to use an HSM as its seal and key storage backend.

The root CA key should always be in an HSM. The intermediate CA key should be in an HSM if the security stakes justify the cost.

---

<a id="trust-anchor"></a>

#### Q: What is a trust anchor?

A **trust anchor** is the certificate trusted unconditionally — without needing to verify it against anything else. It's where chain verification stops.

In practice, the trust anchor is always a **root CA certificate**. When a verifier walks a certificate chain, it keeps checking signatures upward until it reaches a cert already in its trust store.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    R["Root CA cert\n✓ In trust store\n→ Trust anchor: accepted unconditionally"]
    I["Intermediate CA cert\n✓ Signature valid, signed by Root CA"]
    L["Leaf cert (e.g. api.myapp.com)\n✓ Signature valid, signed by Intermediate CA"]

    L -- "verified by" --> I -- "verified by" --> R
    R -- "trusted because\nit's in the trust store" --> T["✓ Chain valid"]
</div>

**Trust anchors are not verified — they are chosen.** There is no verification of a root CA's certificate against anything; it's a deliberate decision to include it in the trust store. This is why:
- Browsers ship with ~150 pre-trusted root CAs (Mozilla, Apple, Microsoft each maintain their own list)
- An OS or browser update can add or remove a root CA — removing one immediately invalidates every cert that chains to it

| Term | Meaning |
|---|---|
| **Trust anchor** | A single root CA cert that has been decided to trust |
| **Trust store** | The collection of all trust anchors on a system |
| **Trust bundle** | The SPIFFE/SPIRE term for the set of root CA certs distributed to workloads |

**Internal PKI trust anchor distribution** is the operational challenge that public PKI solves automatically (via OS/browser updates). When running an internal CA, the root CA cert must be distributed to every service that needs to verify internal certs. If a service doesn't have the root CA in its trust store, it will reject every cert the internal PKI issues.

---

## Part 3: Software Supply Chain Signing

PKI isn't only for securing network connections. The same signing primitives — private key signs, public key verifies — are used to prove that software artifacts (binaries, container images, Git commits, SBOMs) came from a known source and haven't been tampered with.

The most damaging attacks in recent years haven't broken encryption — they've poisoned the software supply chain:
- **SolarWinds (2020)** — attackers compromised the build pipeline and inserted malware into signed software updates. The update was signed with SolarWinds' legitimate cert, so nothing flagged it.
- **XZ Utils (2024)** — a malicious contributor spent two years gaining trust, then inserted a backdoor into a widely-used compression library just before release.
- **Codecov (2021)** — attackers modified a CI script to exfiltrate environment variables (including secrets) from thousands of CI pipelines.

Signing artifacts doesn't prevent all of these — SolarWinds' signing key was legitimate — but it creates an auditable chain of custody and makes tampering detectable after the signing step.

---

<a id="code-signing"></a>

#### Q: What is code signing and how does it work?

Code signing uses the same asymmetric signing mechanism as TLS certificates — a private key signs a hash of the artifact, and anyone with the corresponding public key (embedded in a certificate) can verify it.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
sequenceDiagram
    participant D as Developer / CI Pipeline
    participant CA as Code Signing CA
    participant U as End User / Verifier

    D->>CA: Request code signing certificate
    CA->>D: Issue cert (binds public key to publisher identity)
    Note over D: Hash the artifact (binary, image, etc.)
    Note over D: Sign the hash with private key
    D->>U: Distribute artifact + signature + certificate
    Note over U: Verify signature using cert's public key
    Note over U: Check cert chains to trusted CA
    Note over U: ✓ Artifact is from this publisher and unmodified
</div>

| Artifact | What signing proves |
|---|---|
| Windows/macOS binary | Came from a known publisher; OS shows "verified publisher" |
| Container image | Image digest matches what was built and pushed by the CI pipeline |
| Git commit | Commit was authored by the holder of a specific GPG/SSH key |
| SBOM | The software bill of materials accurately reflects what was built |
| Helm chart | Chart contents haven't been modified since the publisher signed it |
| npm / PyPI package | Package came from the registered maintainer |

---

<a id="sigstore"></a>

#### Q: What is Sigstore and why does it matter?

Traditional code signing has a key management problem: every developer and CI pipeline needs a long-lived signing key, those keys need to be stored securely, and if one leaks everything signed with it must be revoked.

**Sigstore** solves this with **keyless signing** — no long-lived keys, no key management, no revocation problem.

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    CI["CI Pipeline\n(GitHub Actions, etc.)"]
    Fulcio["Fulcio\n(Sigstore's CA)"]
    Rekor["Rekor\n(Sigstore's transparency log)"]
    Cert["Short-lived cert\n(valid for ~10 minutes)"]
    Sig["Signature\n(stored in OCI registry or Rekor)"]

    CI -- "presents OIDC token\n(proves: I am github.com/org/repo, workflow X)" --> Fulcio
    Fulcio -- "issues" --> Cert
    CI -- "signs artifact with ephemeral key" --> Sig
    Sig -- "logged in" --> Rekor
    Cert -- "embedded in" --> Sig
</div>

1. The CI pipeline authenticates to Fulcio using an OIDC token — proving "I am this specific workflow in this specific repo"
2. Fulcio issues a **short-lived certificate** (~10 minutes) binding the ephemeral public key to that OIDC identity
3. The pipeline signs the artifact; the signature + cert are logged in **Rekor** (a tamper-evident transparency log)
4. The ephemeral key is discarded — nothing to steal or revoke

**Cosign** is the CLI tool for signing and verifying container images using Sigstore. **Notation** (CNCF) is the equivalent for OCI artifacts more broadly.

---

<a id="image-signing"></a>

#### Q: What is container image signing and why should it be enforced?

A container image tag (`myapp:latest`) is mutable — it can be overwritten to point to a different image at any time. An image digest (`myapp@sha256:abc123...`) is immutable — it's the SHA-256 hash of the image manifest. Signing the digest makes tampering detectable.

**Without signing — the attack:**

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart LR
    B["CI builds\nmyapp:v1.2.3\n(legitimate)"]
    R["Registry\nmyapp:v1.2.3"]
    A["Attacker overwrites\nmyapp:v1.2.3\n(malicious image)"]
    K["Kubernetes pulls\nmyapp:v1.2.3\n← gets malicious image"]

    B --> R
    A --> R
    R --> K
</div>

**With image signing + admission control:**

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart LR
    B["CI builds + signs\nmyapp:v1.2.3"]
    R["Registry"]
    K["Kubernetes admission webhook\nverifies signature before\nallowing pod to start"]
    D["Deploy ✓ or reject ✗"]

    B --> R --> K --> D
</div>

| Tool | What it does |
|---|---|
| **Cosign** (Sigstore) | Signs and verifies container images; supports keyless via Sigstore |
| **Notation** (CNCF) | Signs OCI artifacts; integrates with Azure Container Registry, AWS ECR |
| **Kyverno / OPA Gatekeeper** | Kubernetes admission controllers that enforce "only signed images allowed" |
| **Connaisseur / Ratify** | Dedicated admission webhooks for image signature verification |

---

<a id="code-signing-risks"></a>

#### Q: What are the security risks of code signing — isn't signing enough?

Signing proves that an artifact was signed by the holder of a private key. It does **not** prove that the private key itself is secure. The entire trust model collapses if the signing key is compromised.

**The three ways code signing breaks:**

1. **Key theft** — an attacker steals the private signing key and uses it to sign malicious software. This is exactly what happened in SolarWinds: the attackers didn't break the signing algorithm, they compromised the build environment and used the legitimate key.

2. **Open developer access** — if a developer's workstation has direct access to the signing key, an attacker who compromises that workstation can submit anything for signing. The signature will be valid.

3. **Signing the wrong thing** — malicious code can be injected at any point before the signing step: in a dependency, in a build script, in a compromised base image. The signature only proves the artifact was signed — not that what was signed is safe.

| Risk | Mitigation |
|---|---|
| Signing key on developer workstation | Keys in HSM or centralized signing service — developers never touch the key |
| CI pipeline has direct key access | Remote signing: pipeline sends artifact hash to signing service, gets signature back |
| No audit trail | Every signing operation logged: who triggered it, what was signed, when |
| Keys never rotated | Signing certs have defined TTLs; rotation is automated |
| Anyone can sign anything | Role-based policy: only specific pipelines/identities can sign specific artifact types |

Sigstore's keyless model sidesteps key management for open-source and CI workflows. For enterprise environments with strict audit requirements, a centralized signing service with HSM-backed keys is the standard.

---

<a id="supply-chain-and-pki"></a>

#### Q: How does supply chain signing relate to PKI?

Supply chain signing *is* PKI — the same root CA → intermediate CA → leaf cert chain, just applied to software artifacts instead of network connections.

| TLS use case | Supply chain equivalent |
|---|---|
| Server presents cert to prove identity | Binary ships with cert to prove publisher identity |
| Client verifies cert chains to trusted CA | OS / verifier checks cert chains to trusted CA |
| Short-lived leaf cert, auto-renewed | Short-lived signing cert (Sigstore: ~10 min) |
| Revocation via CRL/OCSP | Revocation via transparency log (Rekor) |
| Trust anchor in OS trust store | Trust anchor in Sigstore root or your internal CA |

TLS certs are verified at connection time (online, real-time). Artifact signatures are verified at deployment time — and the signature is a permanent record that can be audited later.

This is why supply chain signing shows up in CBOM discussions: the signing keys and certificates used in CI pipelines are cryptographic assets that need to be inventoried, monitored, and rotated just like TLS certs.

---

## Part 4: Cryptographic Posture

---

<a id="what-is-cbom"></a>

#### Q: What is a CBOM?

A **CBOM** (Cryptography Bill of Materials) is a complete inventory of all cryptographic assets in a system — every algorithm, key, certificate, and library, along with where each is used and what its current status is.

It's the cryptographic equivalent of an SBOM (Software Bill of Materials). Just as an SBOM tells you which software dependencies are running and whether any have known vulnerabilities, a CBOM tells you which cryptographic primitives are running and whether any are weak, expired, or quantum-vulnerable.

| Asset | Algorithm | Key size | Location | Status |
|---|---|---|---|---|
| TLS cert for api.myapp.com | RSA | 2048 | nginx, namespace prod | Expires in 14 days ⚠️ |
| JWT signing key | ECDSA P-256 | 256 | auth-service secret | OK ✓ |
| Database connection | TLS 1.2, AES-128-GCM | — | db-client config | TLS 1.2 — upgrade needed ⚠️ |
| SSH host key | RSA | 1024 | bastion host | **Broken — rotate now** ❌ |

---

<a id="cbom-vs-cert-monitoring"></a>

#### Q: Why does a CBOM matter — isn't certificate monitoring enough?

Certificate monitoring (expiry alerts, chain validation) only covers one slice of cryptographic posture. A CBOM covers the full picture:

| What certificate monitoring catches | What only a CBOM catches |
|---|---|
| Expired certificates | Weak key sizes (RSA-1024, SHA-1) |
| Invalid certificate chains | Deprecated TLS versions (1.0, 1.1) |
| Mismatched domain names | Broken cipher suites (RC4, 3DES) |
| — | Hardcoded keys in application code |
| — | Cryptographic libraries with known CVEs |
| — | Algorithms vulnerable to quantum computers |

**Post-quantum cryptography (PQC)** is not a future concern — NIST finalized the first post-quantum standards in 2024 (ML-KEM for key exchange, ML-DSA for signatures). RSA and ECC are broken by a sufficiently powerful quantum computer. Organizations that don't know where their RSA-2048 and ECC keys are cannot migrate them.

A CBOM is the prerequisite for **crypto-agility** — the ability to swap out a cryptographic algorithm across the entire infrastructure when it's deprecated or broken, without a scramble.

---

<a id="build-cbom"></a>

#### Q: How do you build and maintain a CBOM?

<div class="mermaid">
%%{init: {'theme': 'dark'}}%%
flowchart TD
    D["Discover\nScan infrastructure for crypto assets:\ncerts, keys, TLS configs, cipher suites,\ncrypto library versions"]
    I["Inventory\nRecord algorithm, key size, location,\nexpiry, issuer, usage context"]
    A["Assess\nFlag weak algorithms, short expiries,\ndeprecated versions, quantum-vulnerable keys"]
    R["Remediate\nRotate weak keys, upgrade TLS versions,\nreplace broken cipher suites"]
    M["Monitor\nContinuous scanning — new services\nappear, certs expire, CVEs are published"]

    D --> I --> A --> R --> M --> D
</div>

| Source | What it reveals |
|---|---|
| Certificate transparency logs | All publicly-issued certs for your domains |
| Kubernetes secret scanning | Certs and keys stored in cluster secrets |
| Network scanning (TLS probes) | TLS versions and cipher suites in use |
| Code scanning | Hardcoded keys, weak algorithm calls in source |
| Vault / cert-manager audit logs | Centrally-issued cert inventory |
| SPIFFE trust bundle | All workload SVIDs and their properties |

SPIFFE/SPIRE deployments have a natural CBOM advantage: all workload certs are issued centrally through SPIRE, so the inventory is already known. The hardest part is finding crypto that is *not* centrally managed — hardcoded keys in application code, old SSH host keys, TLS configs in legacy services that predate the PKI.

---

*More questions coming as we explore. Ask away.*

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
