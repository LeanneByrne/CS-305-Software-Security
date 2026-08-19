# CS 305 Software Security

A Spring Boot REST service hardened against a real security brief: verify data integrity with cryptographic checksums, encrypt data in transit, and assess the application's inherited risk from third party dependencies.

**Stack:** Java, Spring Boot, Maven, OWASP Dependency-Check, Java Keytool

**Portfolio artifact:** `Byrne_CS305_ProjectTwo_Report.docx` (Practices for Secure Software Report)

## Highlights

- SHA-256 checksum endpoint with explicit UTF-8 encoding and zero-padded hex conversion
- HTTPS on port 8443 via a self signed PKCS12 keystore, RSA 2048, SHA384withRSA
- Full dependency vulnerability scan with a documented false positive suppression
- Before and after scan comparison, with the count difference traced to its actual cause

## The Client and Their Requirements

Artemis Financial is a consulting company that builds individual financial plans for its customers, covering savings, retirement, investments, and insurance. Their web application worked, but it had never been evaluated against modern security expectations, and because the company handles sensitive personal and financial data that gap carried real risk.

The company asked for two things. First, a file verification step using a checksum, so that data could be confirmed unaltered after transfer. Second, secure communications between client and server, so that transmitted data could not be read or modified in transit. Both had to be added without breaking functionality customers already relied on.

## Vulnerability Triage: What Went Well

The dependency scan is the part of this project I would point to first, because it required judgment rather than execution. I did not treat the scan report as a verdict. It flags a long list of items, and the easy responses are to panic at the total or decide the tool is noisy and move on. I worked through the findings and decided on each based on whether the vulnerable code path was actually reachable here.

The baseline scan covered 38 dependencies and flagged 11 as vulnerable, carrying 213 vulnerabilities. The post-refactor scan covered the same 38 dependencies and reported 12 vulnerable ones carrying 220. The total went up.

That looks like the refactoring made things worse. It didn't. The two scans ran about 45 minutes apart, and the National Vulnerability Database publishes continuously, so the second scan read a newer picture of the world than the first. New CVEs landed against libraries that had not changed. The useful question is not whether the total moved but whether any finding traces to code I added, and none does. The hash implementation uses only the standard Java security library and contributes nothing to the dependency tree the scanner evaluates.

The scan also flagged CVE-2023-20873 against spring-boot. Reading the CVE rather than the scanner's summary showed it describes a security bypass that only applies to applications deployed to Cloud Foundry, which is not this deployment model. I suppressed it in `suppression.xml` with the justification written into the file, so the reasoning is auditable. A suppressed finding with no explanation is indistinguishable from a mistake to whoever reads it next, and that person might be me a year later.

**Why coding securely matters.** The cost of a flaw climbs steeply the longer it goes unfound. During development it is a code change. After a breach it is a legal problem, a regulatory problem, and a reputation problem at once. For a company like Artemis Financial, whose business rests entirely on customers being willing to hand over their financial details, security is not a feature bolted on at the end. It protects the trust the company needs in order to exist at all.

## What Was Challenging and What Was Helpful

The most useful thing I took from this project was realizing how much of an application's risk I don't write myself. My own code here is a single controller class. The dependency tree behind it is not small, and essentially every vulnerability surfaced lived in a library I pulled in without reading a line of it. The attack surface is not the file I have open, it's everything that ships with it.

The hardest part was tooling rather than concepts. I spent a long stretch fighting persistent 403 errors from the NVD API. I regenerated my API key more than once, assuming the credential was the problem, and it kept failing the same way. The actual cause turned out to be a documented bug in dependency-check plugin versions below 12.2.2, so no amount of fixing my key was ever going to resolve it. Once I moved to 12.2.2 the scan ran clean.

Frustrating at the time, but the lesson stuck. When something fails the same way repeatedly and the obvious explanation keeps not working, the assumption itself is what needs checking. I had been debugging the wrong layer.

## Adding Layers of Security

**Cryptographic hashing.** `ServerController.java` exposes a `/hash` endpoint returning the name, unique data string, cipher algorithm, and SHA-256 checksum. Two details matter more than they look:

The bytes are converted to hex with a two-character format specifier. A naive conversion drops leading zeros on byte values under sixteen, producing a digest that is too short and silently wrong.

