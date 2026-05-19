# multi-job-pipeline-drill

Assignment repository for LU 2.5 — Building Multi-Job CI Pipelines.

## Pipeline Architecture

| Job | Depends On | Runs When | Purpose |
|---|---|---|---|
| lint | — | Always | Code style validation |
| unit-tests | lint | Always | Unit test suite |
| build | lint | Always | Compile and create dist/ artifact |
| integration-tests | build | Always | Tests against actual build output |
| deploy-staging | unit-tests, integration-tests | main branch only | Deploy to staging environment |
| deploy-production | deploy-staging | main branch only | Deploy to production |
| notify | unit-tests, integration-tests, deploy-staging | Always (even on failure) | Team notification |

## Key Patterns Used

| Pattern | Implementation | Why |
|---|---|---|
| Parallel execution | unit-tests and build both need lint only | Reduces total pipeline time |
| Artifact sharing | upload-artifact in build, download-artifact in integration-tests | Fresh machine isolation requires explicit file transfer |
| Deployment gate | if: github.ref == refs/heads/main | Prevents accidental production deploys from feature branches |
| Always-run notify | if: always() | Notifications must fire even when the pipeline fails |
| Timeouts | timeout-minutes on every job | Prevents stuck jobs from blocking for 6 hours |
