# Lesson 4.14: DevSecOps Practicals

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Add** security scanning to an existing CI/CD pipeline
2. **Configure** OWASP Dependency-Check in CircleCI
3. **Interpret** vulnerability reports and risk scores
4. **Update** vulnerable dependencies to secure versions and verify fixes

---

## Prerequisites

Before starting this lesson, ensure you have:

- Completed Lesson 4.13 (DevSecOps Foundations)
- Completed Lesson 4.7 (CI with CircleCI)
- Completed Lesson 4.12 (Continuous Deployment to Railway)
- Your **devops-demo** project with complete CI/CD pipeline
- CircleCI account with project connected
- Access to your GitHub repository

---

## Pre-Class Setup: Obtain NVD API Key

**IMPORTANT:** OWASP Dependency-Check now requires an NVD API key to function properly. You must complete this setup BEFORE the lesson starts.

### Why Do We Need This?

The National Vulnerability Database (NVD) implemented API key requirements in December 2023 to manage server load. Without an API key:
- ❌ Security scans take 20-30 minutes (vs 4-6 minutes with key)
- ❌ Connection failures and timeouts are common
- ❌ Rate limiting causes unpredictable errors

**With an API key:**
- ✅ Fast scans (4-6 minutes first run, 1-2 minutes after)
- ✅ Reliable connections
- ✅ Professional-grade security scanning

---

### Step 1: Request NVD API Key

1. Go to: **https://nvd.nist.gov/developers/request-an-api-key**
2. Fill out the request form:
   - Enter your email address
   - Agree to terms of service
3. Click **"Request API Key"**
4. Wait for validation (usually completes in 2-5 minutes)
5. Check your email for the API key

**Expected email subject:** "NVD API Key Request"

**Copy the API key from the email - you'll need it in Step 2!**

---

**Troubleshooting:**

**If you see "Validation timed out":**
- Wait 10-15 minutes, then try again
- Check your email anyway - key may have been sent despite timeout
- Try incognito/private browsing mode
- Clear browser cache and retry
- Contact your instructor if issue persists

**Still having issues?**
Don't worry - we'll configure the pipeline to handle this gracefully, though scans will be slower.

---

### Step 2: Add API Key to CircleCI

Now configure CircleCI to use your NVD API key:

1. Go to **https://app.circleci.com/**
2. Click on your **devops-demo** project
3. Click **Project Settings** (gear icon in top right)
4. In the left sidebar, click **Environment Variables**
5. Click **Add Environment Variable** button
6. Configure:
   - **Name:** `NVD_API_KEY`
   - **Value:** (paste your API key from email)
7. Click **Add Environment Variable**

**Verify it was added:**
You should see:
```
Name: NVD_API_KEY
Value: ******************************** (hidden)
```

**Security Note:** CircleCI encrypts environment variables. Your API key is secure.

---

**You're now ready for the lesson!** ✅

---

## Introduction

In Lesson 4.13, you learned the theory behind DevSecOps — why security matters, how shift-left works, and the risks in CI/CD pipelines.

Today, you'll put that knowledge into practice. You'll add **automated security scanning** to your CircleCI pipeline so that every code commit is checked for vulnerabilities before deployment.

By the end of this lesson, your pipeline will automatically:
- Scan all dependencies for known vulnerabilities
- Generate detailed security reports
- Block deployment if critical vulnerabilities are found
- Ensure only secure code reaches Railway production

**This is real-world DevSecOps in action!**

---

## Part 1 - Review Current Pipeline

Before adding security, let's review what your pipeline currently does after completing Lesson 4.12.

### Step 1: Check Your Current CircleCI Config

Open your **devops-demo** project and look at `.circleci/config.yml`

**Your current pipeline (from Lesson 4.12 with Railway deployment):**

