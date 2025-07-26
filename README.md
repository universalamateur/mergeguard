# MergeGuard

Descriuption: Preventing CI Pipeline Execution for Merge Requests with Conflicts

## Executive Summary: 

The Problem of Unmergeable Pipelines the Core ProblemIn modern CI/CD workflows, Merge Request (MR) pipelines are a cornerstone of pre-merge validation. A common and problematic scenario arises when an MR's target branch (e.g., main) is updated with new commits, introducing a merge conflict with the MR's source branch. Despite the MR entering a state where it cannot be merged, GitLab's default behavior is to continue creating and running new "Merge Request pipelines" for any subsequent commits pushed to the source branch.

## The Impact

These pipelines, while technically executing, are fundamentally flawed and misleading. Standard Merge Request pipelines are configured to run tests against the source branch's code in isolation. As the official documentation states, these pipelines "run on the contents of the source branch only, ignoring the content of the target branch". Consequently, a pipeline for a conflict-ridden MR can pass with a "green" checkmark, creating a false sense of security. The code is validated, but it's validated against a context that no longer reflects the reality of the target branch. This behavior has several negative consequences:
- Wasted Resources: Each pipeline consumes valuable and often costly CI/CD runner minutes on jobs that provide no meaningful signal about the code's mergeability.
- Developer Confusion: A passing pipeline on an unmergeable MR creates confusion. Developers may assume their changes are ready for integration, only to be blocked by the conflict at the moment of merging.
- Eroded Trust: When the CI/CD system repeatedly reports success for code that is functionally broken in the context of the project's main line of development, it erodes developer trust in the automation and its signals.

## The Objective

The objective of this technical experiment and its resulting documentation is to develop and validate a robust, reusable pattern to address this issue. The goal is to detect the merge-conflict state as early as possible within a CI/CD pipeline and terminate its execution immediately. This prevents resource waste, eliminates confusing signals, and restores the integrity of the pre-merge validation process.

## Comprehensive Variable Review
A thorough analysis of GitLab's extensive list of predefined CI/CD variables was conducted to determine if any variable exposes the mergeability status of an MR. The review confirmed the availability of numerous variables that provide essential context for merge request pipelines, such as:   

- `CI_MERGE_REQUEST_IID`: The project-local ID of the merge request.
- `CI_MERGE_REQUEST_SOURCE_BRANCH_NAME`: The name of the source branch.
- `CI_MERGE_REQUEST_TARGET_BRANCH_NAME`: The name of the target branch.
- `CI_PROJECT_ID`: The ID of the project.
- `CI_API_V4_URL`: The root URL for the GitLab v4 API.

## Solution I:

Conflict Detection via GitLab API Polling

```bash
# Initial API call to trigger a recheck
API_URL="${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}?with_merge_status_recheck=true"
RESPONSE=$(curl --silent --header "JOB-TOKEN: ${CI_JOB_TOKEN}" "${API_URL}")
# Extract the detailed merge status using jq
MERGE_STATUS=$(echo "${RESPONSE}" | jq -r '.detailed_merge_status')
echo "Current merge status: ${MERGE_STATUS}"
```

## Soltuion II:

Conflict Detection via Git Command Simulation

```bash
# Ensure the runner has the latest information about the target branch
git fetch origin "$CI_MERGE_REQUEST_TARGET_BRANCH_NAME"
git merge --no-commit --no-ff "origin/$CI_MERGE_REQUEST_TARGET_BRANCH_NAME"
```

## Experiment Setup

