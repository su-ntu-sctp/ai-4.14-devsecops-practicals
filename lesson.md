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
- Your **devops-demo** project with CircleCI configured
- CircleCI account with project connected
- Access to your GitHub repository

---

## Introduction

In Lesson 4.13, you learned the theory behind DevSecOps - why security matters, how shift-left works, and the risks in CI/CD pipelines.

Today, you'll put that knowledge into practice. You'll add **automated security scanning** to your CircleCI pipeline so that every code commit is checked for vulnerabilities before deployment.

By the end of this lesson, your pipeline will automatically:
- Scan all dependencies for known vulnerabilities
- Generate detailed security reports
- Block deployment if critical vulnerabilities are found

**This is real-world DevSecOps in action!**

---

## Part 1 - Review Current Pipeline (15 minutes)

Before adding security, let's review what your pipeline currently does.

### Step 1: Check Your Current CircleCI Config

Open your **devops-demo** project and look at `.circleci/config.yml`

**Your current pipeline (from Lesson 4.7):**

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
      - run:
          name: Build
          command: mvn clean install -DskipTests
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
      - run:
          name: Test
          command: mvn test
      - store_test_results:
          path: target/surefire-reports

  publish:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - attach_workspace:
          at: .
      - setup_remote_docker:
          version: 20.10.14
      - run:
          name: Build Docker Image
          command: docker build -t $DOCKER_USERNAME/devops-demo:latest .
      - run:
          name: Push to Docker Hub
          command: |
            echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
            docker push $DOCKER_USERNAME/devops-demo:latest

workflows:
  build_test_publish:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          requires:
            - test
```

**Current flow:**
```
Build → Test → Publish
```

**What's missing? SECURITY!**

---

### Step 2: Understand What We'll Add

**New flow:**
```
Build → Test → Security Scan → Publish
```

**The security scan will:**
1. Check all dependencies in `pom.xml`
2. Match them against CVE database
3. Generate a vulnerability report
4. Fail the build if critical vulnerabilities found
5. Prevent vulnerable code from reaching Docker Hub

---

## Part 2 - Add Security Scanning Job (40 minutes)

Now let's add the security scanning job to your pipeline.

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

### Step 2: Add Security Scan Job to config.yml

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
      
      # Store for CircleCI to display
      - store_test_results:
          path: target/dependency-check-report.xml
```

---

### Step 3: Update Workflow to Include Security

Update the `workflows` section at the bottom of your config:

**Before:**
```yml
workflows:
  build_test_publish:
    jobs:
      - build
      - test:
          requires:
            - build
      - publish:
          requires:
            - test
```

**After (add security_scan):**
```yml
workflows:
  build_test_publish:
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
```

**New flow:**
```
build → test → security_scan → publish
                     ↓
              If vulnerabilities found,
              pipeline STOPS here!
```

---

### Step 4: Complete Updated config.yml

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
      - run:
          name: Build
          command: mvn clean install -DskipTests
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
      - run:
          name: Test
          command: mvn test
      - store_test_results:
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
      - setup_remote_docker:
          version: 20.10.14
      - run:
          name: Build Docker Image
          command: docker build -t $DOCKER_USERNAME/devops-demo:latest .
      - run:
          name: Push to Docker Hub
          command: |
            echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
            docker push $DOCKER_USERNAME/devops-demo:latest

# Updated workflow with security scan
workflows:
  build_test_publish:
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
```

---

### Step 5: Commit and Push Changes

**In your terminal:**

```bash
# Navigate to your project
cd path/to/devops-demo

# Check what changed
git status

# Add the updated config
git add .circleci/config.yml