The input is encoded with an explicit UTF-8 charset rather than the platform default. A platform-dependent encoding produces different checksums on different machines for identical input, which defeats the entire purpose of a verification step.

SHA-256 was selected over MD5 and SHA-1 because both have practical collision attacks against them. A 256 bit digest puts the birthday bound around 2^128 operations, outside computational feasibility.

**Transport security.** A self signed certificate generated with Java Keytool, stored as a PKCS12 keystore holding an RSA 2048 key pair under the alias `artemis`, signed with SHA384withRSA. PKCS12 over the older JKS format because it's an open standard and has been the Java default since version 9. `application.properties` configures the embedded Tomcat container to serve TLS on 8443.

The layering was deliberate and ordered. Integrity first, then transport, because a checksum sent over an unencrypted channel can be altered alongside the data it protects. Then static analysis, to confirm nothing new was introduced. No single measure is sufficient alone: transport security without integrity verification won't catch a file corrupted before it was ever sent, and dependency scanning without manual review misses logic errors entirely.

**What I would use going forward.** I would keep automated dependency scanning as a permanent baseline rather than a one time step, since new vulnerabilities get published against libraries that never changed. I would pair it with static analysis and the OWASP Top Ten as a manual checklist, because scanners are strong at finding known flaws in known libraries and weak at finding logic errors specific to an application. When deciding what to mitigate first, I would weigh whether the vulnerable code path is actually reachable and what data it touches, rather than sorting purely by severity score.

## Verification and Testing

I confirmed the application compiled and started cleanly and served the endpoint over TLS on 8443, returning the expected name, data string, and checksum. I checked the produced checksum against an independent SHA-256 implementation for the same input to confirm the hex conversion was correct. I verified no sensitive values appear in responses or logs, and that the endpoint accepts no user supplied input, which removes injection as an attack surface here.

To check for newly introduced vulnerabilities, I re-ran the OWASP Dependency-Check scan and compared it against the baseline taken before refactoring, then traced the count difference to its actual cause rather than reading the raw total as a score.

Static analysis finds known flaws in known libraries. It doesn't find logic errors in code written last week, which is why the manual pass mattered alongside it.

## Tools and Practices Worth Carrying Forward

The OWASP Dependency-Check plugin is the clearest takeaway, and it's something I would want configured in any Maven project I work on. Java Keytool, and understanding how keystores, certificates, and TLS configuration fit together, transfers directly to any deployment work. The `MessageDigest` API and the encoding and conversion pitfalls around it apply anywhere hashing shows up.

The practice I most want to keep is documenting the reasoning behind a suppression rather than just suppressing the finding. Writing down why something was dismissed is what makes the decision reviewable later.

## Repository Contents

```
rest-service/
  src/main/java/com/twk/restservice/
    ServerController.java        SHA-256 hashing endpoint
  src/main/resources/
    application.properties       HTTPS and keystore configuration
  pom.xml                        Dependencies and Dependency-Check plugin
  suppression.xml                Documented false positive suppression
Byrne_CS305_ProjectTwo_Report.docx   Practices for Secure Software Report
```

## Running It

The service runs over HTTPS on port 8443. Once started:

```
https://localhost:8443/hash
```

The browser will warn on first access because the certificate is self signed. Expected for a development certificate, and safe to bypass locally. In production this would be replaced by a CA-issued certificate; the configuration is identical, only the chain of trust differs.

## Known Limitations

This project targets Spring Boot 2.2.4 and BouncyCastle 1.46 as provided by the course starter code. BouncyCastle 1.46 dates to 2011 and accounts for a meaningful share of the reported findings. In a real engagement, upgrading both would be the first remediation step, ahead of anything else in this repository. They were left in place here because changing dependency versions mid-project would have invalidated the before and after scan comparison the assessment depends on.

## What This Project Demonstrates

The dependency triage rather than the hashing endpoint. Implementing a checksum to spec is a baseline expectation. Reading a scan report, determining which findings actually apply to the deployment context, documenting that judgment so another engineer can review it, and correctly explaining a metric that moved the wrong way is reasoning about risk instead of reacting to tool output. That is the piece I would put in front of an employer.

## Note on Credentials

The keystore password and NVD API key have been replaced with placeholders. Building this requires supplying your own keystore and requesting an API key from the National Vulnerability Database.
