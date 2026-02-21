# Lesson 4.14: DevSecOps Practicals

## Learning Objectives

By the end of this lesson, you will be able to:

1. **Add** security scanning to an existing CI/CD pipeline
2. **Configure** OWASP Dependency-Check in CircleCI
3. **Interpret** vulnerability reports and risk scores
4. **Update** vulnerable dependencies to secure versions
5. **Verify** that security fixes resolve vulnerabilities

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

## Pre-Class Setup: Obtain NVD API Key (15 minutes)

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

In Lesson 4.13, you learned the theory behind DevSecOps - why security matters, how shift-left works, and the risks in CI/CD pipelines.

Today, you'll put that knowledge into practice. You'll add **automated security scanning** to your CircleCI pipeline so that every code commit is checked for vulnerabilities before deployment.

By the end of this lesson, your pipeline will automatically:
- Scan all dependencies for known vulnerabilities
- Generate detailed security reports
- Block deployment if critical vulnerabilities are found
- Ensure only secure code reaches Railway production

**This is real-world DevSecOps in action!**

---

## Part 1 - Review Current Pipeline (15 minutes)

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

## Part 2 - Add Security Scanning (50 minutes)

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
  - Keeps configuration simple for students

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
      
      # Restore Maven cache
      - restore_cache:
          keys:
            - maven-deps-{{ checksum "pom.xml" }}
            - maven-deps-
      
      # Run OWASP Dependency-Check
      - run:
          name: Run Dependency Check
          command: |
            echo "Running security scan..."
            mvn org.owasp:dependency-check-maven:check
      
      # Store the HTML report as artifact
      - store_artifacts:
          path: target/dependency-check-report.html
          destination: security-report
      
      # Store XML report for CircleCI
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
  # Build job (unchanged)
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

  # Test job (unchanged)
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

  # NEW: Security scan job
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

  # Publish job (unchanged)
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

  # Deploy job (unchanged)
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

# Updated workflow with security scan
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

**In your terminal:**

```bash
# Navigate to your project
cd path/to/devops-demo

# Check what changed
git status

# Add the updated config
git add .circleci/config.yml

# Commit with descriptive message
git commit -m "Add security_scan job to CI/CD pipeline"

# Push to trigger pipeline
git push origin main
```

**Expected output:**
```
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 8 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 456 bytes | 456.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0)
To https://github.com/YOUR_USERNAME/devops-demo.git
   abc1234..def5678  main -> main
```

---

## Part 3 - Run Pipeline with Security Scan (30 minutes)

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
- Compiles Java code
- Packages JAR file

**test job:** ~10-15 seconds
- Runs JUnit tests
- Stores test results

**security_scan job:** ~4-6 minutes (first run with API key)
- Downloads NVD database (~3 minutes first time)
- Processes CVE records (~30 seconds)
- Analyzes dependencies (~1 minute)
- Generates HTML report (~30 seconds)
- **Subsequent runs:** ~1-2 minutes (uses cached database)

**publish job:** ~30 seconds (if security passes)
- Builds Docker image
- Pushes to Docker Hub

**deploy job:** ~45 seconds (if publish succeeds)
- Triggers Railway deployment via CLI

**Total pipeline time (first run):** ~6-7 minutes

---

### Step 3: Understanding Security Scan Output

**In CircleCI, click on the `security_scan` job to see logs:**

**Expected output:**

```
#!/bin/bash -eo pipefail
echo "Running security scan..."
mvn org.owasp:dependency-check-maven:check

Running security scan...
[INFO] Scanning for projects...
[INFO] 
[INFO] -------------------< com.example:devops-demo >-------------------
[INFO] Building devops-demo 0.0.1-SNAPSHOT
[INFO] 
[INFO] --- dependency-check-maven:12.1.0:check (default-cli) @ devops-demo ---
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
[INFO]   - (and more...)
[INFO] 
[INFO] Analyzing Dependencies...
[INFO] 
[INFO] Generating Report...
[INFO] Found 1 vulnerabilities
[INFO] 
[INFO] BUILD SUCCESS
```

