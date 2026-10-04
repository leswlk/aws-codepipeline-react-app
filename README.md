 # AWS CodePipeline Secure React Delivery Architecture 🚀

### 🤖 Introduction
This repository serves as an engineering blueprint for continuous security automation, providing a codified, continuous delivery pipeline utilizing AWS CodePipeline and CodeBuild to govern the source, build, and deployment stages of a React.js application. 

By following this guide, you will learn how to build a fully automated CI/CD pipeline where every push to your GitHub repository automatically builds and deploys to a static website on Amazon S3, eliminating the need for manual deployments.

---

### 🛡️ Pipeline Architecture & Security Gates
To enforce strict cost controls, minimize the attack surface, and isolate flawed code before it generates deployable artifacts, this pipeline incorporates deterministic security gates within the AWS CodeBuild runtime container.

| Pipeline Stage | Security Tool | Core Focus | Risk Threshold / Policy |
| ------ | ------ | ------ | ------ |
| **Gate 1: Secrets Scan** | GitLeaks | Hardcoded credentials, API keys, and certificates | **Absolute Zero Tolerance.** Aborts immediately on any match. |
| **Gate 2: Supply Chain** | npm audit | Third-party open-source dependency vulnerabilities | **High & Critical Block.** Halts if severe flaws are found. |
| **Gate 3: SAST Engine** | Semgrep | Proprietary code flaws (XSS, insecure logic) | **OWASP Top 10 Failures.** Fails build on strict rule matches. |
| **Final Stage** | npm ci | Hardened production artifact compilation | **Deterministic Verification.** Rejects build if lockfile drifts. |

<details>
<summary><b>🔍 Deep Dive: Security Gate Justifications (Click to Expand)</b></summary>

#### Gate 1: Hardened Secret Detection (GitLeaks)
*   **Why it was chosen:** Hardcoded credentials, API tokens, and private cryptographic keys committed to source control represent one of the most critical vectors for cloud infrastructure compromise.
*   **Risk Threshold:** **Absolute Zero Tolerance.** GitLeaks scans the entire commit history, structural diffs, and developer workspaces. If any high-entropy string matching known provider signatures (AWS, Okta, GitHub, etc.) is flagged, the pipeline aborts immediately with a non-zero exit code to prevent token exposure.

#### Gate 2: Software Supply Chain Dependency Auditing (npm audit)
*   **Why it was chosen:** Modern web frameworks rely heavily on nested, third-party open-source packages. Vulnerabilities buried deep within a dependency tree can introduce critical execution risks (such as Remote Code Execution or Prototype Pollution) directly into the client-side application.
*   **Risk Threshold:** **High & Critical Vulnerability Block.** The audit engine explicitly filters for structural flaws matching **High or Critical** severity rankings in the public vulnerability database. The pipeline tolerates low or informational alerts to prevent operational friction, but aggressively halts compilation if high-risk packages are present.

#### Gate 3: Static Application Security Testing (Semgrep SAST)
*   **Why it was chosen:** While dependency scanning protects against third-party risk, SAST actively inspects the proprietary application code written by the developer. It identifies custom programming flaws like Cross-Site Scripting (XSS), insecure configuration variables, or broken client-side access logic before compilation.
*   **Risk Threshold:** **CWE / OWASP Top 10 Policy Failures.** Executing with the `--error` flag, the Semgrep engine parses code against pre-configured security rulesets. Any code pattern matching a validated security rule breaks the build path cleanly, forcing developer remediation prior to distribution.

</details>

#### Supply Chain Hardening: Why `npm ci` over `npm install`
This pipeline strictly mandates `npm ci` (Clean Install) for production artifact compilation to prevent severe Software Supply Chain Risks:
*   **Strict Lockfile Enforcement:** `npm ci` enforces exact dependency versions from `package-lock.json`, failing the build immediately if there is a discrepancy with the manifest.
*   **Deterministic Builds:** It removes existing `node_modules` folders and installs a pristine dependency graph, eliminating configuration drift across local and cloud environments.
*   **Malicious Package Prevention:** Guarantees compilation of exclusively peer-reviewed and tested code, mitigating the risk of dynamically pulling compromised patch versions.