```yml
version: 2.1

jobs:
  build:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Install dependencies and build
          command: |
            echo "Building the application..."
            mvn clean install -DskipTests
      - save_cache:
          paths:
            - ~/.m2
          key: maven-deps-{{ checksum "pom.xml" }}
      - persist_to_workspace:
          root: .
          paths:
            - target/*.jar

  test:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Run tests
          command: |
            echo "Running tests..."
            mvn test
      - store_test_results:
          path: target/surefire-reports
      - store_artifacts:
          path: target/surefire-reports

  publish:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - attach_workspace:
          at: .
      - setup_remote_docker
      - run:
          name: Build Docker image
          command: |
            echo "Building Docker image..."
            docker build -t $DOCKER_USERNAME/devops-demo:latest .
      - run:
          name: Push to Docker Hub
          command: |
            echo "Logging in to Docker Hub..."
            echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
            echo "Pushing image to Docker Hub..."
            docker push $DOCKER_USERNAME/devops-demo:latest

  deploy:
    docker:
      - image: cimg/node:18.20
    steps:
      - checkout
      - run:
          name: Deploy to Railway
          command: |
            echo "Deploying to Railway..."
            npx @railway/cli up --service=$RAILWAY_SERVICE_ID --ci

workflows:
  build_test_publish_deploy:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          requires:
            - test
      - deploy:
          requires:
            - publish
```

> **📌 Note on the deploy job:** You may notice this deploy job uses `cimg/node:18.20` with `npx @railway/cli`, which is slightly different from the Railway CLI bash installation approach used in Lesson 4.12. Both methods work correctly — this is simply an alternative way to trigger a Railway deployment. Use whichever version matches your existing working config.

**Current flow:**
```
Build → Test → Publish → Deploy to Railway
```

**What's missing? SECURITY!**

---

### Step 2: Understand What We'll Add

**New flow:**
```
Build → Test → Security Scan → Publish → Deploy to Railway
```

**The security scan will:**
1. Check all dependencies in `pom.xml`
2. Match them against CVE database
3. Generate a vulnerability report
4. Fail the build if HIGH or CRITICAL vulnerabilities found
5. Prevent vulnerable code from reaching Docker Hub AND Railway

**This is the security gate!** No critical vulnerabilities = deployment proceeds. Vulnerabilities found = pipeline stops!

---

## Part 2 - Add Security Scanning

Now let's add automated security scanning to your pipeline.

### Step 1: Understand OWASP Dependency-Check

**OWASP Dependency-Check** is a tool that:
- Scans your `pom.xml` for dependencies
- Downloads the National Vulnerability Database (NVD)
- Checks each dependency version against known CVEs
- Generates an HTML report
- Returns exit code (0 = pass, 1 = fail)

**Maven command:**
```bash
mvn org.owasp:dependency-check-maven:check
```

---

### Step 2: Configure OWASP Plugin in pom.xml

**Before adding the CircleCI job, we need to configure the OWASP plugin.**

**Why?** The plugin needs:
1. Your NVD API key (to download vulnerability database quickly)
2. Failure threshold (what severity should fail the build)
3. OSS Index disabled (to avoid requiring additional credentials)

**Open `pom.xml` and add this plugin configuration in the `<build><plugins>` section:**

```xml
<build>
    <plugins>
        <!-- Your existing Spring Boot plugin -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        
        <!-- ADD THIS: OWASP Dependency-Check Plugin -->
        <plugin>
            <groupId>org.owasp</groupId>
            <artifactId>dependency-check-maven</artifactId>
            <version>12.1.0</version>
            <configuration>
                <!-- Use NVD API key from CircleCI environment variable -->
                <nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>
                
                <!-- Fail build on HIGH (7.0+) or CRITICAL (9.0+) vulnerabilities -->
                <failBuildOnCVSS>7</failBuildOnCVSS>
                
                <!-- Disable OSS Index to avoid extra credentials -->
                <ossindexAnalyzerEnabled>false</ossindexAnalyzerEnabled>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**What each setting does:**

- **`nvdApiKey`**: Reads your API key from CircleCI environment variable
  - Format: `${env.VARIABLE_NAME}` tells Maven to read from environment
  - CircleCI automatically provides this during the build

- **`failBuildOnCVSS`**: Sets the severity threshold
  - `7` = Fail on HIGH (7.0-8.9) or CRITICAL (9.0-10.0)
  - MEDIUM (4.0-6.9) and below won't fail the build
  - Provides warnings but allows deployment

- **`ossindexAnalyzerEnabled`**: Disables secondary analyzer
  - OWASP Dependency-Check uses two sources: NVD and OSS Index
  - OSS Index requires separate Sonatype credentials
  - NVD alone is sufficient for DevSecOps
  - Keeps configuration simple

**Commit this change:**

```bash
git add pom.xml
git commit -m "Configure OWASP Dependency-Check plugin"
git push origin main
```

**Note:** This won't run the security scan yet - we're just configuring the plugin. Next, we'll add it to the CircleCI pipeline.

---

### Step 3: Add Security Scan Job to config.yml

Open `.circleci/config.yml` and add this new job after the `test` job:

```yml
  # Security scanning job
  security_scan:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      
      - run:
          name: Run Dependency Check
          command: |
            echo "Running security scan..."
            mvn org.owasp:dependency-check-maven:check
      
      - store_artifacts:
          path: target/dependency-check-report.html
          destination: security-report
      
      - store_test_results:
          path: target/dependency-check-report.xml