**Wait - it found a vulnerability but passed?** Yes! This is expected - read on!

---

### Step 4: Access the Security Report

**View the report:**

1. In CircleCI, go to **Artifacts** tab
2. Click **security-report/dependency-check-report.html**
3. Report opens in new tab

**Report sections:**

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

**Dependency List:**
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

---

**This is actually GOOD PRACTICE because:**

1. **Not all vulnerabilities are equal**
   - MEDIUM issues may not be exploitable in your context
   - Gives you time to plan updates
   - Doesn't block critical deployments

2. **Prevents alert fatigue**
   - Too many false alarms → teams ignore warnings
   - Focus on HIGH/CRITICAL first
   - Address MEDIUM in regular maintenance

3. **Allows flexible security**
   - HIGH/CRITICAL = immediate fix required
   - MEDIUM = fix in next sprint
   - LOW = fix when convenient

---

### Step 6: Verify the MEDIUM Vulnerability

**Click on the log4j-api vulnerability in the report:**

```
log4j-api-2.24.3.jar
├── Package: pkg:maven/org.apache.logging.log4j:log4j-api@2.24.3
├── Severity: MEDIUM (5.5/10)
├── CVE Count: 1
├── Confidence: Highest
└── Evidence Count: 41
```

**Why is this here?**
- Spring Boot includes log4j as a **transitive dependency**
- This is a real vulnerability in a production library
- Shows your scanner is working correctly!

**Important learning points:**
- Real-world applications always have some vulnerabilities
- Scanning tools find them immediately
- Thresholds let you manage risk intelligently
- This is normal DevSecOps behavior

---

## Part 4 - Add HIGH Severity Vulnerability (25 minutes)

Now let's intentionally add a HIGH severity vulnerability to see the pipeline actually FAIL.

### Step 1: Before Adding Vulnerable Dependency

**Current state:**
- ✅ 1 MEDIUM vulnerability (log4j) - Build passes
- ✅ Pipeline deployed to Railway

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
    <!-- Existing dependencies -->
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
- Well-known vulnerability - perfect for learning
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
[INFO] Analyzing Dependencies...
[INFO] 
[INFO] Generating Report...
[INFO] 
[ERROR] 
[ERROR] One or more dependencies were identified with known vulnerabilities:
[ERROR] 
[ERROR] commons-collections4-4.0.jar (pkg:maven/org.apache.commons/commons-collections4@4.0)
[ERROR]     CVE-2015-6420 Severity: HIGH (7.5)
[ERROR] 
[ERROR] log4j-api-2.24.3.jar (pkg:maven/org.apache.logging.log4j:log4j-api@2.24.3)
[ERROR]     CVE-XXXX-XXXXX Severity: MEDIUM (5.5)
[ERROR] 
[ERROR] See the dependency-check report for more details.
[ERROR] 
[INFO] BUILD FAILURE
```

**Pipeline stops at security_scan!** ✅

**publish job never runs** - Docker image not built!
**deploy job never runs** - Railway not triggered!

**Vulnerable code is completely blocked from production!**

---

### Step 5: View Detailed Report

Click **Artifacts** → **security-report/dependency-check-report.html**

**Report shows:**

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
    CVSS Vector: CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N
    
    Description:
    Serialized-object interfaces in Apache Commons Collections 
    use an insecure deserialization method that allows remote 
    code execution. Attackers can exploit this to execute 
    arbitrary code on the server.
    
    Affected Versions: 
    All versions before 4.1
    
    Fixed Versions:
    4.1, 4.2, 4.3, 4.4 and later
    
    Recommendation:
    Upgrade to commons-collections4 version 4.1 or later
```

**This is a serious vulnerability!** The pipeline correctly blocked it from reaching production.

---

## Part 5 - Fix the Vulnerability (30 minutes)

Now let's fix the vulnerability by updating to a secure version.

### Step 1: Determine Safe Version

