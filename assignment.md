# Assignment (Optional)

## Brief

Implement automated security scanning in your CI/CD pipeline and practice fixing vulnerabilities.

1. **Add OWASP Dependency-Check to Your Pipeline**
   - Use your existing Spring Boot project with CircleCI pipeline (e.g., devops-demo)
   - Open `.circleci/config.yml` file
   - Add a new `security_scan` job after the test job:
     - Use `cimg/openjdk:21.0` Docker image
     - Checkout code
     - Restore Maven cache
     - Run OWASP Dependency-Check: `mvn org.owasp:dependency-check-maven:check`
     - Store the HTML report as an artifact at `target/dependency-check-report.html`
   - Update the workflow to include the security_scan job:
     - Run security_scan after test completes
     - Run publish only after security_scan passes
   - Commit and push your changes:
```bash
     git add .circleci/config.yml
     git commit -m "Add OWASP Dependency-Check to pipeline"
     git push origin main
```
   - Monitor the pipeline in CircleCI:
     - Watch the security_scan job execute (first run takes 5-8 minutes)
     - Check the Artifacts tab for the security report
   - Take screenshots showing:
     - Your CircleCI pipeline with all jobs (build, test, security_scan, publish, deploy)
     - The security scan job completion status
   - Write a brief explanation (3-4 sentences) describing:
     - What the security scan does
     - Where it fits in your pipeline
     - What happens if vulnerabilities are found

2. **Simulate and Fix a Vulnerability**
   - Add an old vulnerable dependency to your `pom.xml`:
```xml
     <dependency>
         <groupId>org.apache.commons</groupId>
         <artifactId>commons-collections4</artifactId>
         <version>4.0</version>
     </dependency>
```
   - Commit and push the change
   - Watch the pipeline fail at the security_scan job
   - Access the security report from CircleCI Artifacts
   - Document the vulnerability:
     - CVE number
     - Severity score
     - Brief description of the vulnerability
   - Fix the vulnerability by updating to version 4.4:
```xml
     <dependency>
         <groupId>org.apache.commons</groupId>
         <artifactId>commons-collections4</artifactId>
         <version>4.4</version>
     </dependency>
```
   - Commit and push the fix
   - Verify the pipeline now passes all jobs including security_scan
   - Write a brief report (4-5 sentences) explaining:
     - What vulnerability was found
     - Why it's dangerous
     - How you fixed it
     - Why automated security scanning is important in CI/CD

## Submission (Optional)

- Submit the URL of the GitHub Repository that contains your work to NTU black board.
- Should you reference the work of your classmate(s) or online resources, give them credit by adding either the name of your classmate or URL.

## References
- Java: https://docs.oracle.com/javase/
- Spring Boot: https://docs.spring.io/spring-boot/docs/current/reference/html/
- PostgreSQL: https://www.postgresql.org/docs/
- OWASP: https://cheatsheetseries.owasp.org/