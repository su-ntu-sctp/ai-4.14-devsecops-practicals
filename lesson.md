# Lesson 4.14: DevSecOps Practicals - Dependabot and OWASP ZAP

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Turn on** Dependabot for a Maven project - alerts, security updates and automatic dependency submission
2. **Distinguish** blocking security checks from non-blocking security checks
3. **Triage** a Dependabot alert - dismiss it with a reason, and compare it with the `.snyk` ignore from Lesson 4.13
4. **Run** an OWASP ZAP baseline (DAST) scan against a deployed application and interpret its alerts

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lesson 4.13 (DevSecOps Foundations), including **Activity 3** (Snyk)
- Your **devops-demo** repository on GitHub (**public**), with the CircleCI pipeline from Lesson 4.13
  (`snyk_scan` failing is expected - the release is blocked until developers fix the findings)
- Your `.snyk` file from Lesson 4.13 (one Snyk finding accepted with a reason and an expiry)
- Your **APP URL** (e.g. `https://devops-demo-sample.onrender.com` or `https://devops-demo-sample.up.railway.app`)
- GitHub Actions basics from Lesson 4.9
- **Admin** access to the repository (it's your own repository, so you have it)

---

## Introduction

In Lesson 4.13, Snyk became a **blocking security check** in your CircleCI pipeline: HIGH and CRITICAL issues stopped publish and deploy, and you accepted one finding temporarily in a `.snyk` file - with a reason and an expiry date.

Today you'll add two more security checks - this time on **GitHub**, in the same `devops-demo` repository:

| Check | Tool | Type (Lesson 4.13) | When it runs | Blocks deploy? |
|---|---|---|---|---|
| Dependency monitoring | **Dependabot** (built into GitHub) | SCA | Whenever `pom.xml` changes, and whenever a new advisory is published | **No** |
| Running-app scan | **OWASP ZAP** (GitHub Actions) | DAST | On demand after a deploy + every Monday | **No** |

```
CircleCI (blocking):      build ─► test ──┐
                          snyk_scan ──────┼─► publish ─► deploy ─► notify
                          snyk_code ──────┘                │
                                                           ▼
                                                    Render / Railway
                                                           ▲
GitHub (non-blocking):    Dependabot: watches your         │
                          dependencies                     │
                          ZAP: scans the live app ─────────┘
```

Neither new check is connected to CircleCI - they never block a release. They **report**, and the results land in GitHub.

---

## Part 1 - Blocking and Non-blocking Security Checks

### Blocking checks

- Run **inside** the delivery pipeline
- A failure **stops** the release
- Must be **reliable** and use a **threshold** you can live with - otherwise nobody can ship
- Today: `snyk_scan` and `snyk_code` in CircleCI (fail on HIGH and CRITICAL)

### Non-blocking checks

- Run **beside** the pipeline, or on a schedule
- A failure **alerts** the team, but doesn't stop the release
- Good for checks that are **noisy**, **slow**, or that find problems in code you **didn't change**
- Today: Dependabot and ZAP

### Why Dependabot is non-blocking here

- Dependabot runs **on GitHub**, outside your CircleCI pipeline - it can't stop a release
- It alerts on **any** severity - LOW, MEDIUM, HIGH or CRITICAL
- It often finds problems in libraries you **didn't change** - and there may be no fix yet
- Your **blocking** checks are still `snyk_scan` and `snyk_code` in CircleCI (HIGH and CRITICAL only)

### Continuous monitoring

New vulnerabilities are published **every day** for libraries that were safe yesterday. You don't need to schedule anything: GitHub keeps a list of your dependencies (the **dependency graph**) and checks it again **every time** a new advisory is published - even if nobody pushes code.

---

## Part 2 - Turn On Dependabot

**Dependabot** is GitHub's built-in dependency scanner. It compares your dependencies with the **GitHub Advisory Database** (which feeds the open OSV.dev database too). It's free for public repositories, and there's **no workflow file** to write - you switch it on in Settings.

| Feature | What it does |
|---|---|
| **Dependency graph** | GitHub's list of every library your project uses |
| **Automatic dependency submission** | GitHub runs **Maven** for you, so the graph includes **transitive** dependencies - not just the ones in `pom.xml` |
| **Dependabot alerts** | Alerts in the **Security** tab when a library in the graph has a known vulnerability |
| **Dependabot security updates** | Opens a **pull request** that upgrades a vulnerable library, when a fix exists |

### What Are Transitive Dependencies?

A **transitive dependency** is a library your project uses only because one of **your** dependencies needs it. You never list it yourself - Maven downloads it for you.

devops-demo's `pom.xml` lists `spring-boot-starter-webmvc`. That starter needs Tomcat to run a web server, so Maven also downloads `tomcat-embed-core` - even though it never appears in your `pom.xml`:

```
pom.xml (you write this)
└── spring-boot-starter-webmvc              ← direct dependency
    └── spring-boot-starter-tomcat          ← transitive
        └── ...
            └── tomcat-embed-core           ← transitive (several levels down)
```

**Why they matter for security:**
- **Most of your code is code you didn't choose** - a few starters bring in dozens of libraries
- **Most vulnerabilities are found there** - the Snyk findings in Lesson 4.13 were in `tomcat-embed-core`, a transitive library
- **A scanner must see the whole tree** - reading `pom.xml` alone only shows the starters
- **You usually can't upgrade them directly** - the fix normally comes through the direct dependency that brings them in (for Spring Boot: a newer Spring Boot version)

**See the tree yourself** - in your devops-demo folder:

```bash
./mvnw dependency:tree
```

Lines with no indent are your **direct** dependencies; everything indented under them is **transitive**.

> **Why automatic dependency submission matters for us:** devops-demo's `pom.xml` lists only the Spring Boot **starters**, without versions - the real libraries (like `tomcat-embed-core`) come in **transitively**. Without submission, GitHub only sees the starters and misses most vulnerabilities.

<!-- INSTRUCTOR: in Spring Boot 4.x the Tomcat path is spring-boot-starter-webmvc → spring-boot-starter-tomcat → spring-boot-starter-tomcat-runtime → tomcat-embed-core (Maven Central, 2026-11-10; the "..." hides the in-between starters). Run ./mvnw dependency:tree on devops-demo before class to show the exact tree. -->

### Step 1: Turn On Dependabot

1. GitHub repository → **Settings**
2. In the sidebar, under **Security and quality**, click **Advanced Security**
3. Under **Dependency graph**:
   - **Dependency graph** - make sure it's enabled (it is by default on public repositories)
   - **Automatic dependency submission** - open the dropdown and select **Enabled**
4. Under **Dependabot**:
   - **Dependabot alerts** - click **Enable**
   - **Malware alerts** - click **Enable**
   - **Dependabot security updates** - click **Enable**
   - Leave **Grouped security updates** off

<!-- INSTRUCTOR: UI labels from GitHub docs (2026-11-10): Settings → "Security and quality" → Advanced Security. Automatic dependency submission needs GitHub Actions enabled and watches the pom.xml in the repository root. Dry-run on devops-demo: check that the submission run succeeds with Spring Boot 4 / Java 21, how many alerts appear, and whether Dependabot can open a security-update PR for a transitive dependency. Malware alerts: off by default; GitHub's malware-alerts docs still say npm only, but the 28 Jul 2026 changelog extended them to more ecosystems (reported as eight, including Maven) - check the setting is there. Auto-triage presets and custom rules: GitHub docs, 2026-11-10. -->

### What Each Dependabot Setting Does

The **Advanced Security** page has more Dependabot settings than we switch on. Here's what each one does, and why we turn it on - or not - today:

| Setting | What it does | Today | Why |
|---|---|---|---|
| **Dependency graph** | GitHub's list of the libraries your project uses | ✅ On (default) | Every other Dependabot feature needs it |
| **Automatic dependency submission** | GitHub runs Maven in GitHub Actions and adds **transitive** libraries to the graph | ✅ On | `pom.xml` only lists the Spring Boot starters - without it, most vulnerable libraries are invisible |
| **Dependabot alerts** | Alerts when a library in the graph has a known **vulnerability** (a bug that can be exploited) | ✅ On | The core of today's lesson |
| **Malware alerts** | Alerts when a library is a known **malicious package** - published on purpose to attack you, e.g. to steal secrets. Data from the OpenSSF malicious-packages list | ✅ On | Supply-chain attacks like the Trivy attack (Lesson 4.13) are real. It's quiet unless something is very wrong - and if it fires, act at once |
| **Dependabot security updates** | Opens a pull request that upgrades a vulnerable library, when a fix exists | ✅ On | Shows the fix, not just the problem |
| **Grouped security updates** | Puts **all** security updates for an ecosystem (e.g. Maven) into **one** pull request | ❌ Off | One PR per alert is easier to read and review today. Teams with many dependencies turn it on to cut PR noise - but one failing upgrade then holds back all the others |
| **Dependabot version updates** (`dependabot.yml`) | Opens a pull request for **every** new version of a dependency - not just security fixes | Only for GitHub Actions (Step 5) | Keeps our SHA pins current. For Maven it would open a PR for every new library version |
| **Auto-triage rule (GitHub preset): Dismiss low impact issues for development-scoped dependencies** | Auto-dismisses some low-risk alerts in **npm** packages used only in development | Leave as is (on by default for public repositories) | It doesn't apply to Maven, so it changes nothing for devops-demo |
| **Auto-triage rule (GitHub preset): Dismiss package malware alerts** | Auto-dismisses malware alerts that flag **every** version of a package - often a false alarm when a company's internal package has the same name as a malicious public one | ❌ Off (default) | We have no internal packages - and it would also hide a **real** malicious dependency |
| **Custom auto-triage rules** | Your own rules that dismiss or snooze alerts automatically (by severity, package, CWE and more), before anyone is notified | ❌ Not today | Triage by hand first, to learn the judgment. Teams add rules once they know which alerts they always accept |
| **Dependabot on Actions runners** | Where Dependabot's own update jobs run | Leave as is | No change needed for a public repository |

> **Auto-triage rules vs your Step 6 dismissal:** a rule dismisses **every** matching alert automatically, now and in the future. Your Step 6 dismissal is **one** decision about **one** alert, with your reason. Rules save time - but only for decisions you'd make the same way every time.

### Step 2: Watch the Dependency Submission Run

Turning on automatic dependency submission starts a run straight away:

1. Open the **Actions** tab - a dependency submission run appears, started by GitHub (you didn't write this workflow)
2. Wait for it to finish (1-3 minutes) - it runs Maven to resolve every dependency
3. From now on it runs again **every time a commit to `main` changes `pom.xml`**

**Then check the graph:** **Insights** tab → **Dependency graph** → **Dependencies**. You should see far more libraries than your `pom.xml` lists - including `tomcat-embed-core`, marked as **transitive**.

### Step 3: Read the Alerts

1. Repository → **Security** tab → **Dependabot**
2. Click an alert

**Each alert shows:**

| Field | Meaning |
|---|---|
| **Package** | The vulnerable library and version - often **transitive** |
| **Severity** | Critical, High, Moderate or Low |
| **Advisory** | The CVE / GHSA ID, a description, and references |
| **Patched version** | The first version that fixes it (if there is one) |
| **Dependency path** | Which of **your** dependencies brings it in |

Expect alerts: devops-demo's dependencies already have known vulnerabilities - the same ones `snyk_scan` stopped in Lesson 4.13.

### Step 4: Look at a Security Update Pull Request

When a fix exists, Dependabot opens a pull request that upgrades the library:

1. Open the **Pull requests** tab - look for PRs by **dependabot**
2. Open one: it shows the version change in `pom.xml` and links to the advisory
3. Notice the **checks**: CircleCI runs your pipeline on the PR - including the **blocking** Snyk checks

**Don't merge it today.** As in Lesson 4.13, fixing the app is the developers' job: they review and test the upgrade, then merge it.

> No PR for an alert? Dependabot can't always upgrade a **transitive** library directly in Maven, or there may be no fix yet. Developers then fix it in `pom.xml` themselves - see the next section.

### How Developers Fix a Transitive Dependency

Say Dependabot (or Snyk) reports a vulnerability in **`tomcat-embed-core`**. Your `pom.xml` doesn't list Tomcat anywhere - so where does its version come from, and how do you change it?

#### Where the versions come from

Your `pom.xml` has a **parent**, and the parent has a parent:

```
devops-demo/pom.xml
└── parent: spring-boot-starter-parent 4.1.1
    └── parent: spring-boot-dependencies 4.1.1   ← sets a version for every library Spring Boot supports
```

`spring-boot-dependencies` stores each version in a **property**. Spring Boot lists them all in the **Version Properties** table:

➡️ [Spring Boot - Version Properties](https://docs.spring.io/spring-boot/appendix/dependency-versions/properties.html) (use the version menu on the page to match your Spring Boot version)

| Library (search the table for) | Property |
|---|---|
| Tomcat | `tomcat.version` |
| Jackson 3 (`tools.jackson...`) | `jackson-bom.version` |

The name isn't always `<library>.version` - **look it up, never guess**. A property you get wrong does nothing, and Maven won't warn you.

#### Step A: Check which version you have

```bash
./mvnw dependency:tree -Dincludes=org.apache.tomcat.embed
```

(On Windows: `mvnw.cmd` instead of `./mvnw`.)

`-Dincludes` shows only the libraries whose **groupId** matches. Every Maven library has an "address" made of these parts - this is one line of the output:

```
org.apache.tomcat.embed:tomcat-embed-core:jar:11.0.N:compile
└────── groupId ──────┘ └─ artifactId ──┘  │    │       └─ scope
                                           │    └─ version
                                           └─ packaging
```

| Part | Meaning | Example |
|---|---|---|
| **groupId** | Who publishes it - usually a reversed domain name | `org.apache.tomcat.embed` (Apache Tomcat) |
| **artifactId** | Which library | `tomcat-embed-core` |
| **version** | Which release | `11.0.N` |

The output looks like this - the path from **your** project down to Tomcat:

```
[INFO] com.example:devops-demo:jar:0.0.1-SNAPSHOT
[INFO] \- org.springframework.boot:spring-boot-starter-webmvc:jar:4.1.1:compile
[INFO]    \- org.springframework.boot:spring-boot-starter-tomcat:jar:4.1.1:compile
[INFO]       \- ...
[INFO]          +- org.apache.tomcat.embed:tomcat-embed-core:jar:11.0.N:compile
[INFO]          +- org.apache.tomcat.embed:tomcat-embed-el:jar:11.0.N:compile
[INFO]          \- org.apache.tomcat.embed:tomcat-embed-websocket:jar:11.0.N:compile
```

To look at **one** library only, use `groupId:artifactId`: `-Dincludes=org.apache.tomcat.embed:tomcat-embed-core`

<!-- INSTRUCTOR: replace this sample with the real output of `./mvnw dependency:tree -Dincludes=org.apache.tomcat.embed` on devops-demo (Spring Boot 4.1.1), including the real Tomcat version and the starters between spring-boot-starter-tomcat and the Tomcat jars. -->

#### Step B, Fix 1 (try first): Upgrade Spring Boot - 4.1.1 → 4.1.2

A newer Spring Boot **patch** release usually brings a fixed Tomcat that's already tested with the rest of Spring Boot. Change **one number** in `pom.xml`:

```diff
 <parent>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-parent</artifactId>
-    <version>4.1.1</version>
+    <version>4.1.2</version>
     <relativePath/> <!-- lookup parent from repository -->
 </parent>
```

Stay on the same **4.1.x** line - a patch upgrade, not a jump to a new minor or major version.

#### Step B, Fix 2 (only if Fix 1 isn't enough): Override `tomcat.version`

If even the newest Spring Boot patch still ships the vulnerable Tomcat, set the version yourself. Take the **Patched version** from the Dependabot alert and add **one line** to `<properties>`:

```diff
 <properties>
     <java.version>21</java.version>
+    <tomcat.version>11.0.N</tomcat.version>  <!-- the patched version from the alert -->
 </properties>
```

- **Don't** add a `<dependency>` for `tomcat-embed-core` - the property moves **all** the Tomcat libraries (core, el, websocket) together
- Stay on the same Tomcat line (11.0.x) - only the last number changes
- **Remove the line** the next time you upgrade Spring Boot - otherwise it may hold Tomcat back at an older version

#### Step C: Check locally

```bash
# 1. Which Tomcat version does Maven use now?
./mvnw dependency:tree -Dincludes=org.apache.tomcat.embed

# 2. (Fix 2 only) Is your property used? Shows the value Maven ends up with
./mvnw help:effective-pom | grep "<tomcat.version>"

# 3. Does the app still work?
./mvnw test
```

Every Tomcat line should show the **new** version, and the tests should pass.

#### Step D: Commit and push

```bash
git add pom.xml
git commit -m "Upgrade Spring Boot to 4.1.2 to fix tomcat-embed-core vulnerability"
git push origin main
```

CircleCI runs again (including the Snyk checks), automatic dependency submission updates the dependency graph, and the Dependabot alert **closes by itself** once the vulnerable version is gone.

<!-- INSTRUCTOR: Spring Boot 4.1.2 is the latest 4.1.x patch (checked 2026-11-10). Before class, check which Tomcat version 4.1.1 and 4.1.2 use (Dependency Versions page or dependency:tree) and whether 4.1.2 closes devops-demo's Tomcat alerts - if not, demo Fix 2. Decide whether students push the fix in class: pushing to main also runs publish and deploy if every blocking check passes. -->

### Step 5: Keep Your Pinned Actions Up to Date

You pin GitHub Actions by **commit SHA** (Lesson 4.13) - but who updates the SHA when a new version comes out? Dependabot can. Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

| Part | Meaning |
|---|---|
| `package-ecosystem: "github-actions"` | Check the actions used in `.github/workflows/` |
| `schedule: weekly` | Look for new versions once a week |

When a pinned action has a new release, Dependabot opens a PR that updates the **SHA** - and the `# v...` comment next to it. You get pinning **and** updates.

```bash
git add .github/dependabot.yml
git commit -m "Add Dependabot version updates for GitHub Actions"
git push origin main
```

> Dependabot can do the same for `pom.xml` (`package-ecosystem: "maven"`) - but that opens a PR for every new library version. We only use it for actions today.

### Step 6: Triage One Alert

In Lesson 4.13 you accepted one Snyk finding in `.snyk`. Dependabot doesn't read `.snyk` - **every tool keeps its own list** of accepted findings. So the same issue may still show up here.

1. Open the Dependabot alert for the **same** issue you accepted in `.snyk` (or any one LOW or MODERATE alert)
2. Click **Dismiss alert** and pick a reason - for example **Risk is tolerable to this project**
3. Add a **comment**: why, and when to look at it again, e.g. *"Same as .snyk SNYK-JAVA-...: accepted until 2026-12-31, fix planned with the next Spring Boot upgrade"*
4. Click **Dismiss alert**

The alert moves to **Closed**, with your reason and comment. You can reopen it at any time.

| | `.snyk` (Lesson 4.13) | Dependabot dismissal |
|---|---|---|
| Where | A file in your repository | GitHub's Security tab |
| Reviewed by | Pull request, like code | Whoever has access |
| Expiry | `--expiry` date - comes back automatically | **No expiry** - put the review date in the comment |

---

## Part 3 - DAST: Scan Your Live App with OWASP ZAP

### What Is OWASP ZAP?

**ZAP (Zed Attack Proxy)** is a free, open-source **DAST** tool. Instead of reading code, it talks to your **running** application over HTTP - like a browser, or an attacker - and reports what it finds.

| Scan type | What ZAP does | Safe for a live app? |
|---|---|---|
| **Baseline** | Crawls the app for about a minute and **passively** inspects the responses: headers, cookies, error pages | ✅ Yes - it sends normal requests only |
| **Full scan** | Also **actively attacks** the app: injection, path traversal and more | ❌ Only against test environments you own |

Today we run the **baseline** scan.

> **⚠️ Only scan applications you own or have written permission to test.** Scanning someone else's website can be illegal.

### Step 1: Store Your APP URL as a Repository Variable

The URL isn't a secret, but it's still configuration - keep it out of the workflow file.

1. GitHub repository → **Settings** → **Secrets and variables** → **Actions**
2. Click the **Variables** tab → **New repository variable**
3. **Name:** `APP_URL`
4. **Value:** your APP URL, e.g. `https://devops-demo-sample.up.railway.app` or `https://devops-demo-sample.onrender.com`
5. Click **Add variable**

### Step 2: Create the Workflow File

Create `.github/workflows/zap-baseline.yml`:

```yaml
name: zap-baseline-scan

on:
  workflow_dispatch:       # run it by hand after a deploy
  schedule:
    - cron: "0 2 * * 1"    # every Monday 02:00 UTC (10:00 Singapore)

permissions:
  contents: read

jobs:
  zap-baseline:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          persist-credentials: false

      - name: ZAP baseline scan of the app
        uses: zaproxy/action-baseline@de8ad967d3548d44ef623df22cf95c3b0baf8b25 # v0.15.0
        with:
          target: ${{ vars.APP_URL }}
          allow_issue_writing: false
          fail_action: false

      - name: Upload ZAP report (viewable in browser)
        if: always()
        uses: actions/upload-artifact@v7
        with:
          path: report_html.html
          archive: false

      - name: Add ZAP report to job summary
        if: always()
        run: cat report_md.md >> "$GITHUB_STEP_SUMMARY"
```

| Part | Meaning |
|---|---|
| `workflow_dispatch` | Run it by hand after a deploy |
| `schedule` | Re-check the live app every Monday |
| `vars.APP_URL` | Your repository variable from Step 1 |
| `allow_issue_writing: false` | Don't open a GitHub issue - we'll read the report instead |
| `fail_action: false` | Don't fail the workflow because of alerts - a **non-blocking** check, like Dependabot |
| Pinned SHAs | Same rule as Lesson 4.13 - and Dependabot keeps them up to date (Part 2, Step 5) |

<!-- INSTRUCTOR: actions/upload-artifact@v7 is pinned by tag - replace it with its commit SHA (`git ls-remote https://github.com/actions/upload-artifact.git refs/tags/v7.0.2^{}`) so it follows the lesson's own rule. zaproxy/action-baseline also uploads its own zipped `zap_scan` artifact. zaproxy/action-baseline v0.15.0 = de8ad967..., actions/checkout v7.0.1 = 3d3c42e5... (git ls-remote, 2026-10-21). The action pulls ghcr.io/zaproxy/zaproxy:stable by default (a tag) - an optional extension is setting docker_name to a digest. Run it once against your own Railway app to see which alerts appear before class. -->

```bash
git add .github/workflows/zap-baseline.yml
git commit -m "Add ZAP baseline scan workflow"
git push origin main
```

### Step 3: Run the Scan

1. Open your **APP URL** + `/hello` in a browser first - a sleeping free Render service takes about a minute to wake up
2. Repository → **Actions** tab → **zap-baseline-scan** (left sidebar)
3. Click **Run workflow** → **Run workflow**
4. Wait 2-3 minutes for it to finish

### Step 4: Read the Report

1. Open the finished run - the job **Summary** shows the report
2. For the full report, scroll to **Artifacts** and click **report_html.html** - it opens in your browser (no unzipping)

**Each alert shows:**

| Field | Meaning |
|---|---|
| **Risk** | High, Medium, Low or Informational |
| **Alert** | What ZAP found, e.g. "Content Security Policy (CSP) Header Not Set" |
| **URL** | Where it found it |
| **Description** | Why it matters |
| **Solution** | How to fix it |

**What you'll probably see** - devops-demo is a minimal API without Spring Security, so ZAP is likely to report **missing security headers**, for example:

| Alert | What the header does |
|---|---|
| Content Security Policy (CSP) Header Not Set | Controls which scripts and resources a browser may load |
| Missing Anti-clickjacking Header | Stops other sites from embedding your pages in a frame |
| X-Content-Type-Options Header Missing | Stops browsers guessing a response's content type |

**Notice what ZAP found that Snyk and Dependabot can't:** these are problems in how the app **behaves when running** - not in its libraries. That's why DAST complements SCA.

### Step 5: Discuss

1. Why is the baseline scan safe to run against your live app, while a full scan is not?
2. Which of the alerts would you fix first, and why?
3. Snyk, Dependabot and ZAP all reported different things. What does each one see that the others can't?
4. Should the ZAP scan block deployment? What would happen if it did?

<!-- Answers: 1) Baseline only sends normal requests and inspects responses; a full scan sends attack payloads that can change data or take the app down. 2) Usually the highest risk first; missing headers are Low/Medium but cheap to fix (Spring Security adds most of them by default). 3) Snyk/Dependabot: known CVEs in libraries (SCA). ZAP: behaviour of the running app - headers, cookies, errors (DAST). 4) ZAP needs a running app, so it can only run after deploy - blocking would mean deploying to a test environment first, scanning, then promoting to production. -->

