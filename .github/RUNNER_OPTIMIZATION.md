# GitHub Actions Runner Optimization Guide

This document provides guidelines for selecting appropriate GitHub Actions runners for workflows in this repository to optimize resource usage, reduce costs, and improve efficiency.

## Overview

Our workflows are categorized by resource requirements to ensure each job runs on appropriately sized runners, preventing both over-provisioning (wasted resources) and under-provisioning (slow builds or timeouts).

## Resource Tier Classification

### Small Tier (ubuntu-latest)
**Characteristics:**
- CPU: 2 cores
- Memory: 7 GB
- Typical duration: < 15 minutes
- Use cases: Linting, security scans, lightweight checks

**Workflows in this tier:**
- **Frogbot Security Scan** (`frogbot.yml`)
  - Timeout: 15 minutes
  - Resources: Standard ubuntu-latest runner
  - Rationale: Security scanning with JFrog Frogbot is I/O bound and doesn't require extensive CPU/memory

- **Lint Job** (`ci.yml` - lint job)
  - Timeout: 10 minutes
  - Resources: Standard ubuntu-latest runner
  - Rationale: RuboCop linting is fast and CPU-light

### Medium Tier (ubuntu-latest)
**Characteristics:**
- CPU: 2 cores
- Memory: 7 GB
- Typical duration: 15-30 minutes
- Use cases: Unit tests, integration tests, gem building

**Workflows in this tier:**
- **Test Job** (`ci.yml` - test job)
  - Timeout: 30 minutes
  - Resources: Standard ubuntu-latest runner
  - Parallelization: Matrix strategy across Ruby versions (2.6, 2.7, 3.0, 3.1, 3.2)
  - Rationale: Ruby unit tests are relatively lightweight. Parallelization across versions provides efficient test coverage.

- **Integration Tests** (`ci.yml` - integration job)
  - Timeout: 30 minutes
  - Resources: Standard ubuntu-latest runner
  - Parallelization: Matrix strategy across Ruby versions (2.7, 3.0, 3.1)
  - Rationale: Integration tests may take longer but standard runners are sufficient

- **Release Job** (`release.yml`)
  - Timeout: 20 minutes
  - Resources: Standard ubuntu-latest runner
  - Rationale: Gem building and publishing is lightweight

### Large Tier (ubuntu-latest-4-core or larger)
**Not currently needed for this project.**

If future workflows require more resources (e.g., complex builds, large-scale integrations), consider:
- ubuntu-latest-4-core: 4 cores, 16 GB RAM
- ubuntu-latest-8-core: 8 cores, 32 GB RAM

## Timeout Strategy

All jobs have explicit timeout-minutes set to prevent runaway processes:

| Workflow | Job | Timeout | Rationale |
|----------|-----|---------|-----------|
| frogbot.yml | frogbot-scan | 15 min | Security scans typically complete in 5-10 minutes |
| ci.yml | lint | 10 min | Linting is very fast, usually < 5 minutes |
| ci.yml | test | 30 min | Allows time for comprehensive test suite across Ruby versions |
| ci.yml | integration | 30 min | Integration tests may need additional time |
| release.yml | release | 20 min | Gem building and publishing is straightforward |

## Parallelization Strategy

### Matrix Strategies
We use matrix strategies to maximize runner efficiency:

**Test Job:**
```yaml
strategy:
  fail-fast: false
  matrix:
    ruby: ['2.6', '2.7', '3.0', '3.1', '3.2']
```
- Runs tests in parallel across 5 Ruby versions
- `fail-fast: false` ensures all versions are tested even if one fails
- Total wall time is equivalent to a single Ruby version test

**Integration Job:**
```yaml
strategy:
  fail-fast: false
  matrix:
    ruby: ['2.7', '3.0', '3.1']
```
- Tests key Ruby versions for integration scenarios
- Reduces total test time through parallelization

### Job Dependencies
Currently, jobs run independently without dependencies to maximize parallelization. If needed, dependencies can be added:

```yaml
jobs:
  test:
    # ...

  integration:
    needs: test  # Only run after test succeeds
    # ...
```

## Resource Monitoring

All workflows include resource monitoring steps to track actual usage:

### Start Monitoring
```yaml
- name: Log resource usage (start)
  run: |
    echo "=== Resource Usage at Start ==="
    echo "CPU cores: $(nproc)"
    echo "Memory: $(free -h)"
    echo "Disk: $(df -h)"
    echo "Time: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

### End Monitoring
```yaml
- name: Log resource usage (end)
  if: always()
  run: |
    echo "=== Resource Usage at End ==="
    echo "Memory: $(free -h)"
    echo "Disk: $(df -h)"
    echo "Time: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

This monitoring helps:
- Identify workflows that consistently underutilize resources
- Detect workflows approaching resource limits
- Make data-driven decisions about runner sizing

## Optimization Checklist

When creating or modifying workflows:

- [ ] Set explicit `timeout-minutes` on all jobs
- [ ] Choose the smallest runner tier that meets requirements
- [ ] Add resource monitoring steps (start and end)
- [ ] Use matrix strategies for parallelization where applicable
- [ ] Set `fail-fast: false` in matrix strategies unless fast failure is desired
- [ ] Consider job dependencies to optimize runner allocation
- [ ] Document the rationale for runner selection
- [ ] Review logs after initial runs to verify resource usage

## Ongoing Optimization

### Monthly Review
Review workflow run times and resource usage monthly:

1. Check GitHub Actions usage dashboard
2. Analyze resource monitoring logs
3. Identify workflows consistently over/under-utilizing resources
4. Adjust runner sizes and timeouts as needed

### When to Upsize Runners
Consider larger runners if:
- Jobs frequently timeout
- Jobs consistently use > 80% of available memory
- Build times are significantly longer than competitors/industry standards
- Parallelization would significantly reduce wall time

### When to Downsize Runners
Consider smaller runners if:
- Jobs complete in < 50% of timeout
- Memory usage is consistently < 50%
- CPU usage is consistently low

## Runner Auto-Scaling

GitHub-hosted runners automatically scale based on demand. Our configuration optimizes for:
- **Peak efficiency**: Matrix strategies distribute load across multiple runners
- **Cost efficiency**: Timeouts prevent wasted runner time
- **Reliability**: Appropriate resource allocation prevents OOM or timeout failures

## Additional Resources

- [GitHub Actions Runner Specifications](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners)
- [GitHub Actions Usage Limits](https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration)
- [Optimizing GitHub Actions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#jobsjob_idtimeout-minutes)

## Questions or Issues?

If you encounter workflow issues or have questions about runner selection:
1. Review the resource monitoring logs in the workflow run
2. Check this guide for recommended configurations
3. Open an issue with the workflow name and relevant logs