# Commit with descriptive message
git commit -m "Add OWASP Dependency-Check to CI pipeline"

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
1. build      (running...)
2. test       (waiting...)
3. security_scan (waiting...)
4. publish    (waiting...)
```

---

### Step 2: Watch Security Scan Execute

**Timeline:**

**build job:** ~2 minutes
- Compiles code
- Packages JAR

**test job:** ~1 minute
- Runs JUnit tests

**security_scan job:** ~5-8 minutes (first run is slower!)
- Downloads NVD database (~4 minutes first time)
- Scans all dependencies (~1-2 minutes)
- Generates report (~1 minute)

**Why is it slow the first time?**
- Must download entire CVE database (500+ MB)
- Subsequent runs use cached database (much faster)

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
[INFO] --- dependency-check-maven:8.4.0:check (default-cli) @ devops-demo ---
[INFO] Checking for updates
[INFO] Download Started for NVD CVE - Modified
[INFO] Download Complete for NVD CVE - Modified  (4523 ms)
[INFO] Processing Started for NVD CVE - Modified
[INFO] Processing Complete for NVD CVE - Modified  (1234 ms)
[INFO] 
[INFO] Scanning Dependencies:
[INFO]   - spring-boot-starter-web-3.2.0.jar
[INFO]   - spring-boot-starter-test-3.2.0.jar
[INFO]   - (and more...)
[INFO] 
[INFO] Analyzing Dependencies...
[INFO] 
[INFO] Generating Report...
[INFO] Found 0 vulnerabilities
[INFO] 
[INFO] BUILD SUCCESS
```

**If no vulnerabilities:** ✅ Pipeline continues to publish
**If vulnerabilities found:** ❌ Pipeline fails, publish doesn't run

---

### Step 4: Access the Security Report

**View the report:**

1. In CircleCI, go to **Artifacts** tab
2. Click **security-report/dependency-check-report.html**
3. Report opens in new tab

**Report sections:**

**Summary:**
```
Scan Date: 2026-02-02
Total Dependencies: 47
Vulnerable Dependencies: 0
```

**Dependency List:**
- All dependencies with versions
- CVE information (if any)
- Risk scores

---

### What If No Vulnerabilities?

**Good news!** Your dependencies are currently secure.

**But let's learn by intentionally adding a vulnerable dependency...**

---

## Part 4 - Intentionally Add a Vulnerable Dependency (25 minutes)

To demonstrate how security scanning works, we'll temporarily add an old, vulnerable version of a library.

### Step 1: Add Vulnerable Dependency

Open `pom.xml` and add this **old version** of Jackson:

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
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.9.8</version>  <!-- OLD VULNERABLE VERSION! -->
    </dependency>
</dependencies>
```

**Why this version?**
- Jackson 2.9.8 has multiple known CVEs
- CVE-2019-12086 (High severity)
- CVE-2019-12384 (High severity)
- Perfect for learning!

---

### Step 2: Commit and Push

```bash
git add pom.xml
git commit -m "Add vulnerable Jackson dependency for testing"
git push origin main
```

---

### Step 3: Watch Pipeline Fail

Go to CircleCI and watch the pipeline.

**This time, security_scan will FAIL!** ❌

**In security_scan logs:**

```
[INFO] Analyzing Dependencies...
[INFO] 
[INFO] Generating Report...
[INFO] 
[ERROR] 
[ERROR] One or more dependencies were identified with known vulnerabilities:
[ERROR] 
[ERROR] jackson-databind-2.9.8.jar (pkg:maven/com.fasterxml.jackson.core/jackson-databind@2.9.8)
[ERROR]     CVE-2019-12086 Severity: HIGH Score: 7.5
[ERROR]     CVE-2019-12384 Severity: HIGH Score: 7.5
[ERROR] 
[ERROR] See the dependency-check report for more details.
[ERROR] 
[INFO] BUILD FAILURE
```

**Pipeline stops at security_scan!** ✅

**publish job never runs** - vulnerable code is blocked!

---

### Step 4: View Detailed Report

Click **Artifacts** → **security-report/dependency-check-report.html**

**Report shows:**

**Summary:**
```
Dependencies Scanned: 48
Vulnerable Dependencies: 1
Critical: 0
High: 2
Medium: 0
Low: 0
```

**Vulnerability Details:**

```
jackson-databind-2.9.8.jar
└── CVE-2019-12086
    Severity: HIGH (7.5/10)
    Description: A Polymorphic Typing issue was discovered in 
    FasterXML jackson-databind before 2.9.9.
    Recommendation: Upgrade to version 2.9.9 or later

└── CVE-2019-12384
    Severity: HIGH (7.5/10)
    Description: FasterXML jackson-databind might allow attackers 
    to have a variety of impacts by leveraging failure to block 
    the logback-core class.
    Recommendation: Upgrade to version 2.9.9.1 or later
