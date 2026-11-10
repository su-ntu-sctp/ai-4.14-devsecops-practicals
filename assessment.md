# Assessment / Quiz

### Q1 (True/False)

Q1: Dependabot alerts need a workflow file in `.github/workflows/` before they work?

A - True

B - False

---

Q2: Why is Dependabot a non-blocking check in this lesson?

A - It is too slow

B - It runs on GitHub, outside the CircleCI pipeline, and alerts on any severity - the blocking Snyk checks decide what stops a release

C - It cannot read pom.xml files

D - It only works on private repositories

---

Q3: Pinning a GitHub Action to a full commit SHA protects you if an attacker moves the action's version tag to malicious code?

A - True

B - False

---

Q4: Why do we turn on automatic dependency submission for devops-demo?

A - So Dependabot can push code to the repository

B - So GitHub runs Maven and the dependency graph includes transitive libraries, not just the starters in pom.xml

C - So CircleCI can read Dependabot alerts

D - So the ZAP scan runs faster

---

Q5: A dismissed Dependabot alert reopens automatically after an expiry date, like an ignore in `.snyk`?

A - True

B - False

---

Q6: What type of security testing does OWASP ZAP perform?

A - SAST - it reads your source code

B - SCA - it checks your dependencies

C - DAST - it tests the running application

D - Container scanning - it checks the Docker image

---

Q7: Which ZAP scan is safe to run against your live app?

A - The baseline scan, because it only sends normal requests and inspects the responses

B - The full scan, because it finds more problems

C - Both are equally safe

D - Neither - ZAP can only scan local apps

---

Q8: When a fix exists for a vulnerability, upgrading the dependency is better than ignoring the finding?

A - True

B - False

---

Q9: Where do Dependabot alerts appear in GitHub?

A - Only in the job logs

B - In the Security tab under Dependabot

C - In the repository's Wiki

D - In a new pull request

---

Q10: It is acceptable to scan any website with OWASP ZAP, even if you don't own it?

A - True

B - False

## Notes
- Keep each question atomic and aligned to the lesson objectives.
- Put the answer immediately after each question.