---

## Part 4 - Your Complete DevSecOps Setup

| Where | Tool | Type | Role |
|---|---|---|---|
| Snyk website | Snyk | SCA | Dashboard; re-tests your project and emails you about new issues |
| CircleCI | Snyk (`snyk_scan`, `snyk_code`) | SCA + SAST | **Blocking** - stop publish and deploy on HIGH and CRITICAL |
| GitHub | Dependabot | SCA | **Non-blocking** - continuous; alerts in the Security tab, fix PRs |
| GitHub Actions | OWASP ZAP | DAST | **Non-blocking** - scans the live app |

**Defence in depth:** different tools, different data, different stages. No single scanner catches everything.

---

## Best Practices for DevSecOps

**1. Block only what must never ship; report everything else** - blocking checks need reliable thresholds; non-blocking checks can report everything

**2. Monitor continuously** - new CVEs appear for code you haven't touched

**3. Pin actions and tools by commit SHA or hash** - not by tag (Lesson 4.13) - and let Dependabot keep the pins up to date

**4. Give workflows the least permissions** - `contents: read` plus only what they need

**5. DAST only against apps you own** - baseline for live apps; full scans in test environments

**6. Every accepted risk needs a reason and a review date** - in `.snyk`, in a Dependabot dismissal, or anywhere else