```

---

### Step 4: Update Workflow to Include Security

Update the `workflows` section at the bottom of your config:

**Before:**
```yml
workflows:
  build_test_publish_deploy:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          requires:
            - test
      - deploy:
          requires:
            - publish
```

**After (add security_scan between test and publish):**
```yml
workflows:
  build_test_publish_deploy:
    jobs:
      - build
      - test:
          requires:
            - build
      - security_scan:
          requires:
            - test
      - publish:
          requires:
            - security_scan
      - deploy:
          requires:
            - publish
```

**New flow:**
```
build → test → security_scan → publish → deploy
                     ↓
              If HIGH/CRITICAL vulnerabilities found,
              pipeline STOPS here!
              No Docker image published!
              No Railway deployment!
```

---

### Step 5: Complete Updated config.yml

Here's your complete updated configuration:

```yml
version: 2.1

jobs:
  build:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Install dependencies and build
          command: |
            echo "Building the application..."
            mvn clean install -DskipTests
      - save_cache:
          paths:
            - ~/.m2
          key: maven-deps-{{ checksum "pom.xml" }}
      - persist_to_workspace:
          root: .
          paths:
            - target/*.jar

  test:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Run tests
          command: |
            echo "Running tests..."
            mvn test
      - store_test_results:
          path: target/surefire-reports
      - store_artifacts:
          path: target/surefire-reports

  security_scan:
    docker:
      - image: cimg/openjdk:21.0
    steps:
      - checkout
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      - run:
          name: Run Dependency Check
          command: |
            echo "Running security scan..."
            mvn org.owasp:dependency-check-maven:check
      - store_artifacts:
          path: target/dependency-check-report.html
          destination: security-report
      - store_test_results:
          path: target/dependency-check-report.xml

  publish:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - attach_workspace:
          at: .
      - setup_remote_docker
      - run:
          name: Build Docker image
          command: |
            echo "Building Docker image..."
            docker build -t $DOCKER_USERNAME/devops-demo:latest .
      - run:
          name: Push to Docker Hub
          command: |
            echo "Logging in to Docker Hub..."
            echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
            echo "Pushing image to Docker Hub..."
            docker push $DOCKER_USERNAME/devops-demo:latest

  deploy:
    docker:
      - image: cimg/node:18.20
    steps:
      - checkout
      - run:
          name: Deploy to Railway
          command: |
            echo "Deploying to Railway..."
            npx @railway/cli up --service=$RAILWAY_SERVICE_ID --ci

workflows:
  build_test_publish_deploy:
    jobs:
      - build
      - test:
          requires:
            - build
      - security_scan:
          requires:
            - test
      - publish:
          requires:
            - security_scan
      - deploy:
          requires:
            - publish
```

---

### Step 6: Commit and Push Changes

```bash
git add .circleci/config.yml
git commit -m "Add security_scan job to CI/CD pipeline"
git push origin main
```

---

## Part 3 - Run Pipeline with Security Scan

Now watch your pipeline run with the new security scan!

### Step 1: Go to CircleCI Dashboard

1. Open https://app.circleci.com/
2. Click on your **devops-demo** project
3. You should see a new pipeline running

**Pipeline stages:**
```
1. build          (running...)
2. test           (waiting...)
3. security_scan  (waiting...)
4. publish        (waiting...)
5. deploy         (waiting...)
```

---

### Step 2: Watch Security Scan Execute

**Timeline expectations:**

**build job:** ~10-15 seconds

**test job:** ~10-15 seconds

**security_scan job:** ~4-6 minutes (first run with API key)
- Downloads NVD database (~3 minutes first time)
- Processes CVE records (~30 seconds)
- Analyzes dependencies (~1 minute)
- Generates HTML report (~30 seconds)
- **Subsequent runs:** ~1-2 minutes (uses cached database)

**publish job:** ~30 seconds (if security passes)

**deploy job:** ~45 seconds (if publish succeeds)

**Total pipeline time (first run):** ~6-7 minutes

---

### Step 3: Understanding Security Scan Output

**In CircleCI, click on the `security_scan` job to see logs:**

**Expected output:**

```
Running security scan...
[INFO] Scanning for projects...
[INFO] --- dependency-check-maven:12.1.0:check @ devops-demo ---
[INFO] Checking for updates
[INFO] Download Started for NVD CVE - Modified
[INFO] Download Complete for NVD CVE - Modified  (178534 ms)
[INFO] Processing Started for NVD CVE - Modified
[INFO] Processing Complete for NVD CVE - Modified  (35892 ms)
[INFO] 
[INFO] Scanning Dependencies:
[INFO]   - spring-boot-starter-web-3.2.0.jar
[INFO]   - spring-boot-starter-test-3.2.0.jar
[INFO]   - log4j-api-2.24.3.jar
[INFO] 
[INFO] Generating Report...
[INFO] Found 1 vulnerabilities
[INFO] 
[INFO] BUILD SUCCESS
```

**Wait - it found a vulnerability but passed?** Yes! This is expected — read on!

---

### Step 4: Access the Security Report

**View the report:**

1. In CircleCI, go to **Artifacts** tab
2. Click **security-report/dependency-check-report.html**
3. Report opens in new tab

**Summary:**
```
Scan Date: 2026-02-20
Dependencies Scanned: 34 (16 unique)
Vulnerable Dependencies: 1
Critical: 0
High: 0
Medium: 1
Low: 0
```

You'll see one MEDIUM severity vulnerability in **log4j-api-2.24.3.jar**

---

### Step 5: Understanding the Results

**Why did the build PASS if a vulnerability was found?**

**Your configuration:**
```xml
<failBuildOnCVSS>7</failBuildOnCVSS>
```

**This means:**
- ✅ **MEDIUM (4.0-6.9):** Build PASSES with warning
- ❌ **HIGH (7.0-8.9):** Build FAILS
- ❌ **CRITICAL (9.0-10.0):** Build FAILS

**The log4j vulnerability is MEDIUM severity (~5.0), so the build passes!**

**This is good practice because:**
- Not all vulnerabilities are equal — MEDIUM issues may not be exploitable in your context
- Prevents alert fatigue — focus on HIGH/CRITICAL first
- Allows flexible security — HIGH/CRITICAL = immediate fix, MEDIUM = next sprint

---

## Part 4 - Add HIGH Severity Vulnerability

Now let's intentionally add a HIGH severity vulnerability to see the pipeline actually FAIL.

### Step 1: Before Adding Vulnerable Dependency

**What we're about to do:**
- Add commons-collections4 version 4.0
- This has CVE-2015-6420 (HIGH severity: 7.5)
- Pipeline should FAIL at security_scan
- Neither Docker Hub nor Railway will be updated

**This demonstrates the security gate in action!**

---

### Step 2: Add Vulnerable Dependency

Open `pom.xml` and add this **old, vulnerable version**:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- ADD THIS VULNERABLE DEPENDENCY -->
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-collections4</artifactId>
        <version>4.0</version>  <!-- OLD VULNERABLE VERSION! -->
    </dependency>
</dependencies>
```

**Why this version?**
- Commons Collections 4.0 has CVE-2015-6420
- HIGH severity (7.5/10)
- Well-known vulnerability — perfect for learning
- Doesn't conflict with Spring Boot dependencies

---

### Step 3: Commit and Push

```bash
git add pom.xml
git commit -m "Add vulnerable commons-collections4 for testing"
git push origin main
```

---

### Step 4: Watch Pipeline Fail

Go to CircleCI and watch the pipeline.

**This time, security_scan will FAIL!** ❌

**Pipeline flow:**
```
1. build          ✅ SUCCESS
2. test           ✅ SUCCESS
3. security_scan  ❌ FAILED (found HIGH severity!)
4. publish        ⏸️  SKIPPED (didn't run)
5. deploy         ⏸️  SKIPPED (didn't run)
```

**In security_scan logs:**

```
[ERROR] One or more dependencies were identified with known vulnerabilities:
[ERROR] 
[ERROR] commons-collections4-4.0.jar
[ERROR]     CVE-2015-6420 Severity: HIGH (7.5)
[ERROR] 
[ERROR] log4j-api-2.24.3.jar
[ERROR]     CVE-XXXX-XXXXX Severity: MEDIUM (5.5)
[ERROR] 
[INFO] BUILD FAILURE
```

**publish job never runs** — Docker image not built!
**deploy job never runs** — Railway not triggered!

**Vulnerable code is completely blocked from production!** ✅

---

### Step 5: View Detailed Report

Click **Artifacts** → **security-report/dependency-check-report.html**

**Summary:**
```
Dependencies Scanned: 48
Vulnerable Dependencies: 2
Critical: 0
High: 1      ← This caused the failure!
Medium: 1
Low: 0
```

**Vulnerability Details:**

```
commons-collections4-4.0.jar
└── CVE-2015-6420
    Severity: HIGH (7.5/10)
    
    Description:
    Serialized-object interfaces in Apache Commons Collections 
    use an insecure deserialization method that allows remote 
    code execution. Attackers can exploit this to execute 
    arbitrary code on the server.
    
    Affected Versions: All versions before 4.1
    Fixed Versions: 4.1, 4.2, 4.3, 4.4 and later
    Recommendation: Upgrade to commons-collections4 version 4.1 or later
```

---

## Part 5 - Fix the Vulnerability

Now let's fix the vulnerability by updating to a secure version.

### Step 1: Determine Safe Version

**From the report, we know:**
- Current version: 4.0 (vulnerable - HIGH severity)
- Recommended: 4.1 or later

Go to https://mvnrepository.com/artifact/org.apache.commons/commons-collections4 to check the latest stable version.

**Latest stable version:** 4.4

---

### Step 2: Update pom.xml

```xml
<!-- BEFORE (vulnerable) -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.0</version>  <!-- VULNERABLE - HIGH SEVERITY -->
</dependency>

<!-- AFTER (fixed) -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>  <!-- SECURE - NO KNOWN VULNERABILITIES -->
</dependency>
```

**That's it!** Just changing the version number fixes the vulnerability.

---

### Step 3: Test Locally (Optional but Recommended)

```bash
mvn clean install -DskipTests
```

**Expected output:**
```
[INFO] BUILD SUCCESS
```

---

### Step 4: Commit and Push Fix

```bash
git add pom.xml
git commit -m "Fix: Update commons-collections4 to secure version 4.4"
git push origin main
```

---

## Part 6 - Verify Fix Works

Watch the pipeline run again with the fix.

### Step 1: Monitor Pipeline in CircleCI

```
1. build          ✅ SUCCESS (10s)
2. test           ✅ SUCCESS (14s)
3. security_scan  ✅ SUCCESS (1m 30s - faster with cache!)
4. publish        ✅ SUCCESS (28s)
5. deploy         ✅ SUCCESS (48s)
```

**All jobs pass!** 🎉

---

### Step 2: Check Security Scan Logs

```
[INFO] Scanning: commons-collections4-4.4.jar
[INFO] Generating Report...
[INFO] Found 1 vulnerabilities (1 medium)
[INFO] BUILD SUCCESS
```

✅ **No HIGH or CRITICAL vulnerabilities found!**

---

### Step 3: View Updated Report

**Summary:**
```
Dependencies Scanned: 48
Vulnerable Dependencies: 1
Critical: 0
High: 0       ← Fixed!
Medium: 1     ← Acceptable (doesn't block deployment)
Low: 0
```

**Status: SECURE ENOUGH TO DEPLOY** ✅

---

### Step 4: Verify Docker Image Published

1. Go to https://hub.docker.com
2. Navigate to your repository: `YOUR_USERNAME/devops-demo`
3. Check **"Tags"** tab — should see a new `latest` push

---

### Step 5: Verify Railway Deployment

1. Go to Railway dashboard: https://railway.app
2. Click on your **devops-demo** service
3. Click **"Deployments"** tab — should see new deployment triggered by CircleCI

**Test the deployed application:**

```bash
curl https://your-railway-url.up.railway.app/hello
```

**Expected response:**
```
DevOps demo application is running!
```

**Success!** Secure code is now live on Railway! 🚀

---

### What Just Happened?

**Complete DevSecOps Flow:**

```
1. Developer updates dependency (fixed vulnerability)
   ↓
2. Commits to Git
   ↓
3. CircleCI builds application
   ↓
4. CircleCI runs tests
   ↓
5. CircleCI scans for vulnerabilities ← SECURITY GATE!
   ├─ Found 1 MEDIUM (acceptable)
   ├─ Found 0 HIGH
   ├─ Found 0 CRITICAL
   └─ Result: PASS ✅
   ↓
6. Build Docker image
   ↓
7. Push to Docker Hub
   ↓
8. Trigger Railway deployment
   ↓
9. Application live in production (secure!)
```

**This is Shift-Left Security in action!**

---

## Part 7 - Understanding the Security Report

### CVE Severity Scores

| Score | Severity | Action Required | Build Status |
|-------|----------|-----------------|--------------|
| 9.0-10.0 | **Critical** | Fix immediately! | FAILS ❌ |
| 7.0-8.9 | **High** | Fix within days | FAILS ❌ |
| 4.0-6.9 | **Medium** | Fix within weeks | PASSES ✅ |
| 0.1-3.9 | **Low** | Fix when convenient | PASSES ✅ |

**Your threshold:** `<failBuildOnCVSS>7</failBuildOnCVSS>`

---

### Example Vulnerability Entry

```
CVE-2015-6420
Severity: HIGH (7.5/10)
CVSS Vector: CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N

Description:
Serialized-object interfaces in Apache Commons Collections 
use an insecure deserialization method that allows remote code execution.

Affected Versions: All versions before 4.1
Fixed Versions: 4.1, 4.2, 4.3, 4.4 and later
Recommendation: Upgrade to commons-collections4 version 4.1 or later
```

---

### CVSS Vector Breakdown

**CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N**

- **AV:N** — Attack Vector: Network (exploitable remotely)
- **AC:L** — Attack Complexity: Low (easy to exploit)
- **PR:N** — Privileges Required: None (no authentication needed)
- **UI:N** — User Interaction: None (fully automated attack)
- **I:H** — Integrity: High (can execute malicious code)

**Translation:** Anyone on the internet can easily exploit this to run malicious code on your server.

---

### False Positives

**Sometimes the scanner reports false positives:**

**Why does this happen?**
- Scanner uses pattern matching on filenames
- May not recognize newer fix versions
- Scanner database may be slightly outdated

**What to do:**
1. Check CVE details manually on https://nvd.nist.gov/
2. Verify your actual version
3. Confirm if vulnerability applies
4. If confirmed false positive, suppress it and document why

**Don't suppress real vulnerabilities — only confirmed false positives!**

---

### Suppressing False Positives

Create `dependency-check-suppression.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
    <suppress>
        <notes>
            False positive - we're using version 1.0.1 which includes the fix.
            Verified at: https://nvd.nist.gov/vuln/detail/CVE-2023-12345
            Date: 2026-02-20
        </notes>
        <cve>CVE-2023-12345</cve>
    </suppress>
</suppressions>
```

**Then add to pom.xml plugin configuration:**

```xml
<suppressionFile>dependency-check-suppression.xml</suppressionFile>
```

---

## Best Practices for DevSecOps

**1. Run Security Scans on Every Build**
- Scan on every branch, not just main
- Catch vulnerabilities in feature branches before merging

**2. Set Appropriate Failure Thresholds**
```xml
<failBuildOnCVSS>7</failBuildOnCVSS>  <!-- Recommended: HIGH and above -->
```

**3. Keep Dependencies Updated Proactively**
```bash
mvn versions:display-dependency-updates
```

**4. Review Reports Regularly**
- Even when builds pass, check MEDIUM/LOW issues
- Track security debt over time
- Share reports with team

**5. Automate Everything**
- Scan on every commit
- Generate reports automatically
- Block vulnerable code from reaching production

---

## Summary

### What You Accomplished Today

1. ✅ Obtained NVD API key for security scanning
2. ✅ Configured OWASP Dependency-Check plugin in pom.xml
3. ✅ Added security_scan job to CircleCI pipeline
4. ✅ Ran first security scan (found MEDIUM log4j vulnerability)
5. ✅ Intentionally added HIGH severity vulnerability
6. ✅ Saw pipeline fail when HIGH vulnerability detected
7. ✅ Read and interpreted detailed security reports
8. ✅ Fixed vulnerability by updating dependency version
9. ✅ Verified pipeline passes with secure code
10. ✅ Confirmed secure code deployed to Docker Hub AND Railway

---

### Your Complete Pipeline Now

```
Build → Test → Security Scan → Publish → Deploy to Railway
```

**Before DevSecOps (Lessons 4.7-4.12):**
```
✅ Automated build, tests, publish, deploy
❌ No security checks
❌ Vulnerable code could reach production
```

**After DevSecOps (Lesson 4.14):**
```
✅ Automated build, tests, publish, deploy
✅ Automated security scanning ← NEW!
✅ Vulnerabilities caught in minutes
✅ Production stays secure
✅ Compliance-ready audit trail
```

---

## Optional: Remove Test Vulnerability

If you added commons-collections4 just for learning and don't actually use it:

```xml
<!-- REMOVE THIS IF NOT NEEDED -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>
</dependency>
```

```bash
git add pom.xml
git commit -m "Remove test dependency (not needed for application)"
git push origin main
```

---

## Troubleshooting Guide

### Issue 0: "NullPointerException: connectionPool is null"

**Cause:** No NVD API key configured

**Solution:**
1. Get free API key: https://nvd.nist.gov/developers/request-an-api-key
2. Add to CircleCI Environment Variables: `NVD_API_KEY`
3. Verify pom.xml has: `<nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>`

---

### Issue 1: "401 Unauthorized - ossindex.sonatype.org"

**Solution:**
Add to pom.xml plugin configuration:
```xml
<ossindexAnalyzerEnabled>false</ossindexAnalyzerEnabled>
```

---

### Issue 2: Security scan takes 20+ minutes

**Solution:**
1. Verify `NVD_API_KEY` is set in CircleCI Environment Variables
2. Verify pom.xml has `<nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>`
3. Add timeout if persistent:
```yml
- run:
    name: Run Dependency Check
    command: mvn org.owasp:dependency-check-maven:check
    no_output_timeout: 30m
```

---

### Issue 3: Build passes but vulnerabilities found

**Cause:** Vulnerabilities are MEDIUM or LOW — below threshold of 7.0

**Expected:** This is correct behavior. To change:
```xml
<failBuildOnCVSS>4</failBuildOnCVSS>  <!-- Fail on MEDIUM+ -->
```

---

### Issue 4: Can't access security report

**Solution:**
Verify in `.circleci/config.yml`:
```yml
- store_artifacts:
    path: target/dependency-check-report.html
    destination: security-report
```

---

### Issue 5: Too many false positives

**Solutions:**
1. Update to latest dependency versions: `mvn versions:use-latest-releases`
2. Suppress confirmed false positives with `dependency-check-suppression.xml`
3. Verify each CVE manually at https://nvd.nist.gov/

---

### Issue 6: "NVD API Key request timed out"

**Solutions:**
1. Wait 10-15 minutes and retry
2. Check email — key may have been sent despite timeout
3. Try incognito/private browsing mode
4. Continue without key (scans will be slower at 10-20 minutes)

---

## Additional Resources

- [OWASP Dependency-Check](https://jeremylong.github.io/DependencyCheck/)
- [Maven Plugin Configuration](https://jeremylong.github.io/DependencyCheck/dependency-check-maven/configuration.html)
- [Suppression File Guide](https://jeremylong.github.io/DependencyCheck/general/suppression.html)
- [NVD API Documentation](https://nvd.nist.gov/developers)
- [National Vulnerability Database](https://nvd.nist.gov/)
- [CVSS Calculator](https://www.first.org/cvss/calculator/3.1)
- [Snyk Vulnerability Database](https://security.snyk.io/)
- [CircleCI Security Best Practices](https://circleci.com/docs/security/)
- [Versions Maven Plugin](https://www.mojohaus.org/versions/versions-maven-plugin/)