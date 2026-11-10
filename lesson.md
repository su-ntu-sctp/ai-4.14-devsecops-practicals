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

> **Why automatic dependency submission matters for us:** devops-demo's `pom.xml` lists only the Spring Boot **starters**, without versions - the real libraries (like `tomcat-embed-core`) come in **transitively**. Without submission, GitHub only sees the starters and misses most vulnerabilities.

### Step 1: Turn On Dependabot

1. GitHub repository → **Settings**
2. In the sidebar, under **Security and quality**, click **Advanced Security**
3. Under **Dependency graph**:
   - **Dependency graph** - make sure it's enabled (it is by default on public repositories)
   - **Automatic dependency submission** - open the dropdown and select **Enabled**
4. Under **Dependabot**:
   - **Dependabot alerts** - click **Enable**
   - **Dependabot security updates** - click **Enable**

<!-- INSTRUCTOR: UI labels from GitHub docs (2026-11-10): Settings → "Security and quality" → Advanced Security. Automatic dependency submission needs GitHub Actions enabled and watches the pom.xml in the repository root. Dry-run on devops-demo: check that the submission run succeeds with Spring Boot 4 / Java 21, how many alerts appear, and whether Dependabot can open a security-update PR for a transitive dependency. -->

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

> No PR for an alert? Dependabot can't always upgrade a **transitive** library directly in Maven, or there may be no fix yet. Developers then upgrade the **Spring Boot parent** version instead.

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