---

## Summary

### What You Accomplished Today

1. ✅ Turned on Dependabot alerts, security updates and **automatic dependency submission** - so transitive Maven dependencies are checked too
2. ✅ Read Dependabot alerts in the GitHub **Security** tab and looked at a security-update pull request
3. ✅ Added `.github/dependabot.yml` so pinned GitHub Actions stay up to date
4. ✅ Dismissed one alert with a reason - and saw that every tool keeps its own list of accepted findings
5. ✅ Compared a **non-blocking** check (Dependabot) with **blocking** checks (Snyk in CircleCI)
6. ✅ Ran an OWASP ZAP baseline scan against your live app
7. ✅ Read ZAP's report and compared DAST with SCA

---

## Troubleshooting Guide

### Issue 1: No Dependabot alerts, or only a few

**Cause:** Dependabot alerts are off, or GitHub only sees the starters in `pom.xml`

**Solution:**
1. **Settings** → **Advanced Security**: **Dependency graph**, **Automatic dependency submission** and **Dependabot alerts** are all on
2. **Actions** tab: the dependency submission run finished successfully
3. **Insights** → **Dependency graph**: transitive libraries such as `tomcat-embed-core` are listed

---

### Issue 2: The dependency submission run fails

**Solution:**
1. **Settings** → **Actions** → **General**: GitHub Actions must be allowed
2. `pom.xml` must be in the **root** of the repository
3. Open the failed run's log - a Maven build error in the project stops the submission too

