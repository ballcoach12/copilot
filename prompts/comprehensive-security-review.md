# Comprehensive Security Review Prompt

You are operating in the Security Analyst chat mode. Perform an exhaustive, end-to-end security analysis of the entire workspace, producing a prioritized set of findings and actionable remediation guidance with clear verification steps. Follow the instructions, workflow, and output requirements below. Use the Security Analyst mode’s principles and templates and cite authoritative sources for your recommendations.

## Objective and scope
- Objective: Identify security weaknesses in code, configs, dependencies, and architecture, and propose minimal, maintainable fixes with validation steps.
- Scope: All code in this workspace, including TypeScript/Node.js packages, Go code, configuration files, build scripts, and tests. Consider local tools, CLIs, services, web endpoints, workers, and extension code.
- Deliverables: Threat Model Summary, Security Review Report with prioritized findings, contrast-based remediation examples, and a Verification Plan. Include a References section with links used.

## Tools you may use
- Workspace analysis: `codebase`, `search`, `searchResults`, `usages`, `findTestFiles`, `problems`
- Web research: `fetch`, optionally `openSimpleBrowser`
- Testing: `runTests`
- External code lookups (read-only): `githubRepo`

## Operating constraints
- Evidence-driven and safe: Prefer read-only analysis. Do not generate exploit payloads. Use sanitized examples. Cite sources you consult.
- Prioritization: Assign a severity (Low/Med/High/Critical) and rationale (likelihood × impact). Suggest a CVSS vector if meaningful.
- Contrast-based remediation: Always show Before/After and why the fix works. Keep changes minimal and idiomatic for the stack.

## Workflow
1) Intake and scoping
- Identify components, services, packages, and trust boundaries. Enumerate sensitive data flows (PII, credentials, tokens, secrets).
- Note compliance or policy constraints if applicable.

2) Fast repository triage
- Use `codebase` and `search` to locate high-risk areas: authn/z, input parsing/serialization, file I/O, network/HTTP, SQL/ORM, crypto, secrets, logging, config, and public interfaces (APIs, CLI, workers, schedulers).
- Identify external entry points and privileged operations.

3) Threat modeling (STRIDE)
- List assets, actors, trust boundaries, and cross-boundary flows. For each flow, enumerate STRIDE threats and primary mitigations.
- Produce a concise Ranked Risks list (top 5–10) based on the application context.

4) Attack surface analysis
- Enumerate reachable interfaces, parameters, preconditions, authN/authZ context, and identity. Capture assumptions (e.g., trusted headers, network segmentation) and flag any that are unverified.

5) Detailed code review using OWASP/CWE lens
- Consider OWASP Top 10 categories at minimum: A01–A10. Map findings to OWASP/CWE where possible.
- Review secrets/config hygiene (CORS, headers, TLS verification, SSRF guards), data protection (crypto, key mgmt), logging (coverage and data minimization), supply-chain integrity (signatures, provenance), and deserialization safety.

6) Dependency and configuration review (SCA)
- Inventory ecosystems and manifests: Node/TS (`package.json`, lockfiles), Go (`go.mod`, `go.sum`).
- Identify vulnerable/outdated dependencies; note whether vulnerable functions are actually reachable.
- Draft a triage plan: group updates, pin versions, add tests to cover risky paths, and plan phased rollouts.

7) Language- and framework-specific checks
- TypeScript/Node.js
  - Type system: enable `strict` and `strictNullChecks`; avoid `any`; prefer `unknown` + type guards; type-validate environment variables (e.g., `environment.d.ts` and/or runtime schema parsing); keep `.env*` out of VCS.
  - Code patterns: prohibit `eval`/`new Function`; defend against prototype pollution (prefer `Map` for untrusted keys; allow-list accepted keys); review regexes for ReDoS; avoid insecure serialization/deserialization; parameterized DB queries; encoding/escaping for XSS contexts.
  - Express/web: `helmet`/CSP, least-privilege CORS, `express-rate-limit`, robust validation (`zod`/`yup`/`joi`), secure cookies, CSRF as applicable, outbound TLS verification and timeouts, secure headers verified in responses.
  - Dependencies: run SCA (e.g., `npm audit` or equivalent); review transitive risk; plan safe upgrades with tests.
  - Observability: adopt standardized security logging vocabulary (e.g., OWASP-js) for auth and security-relevant events.
