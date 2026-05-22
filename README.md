# AWS CodePipeline Secure React Delivery Architecture

A codified, continuous delivery pipeline utilizing AWS CodePipeline and CodeBuild to govern the source, build, and deployment stages of a React.js application.

This repository serves as an engineering blueprint for continuous security automation. To enforce strict cost controls, minimize attack surface, and respect cloud resource lifecycles, the infrastructure is maintained as ephemeral, version-controlled code. Pipeline integrity and syntax validation are strictly governed via local pre-flight linting and deterministic security gates before any code interacts with a cloud environment.

---

## Pipeline Architecture & Security Gates

The continuous delivery cycle enforces three sequential, automated security gates within the AWS CodeBuild runtime container. If any gate detects a violation that exceeds defined corporate risk parameters, the pipeline terminates execution immediately, isolating the flawed code before it can compile or generate deployable distribution artifacts.

| Pipeline Stage | Security Tool | Core Focus | Risk Threshold / Policy |
| :--- | :--- | :--- | :--- |
| **Gate 1: Secrets Scan** | `GitLeaks` | Hardcoded credentials, API keys, and certificates | **Absolute Zero Tolerance.** Aborts immediately on any match. |
| **Gate 2: Supply Chain** | `npm audit` | Third-party open-source dependency vulnerabilities | **High & Critical Block.** Halts if severe flaws are found. |
| **Gate 3: SAST Engine** | `Semgrep` | Proprietary code flaws (XSS, insecure logic) | **OWASP Top 10 Failures.** Fails build on strict rule matches. |
| **Final Stage** | `npm ci` | Hardened production artifact compilation | **Deterministic Verification.** Rejects build if lockfile drifts. |

### Gate 1: Hardened Secret Detection (GitLeaks)
* **Why it was chosen:** Hardcoded credentials, API tokens, and private cryptographic keys committed to source control represent one of the most critical vectors for cloud infrastructure compromise. 
* **Risk Threshold:** **Absolute Zero Tolerance.** GitLeaks scans the entire commit history, structural diffs, and developer workspaces. If any high-entropy string matching known provider signatures (AWS, Okta, GitHub, etc.) is flagged, the pipeline aborts immediately with a non-zero exit code to prevent token exposure.

### Gate 2: Software Supply Chain Dependency Auditing (`npm audit`)
* **Why it was chosen:** Modern web frameworks rely heavily on nested, third-party open-source packages. Vulnerabilities buried deep within a dependency tree can introduce critical execution risks (such as Remote Code Execution or Prototype Pollution) directly into the client-side application.
* **Risk Threshold:** **High & Critical Vulnerability Block.** The audit engine explicitly filters for structural flaws matching **High or Critical** severity rankings in the public vulnerability database. The pipeline tolerates low or informational alerts to prevent operational friction, but aggressively halts compilation if high-risk packages are present.

### Gate 3: Static Application Security Testing (Semgrep SAST)
* **Why it was chosen:** While dependency scanning protects against third-party risk, SAST actively inspects the proprietary application code written by the developer. It identifies custom programming flaws like Cross-Site Scripting (XSS), insecure configuration variables, or broken client-side access logic before compilation.
* **Risk Threshold:** **CWE / OWASP Top 10 Policy Failures.** Executing with the `--error` flag, the Semgrep engine parses code against pre-configured security rulesets (`p2/r/security-codecheck`). Any code pattern matching a validated security rule breaks the build path cleanly, forcing developer remediation prior to distribution.
