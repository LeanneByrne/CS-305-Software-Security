# CS 305 Software Security

A Spring Boot REST service hardened against a real security brief: verify data integrity with cryptographic checksums, encrypt data in transit, and assess the application's inherited risk from third party dependencies. Built for Artemis Financial, a fictional financial planning firm, as a term-long project in a single continuous codebase.

**Stack:** Java, Spring Boot, Maven, OWASP Dependency-Check, Java Keytool

## Highlights

- SHA-256 checksum endpoint with explicit UTF-8 encoding and zero-padded hex conversion
- HTTPS on port 8443 via a self signed PKCS12 keystore, RSA 2048, SHA384withRSA
- Full dependency vulnerability scan with a documented false positive suppression
- Before and after scan comparison, with the count difference traced to its actual cause

## Vulnerability Triage

The dependency scan is the part of this project I'd point to first, because it required judgment rather than execution.

The baseline scan covered 38 dependencies and flagged 11 as vulnerable, carrying 213 vulnerabilities. The post-refactor scan covered the same 38 dependencies and reported 12 vulnerable ones carrying 220. The total went up.

That looks like the refactoring made things worse. It didn't. The two scans ran about 45 minutes apart, and the National Vulnerability Database publishes continuously, so the second scan read a newer picture of the world than the first. New CVEs landed against libraries that had not changed. The useful question is not whether the total moved but whether any finding traces to code I added, and none does. The hash implementation uses only the standard Java security library and contributes nothing to the dependency tree the scanner evaluates.

The scan also flagged CVE-2023-20873 against spring-boot. Reading the CVE rather than the scanner's summary showed it describes a security bypass that only applies to applications deployed to Cloud Foundry, which is not this deployment model. I suppressed it in `suppression.xml` with the justification written into the file, so the reasoning is auditable. A suppressed finding with no explanation is indistinguishable from a mistake to whoever reads it next, and that person might be me a year later.

The broader thing this project taught me is how much of an application's risk I don't write myself. My own code here is a single controller class. The dependency tree behind it is not small, and essentially every vulnerability surfaced lived in a library I pulled in without reading a line of it. The attack surface is not the file I have open, it's everything that ships with it.

## Implementation

**Cryptographic hashing.** `ServerController.java` exposes a `/hash` endpoint returning the name, unique data string, cipher algorithm, and SHA-256 checksum. Two details matter more than they look:

The bytes are converted to hex with a two-character format specifier. A naive conversion drops leading zeros on byte values under sixteen, producing a digest that is too short and silently wrong.

The input is encoded with an explicit UTF-8 charset rather than the platform default. A platform-dependent encoding produces different checksums on different machines for identical input, which defeats the entire purpose of a verification step.

SHA-256 was selected over MD5 and SHA-1 because both have practical collision attacks against them. A 256 bit digest puts the birthday bound around 2^128 operations, outside computational feasibility.

**Transport security.** A self signed certificate generated with Java Keytool, stored as a PKCS12 keystore holding an RSA 2048 key pair under the alias `artemis`, signed with SHA384withRSA. PKCS12 over the older JKS format because it's an open standard and has been the Java default since version 9. `application.properties` configures the embedded Tomcat container to serve TLS on 8443.

The layering was deliberate. Integrity first, then transport, because a checksum sent over an unencrypted channel can be altered alongside the data it protects. Neither measure is sufficient alone: transport security without integrity verification won't catch a file corrupted before it was ever sent.

## Repository Contents

```
rest-service/
  src/main/java/com/twk/restservice/
    ServerController.java        SHA-256 hashing endpoint
  src/main/resources/
    application.properties       HTTPS and keystore configuration
  pom.xml                        Dependencies and Dependency-Check plugin
  suppression.xml                Documented false positive suppression
```

## Running It

The service runs over HTTPS on port 8443. Once started:

```
https://localhost:8443/hash
```

The browser will warn on first access because the certificate is self signed. Expected for a development certificate, and safe to bypass locally. In production this would be replaced by a CA-issued certificate; the configuration is identical, only the chain of trust differs.

## Known Limitations

This project targets Spring Boot 2.2.4 and BouncyCastle 1.46 as provided by the course starter code. BouncyCastle 1.46 dates to 2011 and accounts for a meaningful share of the reported findings. In a real engagement, upgrading both would be the first remediation step, ahead of anything else in this repository. They were left in place here because changing dependency versions mid-project would have invalidated the before and after scan comparison the assessment depends on.

The certificate is self signed and the keystore is excluded from this repository, so the HTTPS configuration will need your own keystore to run.

## Verification and Testing

Beyond the scan, I verified the compiled application started cleanly and served the endpoint over TLS, checked the produced checksum against an independent SHA-256 implementation for the same input to confirm the hex conversion, and confirmed no sensitive values appear in responses or logs. The endpoint accepts no user supplied input, which removes injection as an attack surface here.

Static analysis finds known flaws in known libraries. It doesn't find logic errors in code written last week, which is why the manual pass mattered alongside it.

## What This Project Demonstrates

The dependency triage rather than the hashing endpoint. Implementing a checksum to spec is a baseline expectation. Reading a scan report, determining which findings actually apply to the deployment context, documenting that judgment so another engineer can review it, and correctly explaining a metric that moved the wrong way is reasoning about risk instead of reacting to tool output.

The habit I most want to carry forward is writing down why a finding was dismissed, not just dismissing it.

## Note on Credentials

The keystore password and NVD API key have been replaced with placeholders. Building this requires supplying your own keystore and requesting an API key from the National Vulnerability Database.
