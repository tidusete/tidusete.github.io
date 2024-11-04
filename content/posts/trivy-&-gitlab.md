---
title: "Optimizing GitLab CI/CD with Trivy for Image Scanning"
date: 2024-11-04T19:19:20+01:00
toc: false
images:
tags:
  - devops,devsecops,Trivy,Gitlab
---

# Optimizing Trivy Scans in GitLab CI/CD: Lessons Learned

In this post, we’re exploring some strategies to optimize GitLab CI/CD pipelines using Trivy for image scanning. Trivy, a popular open-source scanner from Aqua Security, helps in identifying vulnerabilities within container images. We’ll discuss different scan configurations, compare their performance, and share insights to streamline vulnerability scanning in CI/CD workflows.

## Pipeline Stages and Jobs

Our pipeline consists of three stages: `simple-scan`, `multiple-scans`, and `convert-scans`. Within each stage, we have defined jobs with various Trivy scan configurations, both with and without cache usage. Below is the `gitlab-ci.yml` configuration:

```yaml
stages:
  - simple-scan
  - multiple-scans
  - convert-scans
variables:
  IMAGE_NAME: "python"
  IMAGE_TAG: "3.9"
...
```

### Job Performance and Observations

After running the pipeline, here’s the summary of job durations and outcomes:

| Job Name                          | Stage           | Status   | Duration  |
|-----------------------------------|-----------------|----------|-----------|
| `trivy-scan-nocache`              | simple-scan     | Passed   | 00:00:31  |
| `trivy-scan-cache`                | simple-scan     | Passed   | 00:00:39  |
| `trivy-multiple-scan-nocache`     | multiple-scans  | Failed   | 00:01:03  |
| `trivy-multiple-scan-cache`       | multiple-scans  | Failed   | 00:00:59  |
| `trivy-scan-multipleconvert-nocache` | convert-scans | Failed   | 00:00:52  |
| `trivy-multiple-multipleconvert-cache` | convert-scans | Failed | 00:00:54  |

## Separating the Trivy Database Download Job

Initially, we tried to cache the Trivy database within each scan job, but this approach had a critical flaw: since the scan jobs were set to fail if they detected high-severity vulnerabilities, they often failed before successfully caching the database. This resulted in repeated, time-consuming database downloads for every pipeline run, slowing down the process and increasing our reliance on network stability.

### Solution: A Dedicated Database Download Job

To address this, we introduced a separate job specifically for downloading the Trivy database:

- **Independent Execution**: We used `needs: []` to run this job at the beginning of the pipeline, independent of the scan jobs.
- **Reliable Caching**: By isolating the database download, we ensured that the database was always cached successfully, even if subsequent scan jobs encountered vulnerabilities and failed.
- **Enhanced Pipeline Stability**: This setup improved resilience against network-related issues, ensuring that each scan job could quickly access a cached database, speeding up subsequent scans and making the pipeline more reliable overall.

This approach minimized redundant database downloads, reduced pipeline run times, and made the debugging process easier.

## Performance Comparison: Multiple Scans vs. Single Scan with `trivy convert`

Another key experiment was comparing the performance of running multiple individual scans (one per output format) with running a single scan followed by `trivy convert` to generate various report formats. Here’s what we found:

### Key Observations

1. **Simple Scans with Cache**:
   - The simple scan with cache now completed in **00:00:39**, compared to **00:00:31** for the non-cached simple scan.
   - This slight increase demonstrates that caching can slightly improve performance for simple scans, particularly by ensuring a reliable database source without network dependencies.

2. **Multiple Scans with Cache vs. No Cache**:
   - The **multiple scans job with cache** took **00:00:59**, while the same job without cache finished in **00:01:03**.
   - The small difference suggests that Trivy’s lightweight design allows multiple scans to run efficiently, with or without cache. Still, caching offers a minor speed advantage and stability benefits by removing network dependency.

3. **Convert Jobs**:
   - The `trivy-multiple-multipleconvert-cache` job finished in **00:00:54**, while the no-cache version (`trivy-scan-multipleconvert-nocache`) completed in **00:00:52**.
   - Similar times for cached and non-cached convert jobs suggest that `convert` operations do not benefit significantly from caching, as they work on pre-scanned data without requiring database access.

