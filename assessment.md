# Assessment / Quiz

### Q1 (True/False)

Q1: OSV-Scanner needs an account and an API key before it can scan a project?

A - True

B - False

---

Q2: Why is OSV-Scanner set up as a non-blocking check in this lesson?

A - It is too slow to run in GitHub Actions

B - It fails on vulnerabilities of any severity, so as a gate a single LOW issue would stop every deployment

C - It cannot read pom.xml files

D - GitHub Actions cannot block deployments

---

Q3: Pinning a GitHub Action to a full commit SHA protects you if an attacker moves the action's version tag to malicious code?

A - True

B - False

---

Q4: Which fields should every entry in `osv-scanner.toml` have?

A - Only the vulnerability ID

B - The ID and the package name

C - The ID, an expiry date (`ignoreUntil`) and a reason

D - The ID and your GitHub username

---

Q5: After the `ignoreUntil` date has passed, OSV-Scanner reports the ignored vulnerability again?

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

Q9: Where do OSV-Scanner's results appear in GitHub?

A - Only in the job logs

B - In the Security tab under Code scanning

C - In the repository's Wiki

D - In a new pull request

---

Q10: It is acceptable to scan any website with OWASP ZAP, even if you don't own it?

A - True

B - False

## Notes
- Keep each question atomic and aligned to the lesson objectives.
- Put the answer immediately after each question.
