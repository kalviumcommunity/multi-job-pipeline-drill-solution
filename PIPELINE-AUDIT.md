# Pipeline Audit and Remediation Report

## Problem 1: Missing Job Dependencies (Parallel execution chaos)
* **Log Evidence:** All jobs (`lint`, `unit-tests`, `build`, `integration-tests`, `deploy-staging`, `deploy-production`, `notify`) start simultaneously upon a push.
* **Root Cause:** GitHub Actions defaults to running all jobs in parallel unless explicit dependencies are defined using the `needs` keyword.
* **Fix Applied:** Added `needs` declarations to establish a sequential execution order. `unit-tests` and `build` need `lint`; `integration-tests` needs `build`; `deploy-staging` needs `unit-tests` and `integration-tests`; `deploy-production` needs `deploy-staging`; `notify` needs `unit-tests`, `integration-tests`, and `deploy-staging`.

## Problem 2: Missing Artifact Management (Integration tests fail)
* **Log Evidence:** The `integration-tests` job fails because the `dist/` directory does not exist on the runner machine.
* **Root Cause:** Jobs run on isolated virtual environments (runners). Even if the `build` job succeeds and creates `dist/`, that directory is lost when the `build` job finishes, and `integration-tests` runs on a fresh machine.
* **Fix Applied:** Used `actions/upload-artifact@v4` in the `build` job to save the `dist/` directory, and `actions/download-artifact@v4` in the `integration-tests` job to retrieve it before running tests.

## Problem 3: Deployments on Feature Branches (Premature deployments)
* **Log Evidence:** The `deploy-staging` and `deploy-production` jobs execute when pushing to a non-main branch (e.g., a feature branch).
* **Root Cause:** There are no conditional checks to restrict deployment jobs to the `main` branch.
* **Fix Applied:** Added `if: success() && github.ref == 'refs/heads/main'` to both deployment jobs, ensuring they only run on the main branch after successful completion of previous steps.

## Problem 4: Missing Timeouts (Risk of hung jobs)
* **Log Evidence:** If a job hangs (e.g., waiting for an external service), it will continue to run until the default 6-hour timeout limit is reached.
* **Root Cause:** No explicit job-level timeout constraints are configured in the workflow.
* **Fix Applied:** Added specific `timeout-minutes` to each job (`lint`: 10, `unit-tests`: 15, `build`: 20, `integration-tests`: 30, `deploy-staging`: 15, `deploy-production`: 15, `notify`: 5) to fail fast if a job hangs.

## Problem 5: Missing Notification on Failure (Silent failures)
* **Log Evidence:** If any job fails upstream (e.g., `unit-tests`), the `notify` job is skipped because it defaults to `if: success()`.
* **Root Cause:** The `notify` job does not have an `always()` condition to guarantee its execution regardless of the pipeline's status.
* **Fix Applied:** Added `if: always()` to the `notify` job, ensuring the team is notified even when the pipeline fails.

## Problem 6: Uncontrolled Concurrency (Redundant parallel runs)
* **Log Evidence:** Pushing multiple commits to a PR in quick succession triggers multiple redundant pipeline runs simultaneously.
* **Root Cause:** The pipeline lacks a concurrency control mechanism to cancel outdated, in-progress runs when new commits are pushed to the same branch.
* **Fix Applied:** Added a workflow-level `concurrency` block, using `group: ${{ github.ref }}-pipeline` and `cancel-in-progress: true`, to ensure only the latest pipeline run for a given branch is active.