---

### Issue 3: Dependabot didn't open a pull request for an alert

**Cause:** There's no fixed version yet, or the vulnerable library is **transitive** and Dependabot can't upgrade it directly

**Solution:** That's expected - developers upgrade the **Spring Boot parent** version instead. Also check **Dependabot security updates** is enabled.

---

### Issue 4: "Unable to resolve action" / workflow not found

**Cause:** A commit SHA in the ZAP workflow is wrong or incomplete

**Solution:** Copy the full 40-character SHA again - one wrong character breaks it.

---

### Issue 5: ZAP fails with "target not reachable" or an empty URL

**Solution:**
1. Check the repository **variable** (not secret) `APP_URL` exists - Settings → Secrets and variables → Actions → **Variables**
2. Check the URL starts with `https://` and the app is running (open `/hello` in your browser - a sleeping Render service needs about a minute)

---

### Issue 6: There's no "Run workflow" button

**Solution:** The workflow file must be on the **default branch** (`main`) and include `workflow_dispatch:`.

---

## Additional Resources

- [GitHub: About Dependabot alerts](https://docs.github.com/en/code-security/dependabot/dependabot-alerts/about-dependabot-alerts)
- [GitHub: Configuring automatic dependency submission](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configuring-automatic-dependency-submission-for-your-repository)
- [GitHub: Keeping your actions up to date with Dependabot](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot)
- [GitHub Advisory Database](https://github.com/advisories)
- [OWASP ZAP](https://www.zaproxy.org/)
- [ZAP Baseline Scan](https://www.zaproxy.org/docs/docker/baseline-scan/)
- [GitHub: Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