```

---

## Part 5 - Fix the Vulnerability (30 minutes)

Now let's fix the vulnerability by updating to a secure version.

### Step 1: Determine Safe Version

**From the report, we know:**
- Current version: 2.9.8 (vulnerable)
- Recommended: 2.9.9 or later

**Let's check latest version:**

Go to https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind

**Latest stable version:** 2.15.3 (or whatever is current)

---

### Step 2: Update pom.xml

**Change the version:**

```xml
<!-- BEFORE (vulnerable) -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.8</version>  <!-- VULNERABLE -->
</dependency>

<!-- AFTER (fixed) -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.3</version>  <!-- SECURE -->
</dependency>
```

---

### Step 3: Test Locally (Optional but Recommended)

**Verify it builds:**

```bash
mvn clean install -DskipTests
```

**Expected output:**
```
[INFO] BUILD SUCCESS
```

**Run dependency check locally:**

```bash
mvn org.owasp:dependency-check-maven:check
```

**Expected output:**
```
[INFO] Found 0 vulnerabilities
[INFO] BUILD SUCCESS
```

✅ **Fixed!**

---

### Step 4: Commit and Push Fix

```bash
git add pom.xml
git commit -m "Fix: Update Jackson to secure version 2.15.3"
git push origin main
```

---

## Part 6 - Verify Fix Works (20 minutes)

Watch the pipeline run again with the fix.

### Step 1: Monitor Pipeline in CircleCI

**Pipeline runs again:**

```
1. build      ✅ SUCCESS
2. test       ✅ SUCCESS
3. security_scan ✅ SUCCESS (no vulnerabilities!)
4. publish    ✅ SUCCESS (image pushed to Docker Hub)
```

**This time, all jobs pass!** 🎉

---

### Step 2: Check Security Scan Logs

**In security_scan job:**

```
[INFO] Analyzing Dependencies...
[INFO] 
[INFO] Scanning: jackson-databind-2.15.3.jar
[INFO] 
[INFO] Generating Report...
[INFO] Found 0 vulnerabilities
[INFO] 
[INFO] BUILD SUCCESS
```

✅ **No vulnerabilities found!**

---

### Step 3: View Updated Report

**Artifacts** → **security-report/dependency-check-report.html**

**Summary:**
```
Dependencies Scanned: 48
Vulnerable Dependencies: 0 ✅
Critical: 0
High: 0
Medium: 0
Low: 0
```

**Status: ALL CLEAR** ✅

---

### Step 4: Verify Docker Image Published

1. Go to https://hub.docker.com
2. Navigate to your repository: `YOUR_USERNAME/devops-demo`
3. Check "Tags" tab
4. You should see a new push with tag `latest`
5. Check timestamp - matches your recent commit

**Success!** Secure code made it to Docker Hub! 🎉

---

### What Just Happened?

**Complete DevSecOps Flow:**

```
1. Developer updates dependency
   ↓
2. Commits to Git
   ↓
3. CircleCI builds application
   ↓
4. CircleCI runs tests
   ↓
5. CircleCI scans for vulnerabilities ← NEW!
   ↓
6. If secure: Build Docker image
   ↓
7. Push to Docker Hub
   ↓
8. Ready for deployment
```

**If vulnerabilities found at step 5:**
- Pipeline stops immediately
- Developer gets notification
- No vulnerable code reaches production
- Must fix before deploying

**This is Shift-Left Security in action!**

---

## Part 7 - Understanding the Security Report (15 minutes)

Let's dive deeper into interpreting security reports.

### CVE Severity Scores

**CVSS (Common Vulnerability Scoring System):**

| Score | Severity | Action Required |
|-------|----------|-----------------|
| 9.0-10.0 | **Critical** | Fix immediately! |
| 7.0-8.9 | **High** | Fix within days |
| 4.0-6.9 | **Medium** | Fix within weeks |
| 0.1-3.9 | **Low** | Fix when convenient |
| 0.0 | **None** | No action needed |

---

### Example Vulnerability Entry

```
CVE-2019-12086
Severity: HIGH (7.5/10)
CVSS Vector: CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N

Description:
A Polymorphic Typing issue was discovered in FasterXML 
jackson-databind before 2.9.9. When Default Typing is enabled,
the class leads to remote code execution.

Affected Versions: 
All versions before 2.9.9

Fixed Versions:
2.9.9, 2.9.9.1, 2.10.0 and later