---

### 🔧 Step-by-Step Deployment Tutorial

#### ➡️ Step 1 - Setup your React.js App on GitHub
First, set up a React app by cloning the provided repository or using your own. Make sure the app is committed to GitHub.
```bash
git clone https://github.com/leswlk/saas-landing-page.git
```

#### ➡️ Step 2 - Create S3 Bucket for Hosting
We will use Amazon S3 as our deploy provider.
1. Head over to the Amazon S3 console and click **Create bucket**.
![Bucket Properties(images/bucketproperties.jpg)
2. Name it something unique (e.g., `demo-react-cicd-bucket`) and select your AWS Region.
Leave the bucket for now; we will configure it for public hosting later.

#### ➡️ Step 3 - Create CodePipeline
1. Go to AWS CodePipeline and click **Create pipeline**.
2. Name your pipeline (e.g., `aws-codepipeline-react-app-demo`) and choose to create a **New service role**.
3. **Add source stage**:
    *   Choose **GitHub (via GitHub App)** as the source provider and connect your account.
    *   Select your repository (e.g., `leswlk/saas-landing-page`) and set the default branch to `main`.

#### ➡️ Step 4 - Create CodeBuild Project
1. In the **Add build stage**, select **AWS CodeBuild** as the provider and click **Create project**.
2. Name the project (e.g., `fuego-cicd-codepipeline-demo`).
3. Choose a managed image: `aws/codebuild/standard:6.0` (or latest) and select "Use a buildspec file".
4. In your GitHub repo's root directory, create a `buildspec.yml` file:
```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo Installing dependencies...
      - npm ci --legacy-peer-deps

  build:
    commands:
      - echo Building the React app...
      - npm run build

artifacts:
  files:
    - '**/*'
  base-directory: dist
  discard-paths: no
```
*Note: This utilizes our mandated `npm ci` command to ensure build determinism and copies the build folder contents as artifacts.*

5. Return to CodePipeline and proceed to the **Add deploy stage**.
6. Select **Amazon S3** as the deploy provider, choose your newly created bucket (`demo-react-cicd-bucket`), and **crucially**, check the box for **Extract file before deploy**.
7. Review and click **Create pipeline**. You will see the pipeline execute successfully through the source, build, and deploy stages.

#### ➡️ Step 5 - Finish up S3 Bucket configuration
1. Go to the Amazon S3 console, select your bucket, and navigate to the **Properties** tab.
2. Scroll down to **Static website hosting**, click **Edit**, and choose **Enable**.
3. Specify `index.html` as the index document and click **Save**.
4. Switch to the **Permissions** tab and click **Edit** under **Block public access (bucket settings)**.
5. Uncheck **Block all public access** and click **Save changes**.
6. Add the following Bucket Policy to allow public read access (replace with your exact bucket name):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::demo-react-cicd-bucket/*"
    }
  ]
}
```
7. Finally, under the **Objects** tab, click on `index.html` and follow the **Object URL** to view your live React application!

---

### 🚀 Local "Pre-Flight" Security Validation
To execute the security verification gates locally without spinning up AWS resources, ensure you have Node.js, Git, and the AWS CLI installed, then execute the following commands in your workspace root:
```bash
# 1. Execute local secret detection
gitleaks detect --verbose

# 2. Execute local production dependency audit
npm audit --production --audit-level=high

# 3. Execute local static analysis scan
semgrep scan --config=p2/r/security-codecheck --error
```

---

### 🗑 Clean Up Resources
When you're done, remember to clean up your AWS resources to avoid a nasty charge waiting for you at the end of the month!
