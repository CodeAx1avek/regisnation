# Scenario-Based Interview Questions — Jio Credit Security Interview Prep

*Compiled from our discussion — organized by topic with expected depth*

---

## 🎯 **1. Network Service Enumeration & Attack (Nginx)**

### Scenario
> "During an external pentest, you discover an Nginx web server on port 443. Walk me through your complete approach — from initial discovery to post-exploitation."

### Follow-up Probes
| Question | What They're Testing |
|----------|---------------------|
| "How do you fingerprint the version without triggering WAF?" | Passive vs active recon tradecraft |
| "What config misconfigurations do you check first?" | Alias traversal, unsafe proxy_pass, nginx_status exposure |
| "You find a path traversal via `alias` misconfig. How do you escalate?" | Chaining: config read → upstream discovery → SSRF/pivot |
| "The server hosts multiple vhosts. How do you enumerate them?" | Host header fuzzing, SNI, certificate transparency logs |
| "Nginx is fronting a Java app. How does that change your approach?" | Backend-specific attacks (deserialization, JNDI) via Nginx headers |

### Expected Methodology Keywords
Passive → Active → Version-specific CVEs → Config audit → Chaining → Pivot → Reporting

---

## ☁️ **2. Cloud Security — Customer Form → S3 Pipeline**

### Scenario
> "A customer-facing web form collects PII documents (PDF, images) and stores them directly in S3. Design the security controls end-to-end."

### Follow-up Probes
| Question | What They're Testing |
|----------|---------------------|
| "How do you prevent a malicious user from uploading a polyglot file that bypasses validation?" | Magic bytes, AV scanning, MIME confusion, re-encoding |
| "The business wants customers to download their files later. How do you handle that securely?" | Presigned GET URLs, prefix isolation, short TTL, audit logging |
| "How do you detect if an insider or compromised key accesses all customer data?" | CloudTrail data events, Macie, GuardDuty, cross-account log replication |
| "Regulatory requirement: data must be immutable for 7 years. What do you configure?" | Object Lock (WORM), Compliance mode, cross-region replication to audit account |
| "The form backend gets compromised. How do you limit blast radius?" | Prefix-scoped IAM roles, presigned URL scope, KMS key per tenant, VPC endpoints |

### Control Layers Expected
Transport → At-rest (SSE-KMS/CMK) → Access (prefix + presigned + SCPs) → Validation (AV + magic bytes) → Network (VPCe + WAF) → Monitoring (CloudTrail + GuardDuty + Macie) → Lifecycle → IR Ready

---

## 🛡️ **3. RASP Bypass — Server-Side (Java/.NET/Node)**

### Scenario
> "The target application has a commercial RASP agent (e.g., Contrast, Fortify, Datadog). You have RCE via deserialization but the RASP blocks `Runtime.exec`. How do you proceed?"

### Follow-up Probes
| Question | What They're Testing |
|----------|---------------------|
| "Name three hook evasion techniques that work across Java/.NET/Node." | Reflection, native code (JNI/FFI), script engines, deserialization gadgets |
| "The RASP inspects `ProcessBuilder` arguments. How do you bypass argument inspection?" | String concatenation at runtime, environment variable indirection, LOLBins |
| "You suspect the RASP has a blind spot in async execution. How do you test?" | `CompletableFuture`, `Task.Run`, Node `setImmediate` — offload to thread pool |
| "Can you disable the RASP agent itself?" | Agent attachment order, bytecode retransformation, JVMTI/CLR Profiling API abuse |
| "What's your approach if you *don't* know which RASP vendor?" | Fingerprint via error messages, hook signatures, telemetry domains, class names |

### Key Categories Expected
Hook evasion → Timing/race → Policy logic abuse → RASP-specific (agent order, bytecode) → Language-specific

---

## 📱 **4. Mobile RASP Bypass — Android & iOS**

### Scenario A (Android)
> "An Android banking app uses a mobile RASP (e.g., Promon, Appdome, Guardsquare). You need to hook a crypto function. The app detects Frida and crashes. Walk me through your bypass chain."

### Scenario B (iOS)
> "An iOS app implements jailbreak detection, certificate pinning, and a Swift-based RASP. You have a non-jailbroken device. How do you analyze the app?"