Recommendation:
Upgrade to jackson-databind version 2.9.9 or later
```

**What this tells you:**

1. **CVE ID:** Unique identifier for tracking
2. **Score:** 7.5 = HIGH priority
3. **Description:** What the vulnerability allows
4. **Affected Versions:** Which versions are vulnerable
5. **Fix:** Update to safe version

---

### False Positives

**Sometimes the scanner reports false positives:**

**Example:**
```
Vulnerability: CVE-2023-12345
Affected: my-library-1.0.0
But: You're using 1.0.1 which fixes it!
```

**Why?**
- Scanner uses pattern matching
- May not recognize fix version
- Requires manual verification

**What to do:**
- Check CVE details manually
- Confirm your version
- If false positive, suppress in config
- Document why suppressed

---

### Suppressing False Positives

Create `dependency-check-suppression.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
    <suppress>
        <notes>False positive - already using fixed version</notes>
        <cve>CVE-2023-12345</cve>
    </suppress>
</suppressions>
```

**Add to pom.xml:**

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <configuration>
        <suppressionFile>dependency-check-suppression.xml</suppressionFile>
    </configuration>
</plugin>
```

**Use sparingly!** Only for confirmed false positives.

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

**Why:** Catch vulnerabilities in feature branches before merging.

---

### 2. Set Failure Thresholds

**Configure what severity causes build failure:**

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>  <!-- Fail on HIGH or CRITICAL -->
    </configuration>
</plugin>
```

**Options:**
- `<failBuildOnCVSS>9</failBuildOnCVSS>` - Only critical
- `<failBuildOnCVSS>7</failBuildOnCVSS>` - High and critical
- `<failBuildOnCVSS>4</failBuildOnCVSS>` - Medium and above

---

### 3. Keep Dependencies Updated

**Regular maintenance:**

```bash
# Check for updates weekly
mvn versions:display-dependency-updates
```

**Don't wait for vulnerabilities!**
- Proactive updates = easier
- Reactive fixes = stressful emergencies

---

### 4. Review Reports Regularly

**Even if builds pass:**
- Check for new Medium/Low severity issues
- Plan updates
- Track security debt

---

### 5. Automate Everything

**Your pipeline should:**
- ✅ Scan on every commit
- ✅ Generate reports automatically
- ✅ Fail builds on critical issues
- ✅ Notify team of vulnerabilities
- ✅ Block vulnerable code from production

**No manual security reviews needed!**

---

## Summary

### What You Accomplished Today

1. ✅ Added OWASP Dependency-Check to CircleCI pipeline
2. ✅ Configured automated security scanning on every commit
3. ✅ Intentionally introduced a vulnerable dependency
4. ✅ Saw pipeline fail when vulnerabilities detected
5. ✅ Read and interpreted security reports
6. ✅ Fixed vulnerability by updating dependency
7. ✅ Verified pipeline passes with secure code
8. ✅ Blocked vulnerable code from reaching Docker Hub

---

### Key Takeaways

**DevSecOps in Action:**
- Security checks run automatically
- Vulnerabilities caught immediately
- No vulnerable code reaches production
- Shift-left prevents expensive late fixes

**Your Pipeline Now:**
```
Build → Test → Security Scan → Publish
```

**Security as a Gate:**
- High/Critical vulnerabilities block deployment
- Forces immediate fixes
- Creates secure-by-default culture

**Continuous Security:**
- Every commit is scanned
- Database updated regularly
- Always know your security status

---

### Before DevSecOps vs After

**Before (Lessons 4.7-4.12):**
```
✅ Automated build
✅ Automated tests
✅ Automated deployment
❌ No security checks
❌ Vulnerable code could reach production
❌ Security issues discovered too late
```

**After (Lesson 4.14):**
```
✅ Automated build
✅ Automated tests
✅ Automated security scanning ← NEW!
✅ Automated deployment (only if secure)
✅ Vulnerabilities caught immediately
✅ Production stays secure
```

---

### Real-World Impact

**Without DevSecOps:**
- Average time to fix vulnerability: 38 days
- Cost of production security fix: $1,000 - $10,000
- Risk of breach: HIGH

**With DevSecOps:**
- Average time to fix vulnerability: 1-2 days
- Cost of development security fix: $100 - $500
- Risk of breach: LOW

**Your pipeline now catches issues in minutes, not months!**

---

## Troubleshooting Guide

### Issue 1: Security scan takes forever (>15 minutes)

**Cause:** Downloading CVE database on every run

**Solution:**
Add caching to speed up subsequent scans:

```yml
security_scan:
  steps:
    - checkout
    - restore_cache:
        keys:
          - nvd-data-{{ epoch }}
          - nvd-data-
    - run:
        name: Run Dependency Check
        command: mvn org.owasp:dependency-check-maven:check
    - save_cache:
        paths:
          - ~/.m2/repository/org/owasp/dependency-check-data
        key: nvd-data-{{ epoch }}
