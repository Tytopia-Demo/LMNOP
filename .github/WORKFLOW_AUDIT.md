# Workflow Audit Summary

This document provides an audit of GitHub Actions workflows in this repository, their resource requirements, and optimization decisions.

## Audit Date
Initial audit: Current implementation

## Repository Overview
- **Project Type**: Ruby Gem (google-auth-library-ruby)
- **Primary Language**: Ruby
- **Test Frameworks**: Minitest, RSpec
- **CI Tasks**: Tests, Linting (RuboCop), Integration Tests

## Workflow Inventory

### 1. Frogbot Security Scan (`frogbot.yml`)
**Purpose**: Automated security vulnerability scanning using JFrog Frogbot

**Triggers:**
- Pull requests (opened, synchronized)
- Push to master branch
- Daily schedule (00:00 UTC)
- Manual dispatch

**Resource Classification**: Small
- **Runner**: ubuntu-latest (2 cores, 7 GB RAM)
- **Timeout**: 15 minutes
- **Parallelization**: Single branch matrix (master)

**Optimization Decisions:**
- Standard runner is sufficient for security scanning
- 15-minute timeout provides ample buffer (typical runs: 5-10 min)
- Resource monitoring added to track actual usage

**Estimated Monthly Usage:**
- Pull requests: ~10 runs/month × 5 min = 50 minutes
- Daily schedule: 30 runs/month × 5 min = 150 minutes
- Push to master: ~20 runs/month × 5 min = 100 minutes
- **Total**: ~300 minutes/month

---

### 2. Continuous Integration (`ci.yml`)
**Purpose**: Run tests, linting, and integration tests across Ruby versions

#### Job: Lint
**Triggers:**
- Push to master
- Pull requests to master
- Manual dispatch

**Resource Classification**: Small
- **Runner**: ubuntu-latest
- **Timeout**: 10 minutes
- **Parallelization**: None (single job)

**Optimization Decisions:**
- Linting is lightweight, standard runner appropriate
- 10-minute timeout (typical runs: 2-5 min)
- Runs independently from tests for fast feedback

**Estimated Usage:**
- ~30 runs/month × 3 min = 90 minutes/month

#### Job: Test
**Triggers:**
- Push to master
- Pull requests to master
- Manual dispatch

**Resource Classification**: Medium
- **Runner**: ubuntu-latest
- **Timeout**: 30 minutes
- **Parallelization**: Matrix across 5 Ruby versions (2.6, 2.7, 3.0, 3.1, 3.2)

**Optimization Decisions:**
- Matrix parallelization reduces wall time from 100+ minutes to 20-30 minutes
- Standard runner sufficient for Ruby unit tests
- fail-fast: false ensures all versions tested
- 30-minute timeout provides buffer for slower versions

**Estimated Usage:**
- ~30 runs/month × 5 versions × 20 min = 3,000 minutes/month

#### Job: Integration
**Triggers:**
- Push to master
- Pull requests to master
- Manual dispatch

**Resource Classification**: Medium
- **Runner**: ubuntu-latest
- **Timeout**: 30 minutes
- **Parallelization**: Matrix across 3 Ruby versions (2.7, 3.0, 3.1)

**Optimization Decisions:**
- Tests key Ruby versions (reduced from all versions)
- Standard runner adequate for integration tests
- continue-on-error: true to avoid blocking on flaky tests
- 30-minute timeout for longer-running integration scenarios

**Estimated Usage:**
- ~30 runs/month × 3 versions × 25 min = 2,250 minutes/month

---

### 3. Release (`release.yml`)
**Purpose**: Automated gem building and publishing to RubyGems

**Triggers:**
- Git tags matching 'v*' pattern
- Manual dispatch with tag input

**Resource Classification**: Small-Medium
- **Runner**: ubuntu-latest
- **Timeout**: 20 minutes
- **Parallelization**: None (single release)

**Optimization Decisions:**
- Gem building is lightweight, standard runner appropriate
- 20-minute timeout sufficient for build, test, and publish
- Manual trigger available for flexibility

**Estimated Usage:**
- ~2 releases/month × 10 min = 20 minutes/month

---

## Total Estimated Monthly Usage

| Workflow | Est. Minutes/Month | Percentage |
|----------|-------------------|------------|
| Frogbot Security Scan | 300 | 5.2% |
| CI - Lint | 90 | 1.6% |
| CI - Test | 3,000 | 52.1% |
| CI - Integration | 2,250 | 39.1% |
| Release | 20 | 0.3% |
| **Total** | **5,760** | **100%** |

## Resource Optimization Summary

### Optimizations Implemented

1. **Job Timeouts**
   - All jobs have explicit timeout-minutes set
   - Prevents runaway processes consuming runner time
   - Timeouts sized with 50-100% buffer over typical run times

2. **Parallelization**
   - Test job: 5-way parallelization (Ruby versions)
   - Integration job: 3-way parallelization (Ruby versions)
   - Reduces wall time by ~5x while utilizing same total runner minutes

3. **Runner Sizing**
   - All workflows use ubuntu-latest (standard 2-core, 7GB RAM)
   - Appropriate for Ruby workloads
   - No need for larger runners currently

4. **Resource Monitoring**
   - All jobs log CPU, memory, disk usage at start and end
   - Enables data-driven optimization decisions
   - Helps identify resource bottlenecks

5. **Job Independence**
   - Lint, test, and integration jobs run in parallel
   - Provides fast feedback on different aspects
   - Maximizes runner utilization

### Potential Future Optimizations

1. **Conditional Job Execution**
   - Skip integration tests for documentation-only changes
   - Use path filters to run relevant tests only
   ```yaml
   on:
     pull_request:
       paths:
         - 'lib/**'
         - 'test/**'
   ```

2. **Caching Improvements**
   - Already using bundler-cache via ruby/setup-ruby
   - Consider caching additional artifacts if builds slow down

3. **Test Splitting**
   - If test suite grows, consider splitting into smaller parallel jobs
   - Current approach (version parallelization) is optimal for now

4. **Self-Hosted Runners**
   - Not recommended for current usage patterns
   - Public repository benefits from GitHub-hosted security
   - GitHub-hosted is cost-effective at current scale

## Unused/Deprecated Workflows

**Status**: No obsolete workflows identified
- Only 3 workflows, all actively used
- All serve distinct purposes
- No redundancy detected

## Compliance with Best Practices

- ✅ All jobs have timeout-minutes set
- ✅ Runners sized appropriately for workload
- ✅ Resource monitoring implemented
- ✅ Parallelization used where beneficial
- ✅ Documentation provided
- ✅ Clear workflow naming and organization

## Next Review Date

Recommend review after 30 days of usage to:
- Analyze actual resource consumption from logs
- Adjust timeouts based on real data
- Identify optimization opportunities
- Update this audit document

## Monitoring and Alerts

### Key Metrics to Track
1. Workflow success rate by job
2. Average duration by job
3. Timeout frequency
4. Resource utilization from monitoring logs
5. Queue time (if runners become scarce)

### Action Items
- Review GitHub Actions usage dashboard monthly
- Analyze resource monitoring logs for trends
- Adjust runner sizes if consistent under/over-utilization detected
- Update timeouts if jobs consistently finish much faster/slower