**From the report, we know:**
- Current version: 4.0 (vulnerable - HIGH severity)
- Recommended: 4.1 or later

**Let's check the latest version:**

Go to https://mvnrepository.com/artifact/org.apache.commons/commons-collections4

**Latest stable version:** 4.4 (or whatever is current when you check)

---

### Step 2: Update pom.xml

**Change the version:**

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

**Verify it builds:**

```bash
mvn clean install -DskipTests
```

**Expected output:**
```
[INFO] BUILD SUCCESS
[INFO] Total time: 5.234 s
```

**Run dependency check locally (if you want to verify before pushing):**

```bash
mvn org.owasp:dependency-check-maven:check
```

**Expected output:**
```
[INFO] Analyzing Dependencies...
[INFO] 
[INFO] Generating Report...
[INFO] Found 1 vulnerabilities (1 medium)
[INFO] 
[INFO] BUILD SUCCESS
```

✅ **Fixed!** Only the MEDIUM log4j issue remains (which doesn't fail the build).

---

### Step 4: Commit and Push Fix

```bash
git add pom.xml
git commit -m "Fix: Update commons-collections4 to secure version 4.4"
git push origin main
```

---

## Part 6 - Verify Fix Works (20 minutes)

Watch the pipeline run again with the fix.

### Step 1: Monitor Pipeline in CircleCI

**Pipeline runs again:**

```
1. build          ✅ SUCCESS (10s)
2. test           ✅ SUCCESS (14s)
3. security_scan  ✅ SUCCESS (1m 30s - faster with cache!)
4. publish        ✅ SUCCESS (28s - image built and pushed)
5. deploy         ✅ SUCCESS (48s - Railway deployment triggered)
```

**This time, all jobs pass!** 🎉

**Notice:** Security scan was much faster (1-2 minutes) because it used the cached NVD database!

---

### Step 2: Check Security Scan Logs

**In security_scan job:**

```
[INFO] Analyzing Dependencies...
[INFO] 
[INFO] Scanning: commons-collections4-4.4.jar
[INFO] 
[INFO] Generating Report...
[INFO] Found 1 vulnerabilities (1 medium)
[INFO] 
[INFO] BUILD SUCCESS
```

✅ **No HIGH or CRITICAL vulnerabilities found!**

The MEDIUM log4j issue is still there, but that's expected and acceptable.

---

### Step 3: View Updated Report

**Artifacts** → **security-report/dependency-check-report.html**

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
3. Click **"Tags"** tab
4. You should see a new push with tag `latest`
5. Check timestamp - should match your recent commit time

**Success!** Secure code made it to Docker Hub! 🎉

---

### Step 5: Verify Railway Deployment

1. Go to Railway dashboard: https://railway.app
2. Click on your **devops-demo** project
3. Click on your **devops-demo** service
4. Click **"Deployments"** tab
5. You should see a new deployment triggered by CircleCI
6. Status should be "Success" or "Active"

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
   ├─ Found 0 HIGH (good!)
   ├─ Found 0 CRITICAL (good!)
   └─ Result: PASS ✅
   ↓
6. Build Docker image
   ↓
7. Push to Docker Hub
   ↓
8. Trigger Railway deployment
   ↓
9. Railway pulls code and deploys
   ↓
10. Application live in production (secure!)
```

**If HIGH/CRITICAL vulnerabilities had been found at step 5:**
- Pipeline stops immediately ❌
- Developer gets notification
- No Docker image built
- No Railway deployment triggered
- No vulnerable code reaches production
- Must fix vulnerability before deploying

**This is Shift-Left Security in action!**

---

## Part 7 - Understanding the Security Report (15 minutes)

Let's dive deeper into interpreting security reports.

### CVE Severity Scores

**CVSS (Common Vulnerability Scoring System):**

| Score | Severity | Action Required | Build Status |
|-------|----------|-----------------|--------------|
| 9.0-10.0 | **Critical** | Fix immediately! | FAILS ❌ |
| 7.0-8.9 | **High** | Fix within days | FAILS ❌ |
| 4.0-6.9 | **Medium** | Fix within weeks | PASSES ✅ |
| 0.1-3.9 | **Low** | Fix when convenient | PASSES ✅ |
| 0.0 | **None** | No action needed | PASSES ✅ |

**Your threshold:** `<failBuildOnCVSS>7</failBuildOnCVSS>`
- Fails on HIGH (7.0+) or CRITICAL (9.0+)
- Allows MEDIUM (4.0-6.9) and below

---

### Example Vulnerability Entry

```
CVE-2015-6420
Severity: HIGH (7.5/10)
CVSS Vector: CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N

Description:
Serialized-object interfaces in Apache Commons Collections 
use an insecure deserialization method that allows remote 
code execution.

Affected Versions: 
All versions before 4.1

Fixed Versions:
4.1, 4.2, 4.3, 4.4 and later

Recommendation:
Upgrade to commons-collections4 version 4.1 or later
```

**What this tells you:**

1. **CVE ID:** Unique identifier for tracking (CVE-2015-6420)
2. **Score:** 7.5 = HIGH priority = Build fails
3. **CVSS Vector:** Technical details about exploit difficulty
4. **Description:** What the vulnerability allows (remote code execution!)
5. **Affected Versions:** Which versions are vulnerable (< 4.1)
6. **Fix:** Update to safe version (4.1+)

---

### CVSS Vector Breakdown

**CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N**

Decoded:
- **AV:N** (Attack Vector: Network) - Can be exploited remotely over internet
- **AC:L** (Attack Complexity: Low) - Easy to exploit
- **PR:N** (Privileges Required: None) - No authentication needed
- **UI:N** (User Interaction: None) - Fully automated attack
- **S:U** (Scope: Unchanged) - Affects only vulnerable component
- **C:N** (Confidentiality: None) - Doesn't expose data
- **I:H** (Integrity: High) - Can modify/execute code
- **A:N** (Availability: None) - Doesn't cause service disruption

**Translation:** Anyone on the internet can easily exploit this to run malicious code on your server. Very dangerous!

---

### False Positives

**Sometimes the scanner reports false positives:**

**Example:**
```
Vulnerability: CVE-2023-12345
Affected: my-library-1.0.0
But: You're using 1.0.1 which fixes it!
```

**Why does this happen?**
- Scanner uses pattern matching on filenames
- May not recognize newer fix versions
- Dependency naming changes between versions
- Scanner database may be slightly outdated

**What to do:**
1. Check CVE details manually on https://nvd.nist.gov/
2. Verify your actual version
3. Confirm if vulnerability applies
4. If false positive, suppress it (see below)
5. Document why you suppressed it

**Don't suppress real vulnerabilities - only confirmed false positives!**

---

### Suppressing False Positives

If you have a confirmed false positive, create `dependency-check-suppression.xml`:

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
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>12.1.0</version>
    <configuration>
        <nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <ossindexAnalyzerEnabled>false</ossindexAnalyzerEnabled>
        <!-- Add suppression file -->
        <suppressionFile>dependency-check-suppression.xml</suppressionFile>
    </configuration>
</plugin>
```

**Use suppressions sparingly!** Only for confirmed false positives, never to ignore real vulnerabilities.

---

## Best Practices for DevSecOps (10 minutes)

### 1. Run Security Scans on Every Build

**Don't:**
```yml
# Bad - only scan on main branch
- security_scan:
    filters:
      branches:
        only: main
```

**Do:**
```yml
# Good - scan on every branch
- security_scan:
    requires:
      - test
```

**Why:** Catch vulnerabilities in feature branches before merging to main.

---

### 2. Set Appropriate Failure Thresholds

**Configure based on your risk tolerance:**

```xml
<!-- Strict - Fail on MEDIUM and above -->
<failBuildOnCVSS>4</failBuildOnCVSS>

<!-- Balanced - Fail on HIGH and above (recommended) -->
<failBuildOnCVSS>7</failBuildOnCVSS>

<!-- Relaxed - Fail only on CRITICAL -->
<failBuildOnCVSS>9</failBuildOnCVSS>
```

**Recommendation:** Start with `7` (HIGH+) and adjust based on your organization's security posture.

---

### 3. Keep Dependencies Updated Proactively

**Don't wait for vulnerabilities!**

```bash
# Check for updates weekly
mvn versions:display-dependency-updates
```

**Benefits:**
- Proactive updates = easier, planned work
- Reactive fixes = stressful emergency deployments
- Latest versions often have performance improvements
- Staying current prevents "upgrade debt"

---

### 4. Review Reports Regularly

**Even when builds pass:**
- Check for new MEDIUM/LOW severity issues
- Plan updates in upcoming sprints
- Track security debt over time
- Share reports with team in standups

**Set up a weekly routine:**
- Monday: Check security scan results
- Plan: Schedule updates in next sprint
- Track: Maintain security backlog
- Communicate: Share findings with team

---

### 5. Automate Everything

**Your pipeline should:**
- ✅ Scan on every commit
- ✅ Generate reports automatically
- ✅ Fail builds on HIGH/CRITICAL issues
- ✅ Store reports as artifacts
- ✅ Block vulnerable code from Docker Hub
- ✅ Block vulnerable code from Railway production

**Benefits:**
- No manual security reviews needed
- Consistent enforcement
- Fast feedback (minutes, not days)
- Audit trail for compliance

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

### Key Takeaways

**DevSecOps in Action:**
- Security checks run automatically on every commit
- Vulnerabilities caught immediately (shift-left)
- No vulnerable code reaches production
- Prevents expensive late-stage fixes

**Your Complete Pipeline Now:**
```
Build → Test → Security Scan → Publish → Deploy to Railway
```

**Security as a Gate:**
- HIGH/CRITICAL vulnerabilities block everything downstream
- MEDIUM vulnerabilities generate warnings but allow deployment
- Forces immediate fixes for serious issues
- Creates secure-by-default culture

**Continuous Security:**
- Every commit is scanned
- Database updates regularly (every 4 hours by default)
- Always know your security status
- Audit trail for compliance

---

### Before DevSecOps vs After

**Before (Lessons 4.7-4.12):**
```
✅ Automated build
✅ Automated tests
✅ Automated Docker publishing
✅ Automated Railway deployment
❌ No security checks
❌ Vulnerable code could reach production
❌ Security issues discovered too late (or never!)
```

**After (Lesson 4.14):**
```
✅ Automated build
✅ Automated tests
✅ Automated security scanning ← NEW!
✅ Automated Docker publishing (only if secure)
✅ Automated Railway deployment (only if secure)
✅ Vulnerabilities caught in minutes
✅ Production stays secure
✅ Compliance-ready audit trail
```

---

### Real-World Impact

**Without DevSecOps:**
- Average time to fix vulnerability: 38 days
- Cost of production security fix: $1,000 - $10,000
- Risk of data breach: HIGH
- Regulatory compliance: DIFFICULT

**With DevSecOps:**
- Average time to fix vulnerability: 1-2 days
- Cost of development security fix: $100 - $500
- Risk of data breach: LOW
- Regulatory compliance: AUTOMATED

**Your pipeline now catches issues in minutes, not months!**

---

## Troubleshooting Guide

### Issue 0: "NullPointerException: connectionPool is null"

**Symptoms:**
```
java.lang.NullPointerException: Cannot invoke 
"org.apache.commons.dbcp2.BasicDataSource.getConnection()"
```

**Cause:** No NVD API key configured

**Solution:**
1. Get free API key: https://nvd.nist.gov/developers/request-an-api-key
2. Add to CircleCI Environment Variables: `NVD_API_KEY`
3. Verify pom.xml has: `<nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>`
4. Push changes and retry

**Prevention:** Complete Pre-Class Setup before starting lesson

---

### Issue 1: "401 Unauthorized - ossindex.sonatype.org"

**Symptoms:**
```
AnalysisException: Failed to request component-reports
DownloadFailedException: https://ossindex.sonatype.org/api/v3/component-report
Server status: 401 - Server reason: Unauthorized
```

**Cause:** OSS Index analyzer not disabled (requires separate credentials)

**Solution:**
Add to pom.xml plugin configuration:
```xml
<ossindexAnalyzerEnabled>false</ossindexAnalyzerEnabled>
```

Commit and push changes. NVD alone is sufficient for DevSecOps.

---

### Issue 2: Security scan takes 20+ minutes

**Symptoms:** First security_scan job runs for 20-30 minutes

**Causes:**
- No NVD API key configured (rate limited)
- Network issues downloading database

**Solutions:**

**Option 1: Verify API key is configured**
1. Check CircleCI Environment Variables for `NVD_API_KEY`
2. Check pom.xml has `<nvdApiKey>${env.NVD_API_KEY}</nvdApiKey>`
3. Restart build

**Option 2: Add timeout (if persistent)**
```yml
- run:
    name: Run Dependency Check
    command: mvn org.owasp:dependency-check-maven:check
    no_output_timeout: 30m
```

**Note:** First run with API key should be 4-6 minutes. Subsequent runs: 1-2 minutes.

---

### Issue 3: Build passes but vulnerabilities found

**Symptoms:** Report shows vulnerabilities but build status is SUCCESS

**Cause:** Vulnerabilities are below threshold (MEDIUM or LOW severity)

**Expected:** This is correct behavior!
- Threshold is 7.0 (HIGH+)
- MEDIUM (4.0-6.9) doesn't fail build
- Generates warnings but allows deployment

**To change behavior:**
Adjust threshold in pom.xml:
```xml
<!-- Fail on MEDIUM+ -->
<failBuildOnCVSS>4</failBuildOnCVSS>

<!-- Fail on HIGH+ (recommended) -->
<failBuildOnCVSS>7</failBuildOnCVSS>

<!-- Fail only on CRITICAL -->
<failBuildOnCVSS>9</failBuildOnCVSS>
```

---

### Issue 4: Can't access security report

**Symptoms:** Artifacts tab shows no security-report

**Cause:** Report not stored as artifact

**Solution:**
Verify in `.circleci/config.yml`:
```yml
- store_artifacts:
    path: target/dependency-check-report.html
    destination: security-report
```

Check report was generated:
```yml
- run:
    name: Run Dependency Check
    command: mvn org.owasp:dependency-check-maven:check
```

---

### Issue 5: Too many false positives

**Symptoms:** Many vulnerabilities reported that don't actually apply

**Causes:**
- Scanner pattern matching is aggressive
- Dependency naming changes
- Database slightly outdated

**Solutions:**

**Option 1: Update to latest dependency versions**
```bash
mvn versions:display-dependency-updates
mvn versions:use-latest-releases
```

**Option 2: Suppress confirmed false positives**
1. Verify each CVE manually at https://nvd.nist.gov/
2. Create `dependency-check-suppression.xml`
3. Add to pom.xml configuration
4. Document why suppressed

**Option 3: Consider alternative tools**
- Snyk (https://snyk.io/) - more accurate, requires account
- GitHub Dependabot - built-in, automated PRs
- Trivy - fast, containerized scanning

---

### Issue 6: "NVD API Key request timed out"

**Symptoms:** NVD website shows "Validation timed out. Administrators have been notified."

**Cause:** NVD website overload or server issues

**Solutions:**

**Option 1: Wait and retry**
- Wait 10-15 minutes
- Try again - server may recover

**Option 2: Check email**
- Key may have been sent despite timeout
- Check spam folder
- Look for email from noreply@nist.gov

**Option 3: Try different browser**
- Clear cache
- Use incognito/private mode
- Try different browser

**Option 4: Continue without key**
- Pipeline will work but slower (10-20 minutes)
- Get key later and add to CircleCI
- Update lesson accordingly

---

## Additional Resources

### Official Documentation
- [OWASP Dependency-Check](https://jeremylong.github.io/DependencyCheck/)
- [Maven Plugin Configuration](https://jeremylong.github.io/DependencyCheck/dependency-check-maven/configuration.html)
- [Suppression File Guide](https://jeremylong.github.io/DependencyCheck/general/suppression.html)
- [NVD API Documentation](https://nvd.nist.gov/developers)

### CVE Information
- [National Vulnerability Database](https://nvd.nist.gov/)
- [CVE Details](https://www.cvedetails.com/)
- [CVSS Calculator](https://www.first.org/cvss/calculator/3.1)
- [Snyk Vulnerability Database](https://security.snyk.io/)

### CircleCI Resources
- [CircleCI Security Best Practices](https://circleci.com/docs/security/)
- [Environment Variables](https://circleci.com/docs/env-vars/)
- [Storing Artifacts](https://circleci.com/docs/artifacts/)
- [Using Orbs for Security](https://circleci.com/developer/orbs)

### Maven Security
- [Maven Security Scanning Guide](https://maven.apache.org/guides/mini/guide-maven-security.html)
- [Dependency Management](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)
- [Versions Maven Plugin](https://www.mojohaus.org/versions/versions-maven-plugin/)

### Videos & Tutorials
- [DevSecOps Pipeline Tutorial](https://www.youtube.com/results?search_query=devsecops+pipeline+tutorial)
- [OWASP Dependency Check Demo](https://www.youtube.com/results?search_query=owasp+dependency+check)
- [Fixing Vulnerabilities in CI/CD](https://www.youtube.com/results?search_query=fixing+vulnerabilities+cicd)
- [Understanding CVSS Scores](https://www.youtube.com/results?search_query=cvss+score+explained)

---

## Next Steps

### Continue Learning

**1. Explore Other Security Tools:**
- Try Snyk for comparison (more user-friendly, requires account)
- Experiment with Trivy for container scanning
- Add SAST (static application security testing)
- Explore DAST (dynamic application security testing)

**2. Advanced Configuration:**
- Set custom failure thresholds per project
- Configure email notifications for vulnerabilities
- Add Slack webhooks for security alerts
- Create security dashboards

**3. Monitoring & Metrics:**
- Track security metrics over time
- Create vulnerability trend reports
- Monitor dependency health scores
- Set up security KPIs

**4. Team Practices:**
- Establish security review process
- Create vulnerability response plan
- Share security reports in standups
- Train team on security awareness

---

## Optional: Remove Test Vulnerability

If you added commons-collections4 just for learning and don't actually use it in your application, you can remove it now:

**Edit pom.xml:**

```xml
<!-- REMOVE THIS IF NOT NEEDED -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>
</dependency>
```

**Why consider removing?**
- Only add dependencies you actually use
- Fewer dependencies = smaller attack surface
- Smaller Docker images
- Faster builds
- Cleaner pom.xml

**Commit:**
```bash
git add pom.xml
git commit -m "Remove test dependency (not needed for application)"
git push origin main
```

**Result:** Next security scan will show only the MEDIUM log4j vulnerability (which is acceptable).

---

**End of Lesson 4.14**

**Congratulations!** 🎉 You've successfully implemented complete DevSecOps practices in your CI/CD pipeline!

**Your application is now protected by:**
- ✅ Automated security scanning on every commit
- ✅ Vulnerability detection before deployment
- ✅ Security gate that blocks HIGH/CRITICAL issues
- ✅ Comprehensive security reports
- ✅ Secure code delivery to production

**This is a critical skill for modern software development!**

**You've completed the entire DevSecOps module!**

**Your pipeline now:**
- ✅ Builds automatically
- ✅ Tests automatically
- ✅ Scans for security vulnerabilities
- ✅ Publishes Docker images (only if secure)
- ✅ Deploys to Railway (only if secure)

**You are now equipped with real-world DevSecOps skills that companies actively seek!** 

**Well done!** 🚀