```

---

### Issue 2: Build fails but report shows 0 vulnerabilities

**Cause:** Different failure reason (not security)

**Solution:**
1. Check full build logs
2. Look for other errors (compilation, tests)
3. Security scan only runs if previous jobs pass

---

### Issue 3: Can't access security report

**Cause:** Report not stored as artifact

**Solution:**
Verify in config.yml:
```yml
- store_artifacts:
    path: target/dependency-check-report.html
    destination: security-report
```

---

### Issue 4: Too many false positives

**Cause:** Aggressive matching

**Solution:**
1. Verify each CVE manually
2. Update to latest versions
3. Suppress confirmed false positives
4. Consider different scanning tool

---

### Issue 5: "Unable to download NVD data"

**Cause:** Network issues or NVD API rate limiting

**Solution:**
```yml
- run:
    name: Run Dependency Check
    command: mvn org.owasp:dependency-check-maven:check -DretryCount=3
    no_output_timeout: 20m
```

---

## Additional Resources

### OWASP Dependency-Check
- [Official Documentation](https://jeremylong.github.io/DependencyCheck/)
- [Maven Plugin Configuration](https://jeremylong.github.io/DependencyCheck/dependency-check-maven/)
- [Suppression File Guide](https://jeremylong.github.io/DependencyCheck/general/suppression.html)

### CVE Information
- [National Vulnerability Database](https://nvd.nist.gov/)
- [CVE Details](https://www.cvedetails.com/)
- [Snyk Vulnerability Database](https://snyk.io/vuln/)

### CircleCI Security
- [CircleCI Security Best Practices](https://circleci.com/docs/security/)
- [Storing Artifacts](https://circleci.com/docs/artifacts/)
- [Using Orbs for Security](https://circleci.com/developer/orbs)

### Maven Security
- [Maven Security Scanning Guide](https://maven.apache.org/guides/mini/guide-maven-security.html)
- [Dependency Management](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)

### Videos
- [DevSecOps Pipeline Tutorial](https://www.youtube.com/results?search_query=devsecops+pipeline+tutorial)
- [OWASP Dependency Check Demo](https://www.youtube.com/results?search_query=owasp+dependency+check)
- [Fixing Vulnerabilities in CI/CD](https://www.youtube.com/results?search_query=fixing+vulnerabilities+cicd)

---

## Next Steps

### Continue Learning

**1. Explore Other Security Tools:**
- Try Snyk for comparison
- Experiment with Trivy for container scanning
- Add SAST (static code analysis)

**2. Advanced Configuration:**
- Set custom failure thresholds
- Configure email notifications
- Add Slack alerts for vulnerabilities

**3. Monitoring:**
- Track security metrics over time
- Create security dashboard
- Monitor dependency health

**4. Team Practices:**
- Establish security review process
- Create vulnerability response plan
- Share security reports with team

---

## Optional: Remove Test Vulnerability

If you added the vulnerable Jackson dependency just for learning, remove it now:

**Edit pom.xml:**

```xml
<!-- REMOVE THIS -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.3</version>
</dependency>
```

**Why remove?**
- Spring Boot already includes Jackson
- No need for explicit dependency
- Cleaner pom.xml

**Commit:**
```bash
git add pom.xml
git commit -m "Remove redundant Jackson dependency"
git push origin main
```

---

**End of Lesson 4.14**

**Congratulations!** You've successfully implemented DevSecOps practices in your CI/CD pipeline. Your application is now protected by automated security scanning that catches vulnerabilities before they reach production. This is a critical skill for modern software development! 

**You've completed the entire DevSecOps module!** 

**You are now equipped with real-world DevSecOps skills!**