### Follow-up Probes (Cross-Platform)
| Question | What They're Testing |
|----------|---------------------|
| "How do you bypass root/jailbreak detection without triggering RASP?" | Magisk Zygisk / KernelSU (Android), FlyJB / Shadow / KFD (iOS) — hide artifacts at kernel/syscall level |
| "The RASP validates its own integrity via checksum. How do you hook anyway?" | Frida early hook (`Java.perform` / `constructor` swizzle) before checksum runs; or patch the checksum function |
| "Telemetry goes to `rasp.vendor.com` with cert pinning. How do you silence it?" | `/etc/hosts` (root), NetworkExtension (iOS), Frida `SSL_CTX_set_verify` hook, replay "clean" response |
| "App uses Play Integrity / App Attest. How do you spoof a 'genuine' verdict?" | PlayIntegrityFix (Magisk), TrickyStore (Android); `attres` bypass / `mis_agent` exploit (iOS) |
| "You want to persist your hooks across app updates. What do you do?" | Repackage `.apk`/`.ipa` with Frida gadget embedded + re-sign (TrollStore / AltStore / objection) |

### Android-Specific Keywords
Frida/Xposed/LSPosed → Magisk/Zygisk/KernelSU → VirtualXposed/VMOS → Native injection → Play Integrity bypass → Repackaging

### iOS-Specific Keywords
Frida gadget + re-sign → ElleKit/Theos tweaks → FlyJB/Shadow/KFD → `DYLD_INSERT_LIBRARIES` (legacy) → TrollStore/SideStore → Keychain extraction

---

## 🏗️ **5. IaaS + IaC Security**

### Scenario
> "Your organization runs 50+ AWS accounts. Developers deploy via Terraform. You're designing the security guardrails. Describe the complete pipeline from code commit to runtime drift detection."

### Follow-up Probes
| Question | What They're Testing |
|----------|---------------------|
| "A developer commits an S3 bucket with `public_read = true`. Where does your pipeline catch it?" | Pre-commit (`tfsec`), PR gate (OPA/Rego), CI (`checkov`), plan approval (human) |
| "How do you prevent the CI/CD pipeline itself from being compromised?" | Ephemeral runners, OIDC → IAM Role (no static keys), least-privilege `iam:PassRole`, signed providers |
| "State file contains secrets. How do you protect it?" | S3 backend + SSE-KMS + versioning, DynamoDB locking, no local state, separate state per env |
| "An admin makes a manual console change (SG rule). How do you detect and remediate?" | Scheduled `terraform plan -refresh-only`, `driftctl`, AWS Config Rules → EventBridge → Lambda auto-remediate |
| "How do you enforce multi-account guardrails (e.g., no CloudTrail disabling)?" | SCPs at Organization level, Control Tower / custom landing zone, `AWSControlTower` baseline |

### Expected Pipeline Stages
IDE → Pre-commit → PR (Policy-as-Code) → CI (Scan + Test) → Plan Review (2 approvals) → Apply (OIDC + Locked State) → Drift Detection → Feedback Loop

---

## 🔐 **6. Android App Testing with ProGuard/R8**

### Scenario
> "You're given a release APK of a fintech app. `proguard-rules.pro` shows heavy obfuscation (`-obfuscation`, `-optimizationpasses 5`). No mapping file provided. How do you test the authentication and payment flows?"

### Follow-up Probes
| Question | What They're Testing |
|----------|---------------------|
| "What artifacts *survive* ProGuard that you can anchor on?" | Strings (API URLs, keys), Manifest component names, Framework APIs (`javax.crypto`), Reflection strings, Native `.so` files |
| "How do you find the obfuscated class handling login?" | `grep -r "api/auth" smali/` → trace callers → label; or Frida `enumerateLoadedClasses()` + filter by package |
| "Frida hooks keep breaking due to R8 inlining. What do you do?" | `Java.deoptimizeEverything()`, hook at higher level (interface/callback), use `Java.choose()` for instances |
| "The app uses DexGuard (string encryption + class encryption). Now what?" | Dynamic dump (`dex2jar` from memory via Frida), hook `ClassLoader.defineClass`, target native layer |
| "You need to test certificate pinning bypass. ProGuard renamed `okhttp3.CertificatePinner`. How?" | Hook `javax.net.ssl.X509TrustManager` (framework, not obfuscated) or `NetworkSecurityConfig` parser |