- Go
  - Parsing/serialization: prefer strict decoding defaults. JSON: use `json.Decoder.DisallowUnknownFields()`; decode into typed structs; reject duplicate/ambiguous keys. YAML: `yaml.Decoder.KnownFields(true)`.
  - Concurrency hygiene: avoid goroutine leaks and deadlocks; close channels correctly; honor `ctx.Done()`; consider `errgroup.WithContext` patterns; timeouts on I/O.
  - Data access and web: `database/sql` prepared statements or safe builders; avoid unsafe template bypasses; HTTP server hardening (e.g., `ReadHeaderTimeout`); strong randomness and constant-time crypto operations.
  - Tooling: verify with `go test -race`, `go vet`, fuzzing for parsers and input handlers, and `govulncheck` for low-noise vulnerability detection.
- Other stacks present (briefly):
  - Python (Flask/Django/FastAPI): CSRF as relevant, Pydantic validation, ORM parameterization, `secrets` for randomness, safe deserialization, secure headers and CORS hygiene.
  - Java/Spring: Spring Security, `@Validated` + Bean Validation, JPA parameter binding, disable unsafe deserialization, security headers, CSRF per context.

8) Findings and prioritization
- For each finding, produce: ID, Title, Description, Location(s), Affected Component(s), Severity, Likelihood, Impact, References (OWASP/CWE/docs), suggested CVSS vector (if meaningful), and proof indicator (symptoms/logs/tests).
- Provide risk acceptance options and trade-offs where applicable.

9) Remediation guidance (contrast code)
- Provide Before/After examples, explain why the fix closes the gap, and note any test updates or new tests required. Favor small, safe, maintainable changes aligned with project conventions.

10) Verification
- Define how to validate fixes:
  - TypeScript/Node: unit/integration tests, tests for prototype pollution attempts and ReDoS-prone regexes, assert presence of security headers (Helmet), and run dependency SCA (e.g., `npm audit`).
  - Go: run `go test -race`, fuzz parsing and input handlers, `go vet`, `govulncheck`, and verify strict decoding rejects unknown/duplicate keys. Check for goroutine leaks and cancellation behavior under load.

11) CI gate checks (execute and report)
- Run the following commands (or equivalents) locally or in CI and include a short summary of results with links to any findings:
  - Go security and quality:
    - `go vet ./...`
    - `go test -race ./...`
    - `govulncheck ./...`
  - TypeScript/Node supply chain:
    - `npm audit --audit-level=high` (or ecosystem-appropriate SCA) in each Node/TS package
  - SAST (Semgrep):
    - `semgrep --config auto --error` (or project-specific rules)
- Treat high/critical results as merge-blocking gates in CI; document any temporary waivers with an expiry and mitigation plan.

11) Documentation
- Produce a Threat Model Summary and a Security Review Report using the mode’s templates. Include a Validation Plan and prioritized next steps. Add a References section with links used; for High/Critical issues, cite at least two independent sources when feasible.

## Automation and CI/CD guidance (optional but recommended)
- Suggest SAST on every PR (Semgrep, SonarQube, `gosec` for Go). Fail builds for high/critical issues.
- Suggest SCA: `govulncheck` for Go and `npm audit` (or equivalent) for Node/TS.
- Add secrets scanning (e.g., Gitleaks/GitGuardian) to prevent committed secrets.
- Use tests as quality gates: Go (`go test -race`, fuzzing, `go vet`); Node/TS (regex/ReDoS tests and dependency checks).
 - Make CI gates explicit: run `go vet`, `go test -race`, `govulncheck`, Semgrep, and `npm audit --audit-level=high` on PRs; fail on high/critical.

## Acceptance criteria (exit checklist)
- [ ] Threat Model Summary completed with trust boundaries, assets, and ranked risks.
- [ ] Full Security Review Report produced with mapped OWASP/CWE categories.
- [ ] Findings list includes severity, likelihood, impact, locations, and references.
- [ ] Remediation guidance includes contrast (Before/After) and rationale.
- [ ] Verification Plan details concrete steps and test updates.
- [ ] References section included; High/Critical items have at least two authoritative citations when feasible.

## Output format
- Provide the Threat Model Summary, then the Security Review Report, followed by the full Findings list (with IDs), the Remediation Plan (prioritized), the Verification Plan, and the References section.
- Keep examples sanitized. Do not include exploit payloads. Focus on minimal, correct fixes and clear validation steps.