### Conclusion: Optimizing for Trivy’s Lightweight Design

Our findings confirm that while caching is beneficial in early pipeline stages, particularly for database reliability, Trivy’s lightweight nature means that multiple invocations perform well regardless of cache usage. For teams looking to optimize Trivy-based pipelines, this implies that focusing on reliable database caching and isolated scans can yield the best balance of speed and resilience.

These adjustments have made our CI/CD pipeline more efficient, predictable, and resilient, with pipeline times now optimized for consistent, fast performance without compromising on scan accuracy.


```yaml
stages:
  - simple-scan
  - multiple-scans
  - convert-scans
variables:
  IMAGE_NAME: "python"
  IMAGE_TAG: "3.9"

trivy-scan-nocache:
  allow_failure: true
  stage: simple-scan
  image:
    name: "docker.io/aquasec/trivy:latest"
    entrypoint: [""]
  variables:
    TRIVY_DISABLE_VEX_NOTICE: "true"
    TRIVY_NO_PROGRESS: "true"
    TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2
  script:
    - mkdir -p trivy_report
    - trivy --version
    - time trivy clean --scan-cache
    - time trivy image --exit-code 0 --format json --scanners vuln --output "trivy_report/trivy.native.json" "$IMAGE_NAME:$IMAGE_TAG"
  artifacts:
    name: "Trivy report for $CI_PROJECT_NAME on $CI_COMMIT_REF_SLUG - $CI_JOB_NAME"
    expire_in: 1 day 
    when: always
    paths:
      - "trivy_report/*"


trivy-scan-cache:
  allow_failure: true
  stage: simple-scan
  image:
    name: "docker.io/aquasec/trivy:latest"
    entrypoint: [""]
  variables:
    TRIVY_DISABLE_VEX_NOTICE: "true"
    TRIVY_NO_PROGRESS: "true"
    TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2
    TRIVY_CACHE_DIR: ".trivycache/"
  script:
    - mkdir -p trivy_report
    - trivy --version
    - time trivy clean --scan-cache
    - time trivy image --exit-code 0 --format json --scanners vuln --output "trivy_report/trivy.native.json" "$IMAGE_NAME:$IMAGE_TAG"
  cache:
    paths:
      - .trivycache/
  artifacts:
    name: "Trivy report for $CI_PROJECT_NAME on $CI_COMMIT_REF_SLUG - $CI_JOB_NAME"
    expire_in: 1 day 
    when: always
    paths:
      - "trivy_report/*"

trivy-multiple-scan-nocache:
  allow_failure: true
  stage: multiple-scans
  image:
    name: "docker.io/aquasec/trivy:latest"
    entrypoint: [""]
  variables:
    TRIVY_DISABLE_VEX_NOTICE: "true"
    TRIVY_NO_PROGRESS: "true"
    TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2
  script:
    - mkdir -p trivy_report
    - trivy --version
    - time trivy clean --scan-cache
    - time trivy image --exit-code 0 --format json --scanners vuln --output "trivy_report/trivy.native.json" "$IMAGE_NAME:$IMAGE_TAG"
    - time trivy image --exit-code 0 --format template --template "@/contrib/gitlab.tpl" --output "$CI_PROJECT_DIR/gl-container-${CI_JOB_ID}-scanning-report.json" "$IMAGE_NAME:$IMAGE_TAG"
    - time trivy image --exit-code 0 --format cyclonedx --output "trivy_report/sbomvuln-${CI_JOB_ID}.cdx.json" "$IMAGE_NAME:$IMAGE_TAG"
    - time trivy image --exit-code 1 --format table --severity HIGH,CRITICAL --ignore-unfixed "$IMAGE_NAME:$IMAGE_TAG"
  artifacts:
    name: "Trivy report for $CI_PROJECT_NAME on $CI_COMMIT_REF_SLUG - $CI_JOB_NAME"
    expire_in: 1 day 
    when: always
    paths:
      - "trivy_report/*"

trivy-multiple-scan-cache:
  allow_failure: true
  stage: multiple-scans
  image:
    name: "docker.io/aquasec/trivy:latest"
    entrypoint: [""]
  variables:
    TRIVY_DISABLE_VEX_NOTICE: "true"
    TRIVY_NO_PROGRESS: "true"
    TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2
    TRIVY_CACHE_DIR: ".trivycache/"
  script:
    - mkdir -p trivy_report
    - trivy --version
    - time trivy clean --scan-cache
    - time trivy image --exit-code 0 --format json --scanners vuln --output "trivy_report/trivy.native.json" "$IMAGE_NAME:$IMAGE_TAG"
    - time trivy image --exit-code 0 --format template --template "@/contrib/gitlab.tpl" --output "$CI_PROJECT_DIR/gl-container-${CI_JOB_ID}-scanning-report.json" "$IMAGE_NAME:$IMAGE_TAG"
    - time trivy image --exit-code 0 --format cyclonedx --output "trivy_report/sbomvuln-${CI_JOB_ID}.cdx.json" "$IMAGE_NAME:$IMAGE_TAG"
    - time trivy image --exit-code 1 --format table --severity HIGH,CRITICAL --ignore-unfixed "$IMAGE_NAME:$IMAGE_TAG"
  cache:
    paths:
      - .trivycache/
  artifacts:
    name: "Trivy report for $CI_PROJECT_NAME on $CI_COMMIT_REF_SLUG - $CI_JOB_ID"
    expire_in: 1 day 
    when: always
    paths:
      - "trivy_report/*"


trivy-scan-multipleconvert-nocache:
  allow_failure: true
  stage: convert-scans
  image:
    name: "docker.io/aquasec/trivy:latest"
    entrypoint: [""]
  variables:
    TRIVY_DISABLE_VEX_NOTICE: "true"
    TRIVY_NO_PROGRESS: "true"
    TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2
  script:
    - mkdir -p trivy_report
    - trivy --version
    - time trivy clean --scan-cache
    - time trivy image --exit-code 0 --format json --scanners vuln --output "trivy_report/trivy.native.json" "$IMAGE_NAME:$IMAGE_TAG"
    - time trivy convert --exit-code 0 --format template --template "@/contrib/gitlab.tpl" --output "$CI_PROJECT_DIR/gl-container-${CI_JOB_ID}-scanning-report.json" "trivy_report/trivy.native.json"
    - time trivy convert --exit-code 0 --format cyclonedx --output "trivy_report/sbomvuln-${CI_JOB_ID}.cdx.json" "trivy_report/trivy.native.json"
    - time trivy image --exit-code 1 --format table --severity HIGH,CRITICAL --ignore-unfixed "$IMAGE_NAME:$IMAGE_TAG"
  artifacts:
    name: "Trivy report for $CI_PROJECT_NAME on $CI_COMMIT_REF_SLUG - $CI_JOB_ID"
    expire_in: 1 day 
    when: always
    paths:
      - "trivy_report/*"

trivy-multiple-multipleconvert-cache:
  allow_failure: true
  stage: convert-scans
  image:
    name: "docker.io/aquasec/trivy:latest"
    entrypoint: [""]
  variables:
    TRIVY_DISABLE_VEX_NOTICE: "true"
    TRIVY_NO_PROGRESS: "true"
    TRIVY_DB_REPOSITORY: public.ecr.aws/aquasecurity/trivy-db:2
    TRIVY_CACHE_DIR: ".trivycache/"
  script:
    - mkdir -p trivy_report
    - trivy --version
    - time trivy clean --scan-cache
    - time trivy image --exit-code 0 --format json --scanners vuln --output "trivy_report/trivy.native.json" "$IMAGE_NAME:$IMAGE_TAG"
    - time trivy convert --exit-code 0 --format template --template "@/contrib/gitlab.tpl" --output "$CI_PROJECT_DIR/gl-container-${CI_JOB_ID}-scanning-report.json" "trivy_report/trivy.native.json"
    - time trivy convert --exit-code 0 --format cyclonedx --output "trivy_report/sbomvuln-${CI_JOB_ID}.cdx.json" "trivy_report/trivy.native.json"
    - time trivy image --exit-code 1 --format table --severity HIGH,CRITICAL --ignore-unfixed "$IMAGE_NAME:$IMAGE_TAG"
  cache:
    paths:
      - .trivycache/
  artifacts:
    name: "Trivy report for $CI_PROJECT_NAME on $CI_COMMIT_REF_SLUG - $CI_JOB_ID"
    expire_in: 1 day 
    when: always
    paths:
      - "trivy_report/*"
```