### Expected Approach
1. Check for mapping.txt (CI artifacts, dev builds) → if yes: JADX + mapping = done
2. Black-box: String/manifest/framework anchors → grep smali → dynamic Frida hooks by runtime name
3. Label only critical flows (auth, crypto, payments) — don't deobfuscate entire app
4. Native libs → Ghidra/IDA (ProGuard doesn't touch NDK)
5. Network MITM → API contracts visible regardless of obfuscation

---

## 📋 **Quick "Name 3" Cheatsheets by Topic**

| Topic | 3 Things to Say in 30 Seconds |
|-------|-------------------------------|
| **Nginx Enum** | 1. Passive (headers, TLS, CT logs) 2. Active (vhost fuzz, config paths, `nginx_status`) 3. Config misconfigs (alias, proxy_pass, CORS) → chain to backend |
| **S3 Form Pipeline** | 1. Presigned PUT (prefix-scoped, short TTL, SSE-KMS) 2. Server-side validation + AV scan 3. CloudTrail data events + Macie + GuardDuty + cross-account logs |
| **RASP Bypass** | 1. Hook evasion (reflection, JNI, script engines, deserialization) 2. Policy abuse (string concat, env vars, LOLBins) 3. Agent tampering (attachment order, bytecode retransformation) |
| **Mobile RASP** | 1. Kernel-level hide (Magisk Zygisk / KFD) 2. Early Frida hook before integrity check 3. Repackage with gadget + re-sign for persistence |
| **IaaS + IaC** | 1. Pipeline gates: pre-commit → PR (OPA) → CI → Plan approval → Apply (OIDC) 2. State: S3+KMS+DynamoDB lock, no local 3. Drift: scheduled refresh + driftctl + Config Rules → auto-remediate |
| **ProGuard Testing** | 1. Mapping.txt? → JADX = vanilla review 2. No mapping? Anchors: strings, manifest, framework APIs → grep smali 3. Dynamic first: Frida by runtime name, MITM for APIs |

---

## 🎤 **Interview Soundbites (60-90 sec each)**

### Nginx
> "Passive recon first — headers, TLS, CT logs. Then active: vhost fuzzing, config disclosure paths (`nginx_status`, `.git`, backups). Version → CVE mapping. Config audit: alias traversal, unsafe `proxy_pass`, open redirects. If RCE: pivot via upstream to internal services. Report with business impact, not just 'found Nginx 1.18'."

### S3 Form
> "TLS everywhere. Presigned PUT URLs scoped to `uploads/{userId}/` with 5-min TTL, forced SSE-KMS via bucket policy. Server-side: magic-byte validation, ClamAV/GuardDuty scan, strip metadata. VPC endpoints only, WAF rate limiting. CloudTrail data events + Macie + GuardDuty → central log account. Lifecycle: IA → Glacier → expire. Cross-region replication to audit account for immutability."

### RASP (Server)
> "RASP raises the bar but has blind spots: JNI/native, script engines (Nashorn, SpEL), deserialization gadgets, classloader isolation. I'd fingerprint the vendor first, then target their specific hook gaps — reflection for Java, `DynamicMethod` for .NET, `vm.runInNewContext` for Node. Always assume egress filtering catches post-exploitation."

### Mobile RASP
> "Attacker owns the hardware. Step 1: break attestation (Play Integrity / App Attest) via Magisk Zygisk / KFD. Step 2: early Frida hook before RASP initializes — or patch the integrity check. Step 3: kill telemetry (MITM or block egress). Step 4: repackage with embedded gadget for persistence. RASP + MDM + hardware keystore raises cost — goal is 'too expensive for threat model.'"

### IaaS + IaC
> "IaaS = shared responsibility. I own config, data, identity, network above hypervisor. Controls: default-deny SGs, IMDSv2, SSE-KMS/CMK, IAM roles (no keys), GuardDuty+Security Hub+Config. IaC = how I enforce it. Pipeline: pre-commit `tfsec`, PR gate with OPA (no 0.0.0.0/0, mandatory tags), plan posted for 2 approvals, ephemeral OIDC runners, locked/encrypted state. Drift detected via scheduled refresh + driftctl + Config Rules → auto-remediate. Runtime findings feed back into policy — closed loop."

### ProGuard
> "ProGuard is obfuscation, not encryption. Strings, manifest, framework APIs, native code survive. If mapping.txt exists (CI artifact), JADX + mapping = standard review. Black-box: anchor on strings (API endpoints, prefs keys), grep smali for `javax/crypto`, dynamic Frida hooks via `enumerateLoadedClasses()`, MITM for API contracts. Label only critical flows — don't deobfuscate everything. Native libs → Ghidra."

---

## 🧠 **Prep Strategy**

| Phase | Action |
|-------|--------|
| **Week 1** | Pick 2 topics → write out full answers by hand → time yourself (90 sec each) |
| **Week 2** | Mock interview with peer/mentor → record → critique: jargon, structure, "so what?" |
| **Week 3** | Red-team each answer: "How would I break this?" → add caveats/limitations to responses |
| **Day Before** | Review cheatsheets only. Sleep. |

---

*Good luck — you know the tech. Now own the narrative.*


🎯 API Security
#	Scenario
1	You’re reviewing a fintech REST API. GET /api/v1/transactions/{id} only checks user.isAuthenticated() — no ownership validation. Walk me through: how you confirm exploitability, demonstrate impact to the client, and remediate (immediate + systemic).
2	A developer says: “We use JWT, so it’s secure.” You decode a token and see "alg": "none" and "jku": "https://attacker.com/jwks.json". Explain the vulnerabilities and how you’d fix the validation logic.
3	GraphQL endpoint allows batched queries with arbitrary depth. A single request can enumerate the entire schema and cause DoS. How do you secure it?
4	Two applications (Admin Portal + Mobile App) share the same backend API. How do you test for cross-tenant data leakage, token scope issues, and shared rate-limiting abuse?
📱 Mobile Security (Android/iOS)
#	Scenario
5	APK decompile reveals: API key in strings.xml, session tokens in plaintext SharedPreferences, allowBackup="true", no cert pinning. Dev says: “ProGuard + Play Store = secure.” Your response and findings report structure.
6	An exported Activity (android:exported="true") accepts arbitrary Intent extras and performs a sensitive action. How does a malicious app exploit this? Mitigation?
7	Implicit Intent used for internal navigation. Malicious app registers same action with higher priority. Attack flow and fix?
8	iOS app has NSAppTransportSecurity allowing arbitrary loads + no certificate pinning. You intercept traffic with Burp + custom CA. Business risk and remediation.
🏗️ Architecture Review / Threat Modeling
#	Scenario
9	Team presents: API Gateway → Order Service → Payment Service → Notification Service (async Kafka). Shared PostgreSQL (separate schemas). Secrets in K8s ConfigMaps. No mTLS, no service mesh. You have 30 min. Top 5 questions + critical risks.
10	Third-party SaaS integration requests admin scope for a read-only webhook. Vendor provides SOC2 but no pen test. Your vendor assessment process.
11	Event-driven system: services emit unsigned Kafka events. How do you prevent event injection, replay, and spoofing?
🔍 Secure Code Review / DevSecOps
#	Scenario
12	PR: Spring Boot file upload uses file.getOriginalFilename() directly in Files.copy(). List every issue, prioritize top 3, write the PR comment.
13	SAST pipeline reports 500 findings. How do you triage, suppress false positives, and create a sustainable fix cadence?
14	CI detects hardcoded AWS keys in a merged commit. Immediate actions + systemic fix.
15	Java deserialization in legacy code: ObjectInputStream.readObject() on user-supplied byte array. Exploit chain and fix.
🌐 Web App Security
#	Scenario
16	Search endpoint reflects input inside <h1>Hello</h1>. Payloads that work here? Escalation from alert(1) to account takeover?
17	Hidden field price=100 in checkout form. Tester changes to price=1. Dev says “frontend validation exists.” Your response.
18	PDF generation: user-controlled name rendered in PDF template. Vectors: XSS via PDF.js, LaTeX injection, XXE in parser. Mitigations.
19	Blind SSRF in webhook URL parameter. No outbound response. Escalation path to cloud metadata, internal port scan, RCE.
20	Request smuggling suspected on nginx → Java backend. CL.TE vs TE.CL detection and mitigation.
21	CRLF injection in Location header via redirect?url=user_input. Cache poisoning + response splitting demo.
🔐 Crypto / Identity
#	Scenario
22	JWT uses "jku": "https://attacker.com/jwks.json" and "kid": "test-key". Explain JKU/JKID injection and key confusion. Secure validator design.
23	Certificate Authority compromise (like DigiNotar). How does CA trust chain work? What fails when root is compromised?
23	Bind shell vs reverse shell in post-exploitation. Real-world chain: RCE → shell → persistence → lateral movement → cloud metadata.
🚨 Incident Response / Judgment
#	Scenario
24	Friday 4:45 PM: Critical RCE (CVE) in transitive dependency of public auth service. Release in 2 hours. Walk me through your 60-minute decision process and stakeholder communication.
25	Critical vuln found in production Saturday 2 AM. No on-call dev. Your playbook.
