# Flaky-test-retryer

Flaky-test-retryer is a tool that automatically detects when presubmit jobs fail
due to test flakiness, and reruns them atmost 3 times. Test flakiness and other
configuration details are determined by the
[flaky-test-reporter](https://github.com/knative/test-infra/tree/master/tools/flaky-test-reporter).

## Basic Usage

Flags for this tool are:

- `--service-account` specifies the path to the file containing a service
  account for GCS access.
- `--github-account` specifies the path to the file containing a Github token
  for Github API calls.
- `--dry-run` enables dry-run mode.

### NOTE: This tool is highly coupled to Prow artifacts, Pub/Sub message formats, and the flaky-test-reporter

If debugging locally, without access to the Knative test projects, you will need
to create your own GCP project, Pub/Sub topics, and mock Prow crier. Remember to
always run with the `--dry-run` flag set.

## Architecture

The retryer is a K8S service, listening for incoming Pub/Sub messages published
by prow on job status changes. The deployment is described in
[this YAML](gke_deployment/retryer_service.yaml), and the basic flow of
execution is described below.

1. Github PR or `/test` comment triggers presubmit job.
2. Presubmit job stores artifacts in GCS and publishes Pub/Sub message.
3. flaky-test-retryer receives Pub/Sub message, determines if job can be
   processed.
4. Compare job's failed test artifacts (if any) with flaky-test-reporter's daily
   results.
5. If all failed tests are flaky, post a GitHub comment containing `/test`. If
   some failed tests are _not_ flaky, list the non-flaky tests preventing retry.
6. Repeat up to 3 times.

### Configuration

All configuration options, such as supported repositories, are inferred from the
flaky-test-reporter's results. If/when the reporter's updated to support new
jobs or repos, the retryer will automatically support it as well.

### Pub/Sub

The main thread in the retryer serves as a Pub/Sub listener and handler, waiting
for messages to come in on the specified topic. When a message is received, if
it fits our retry criteria (job failed, from supported repo, and is a presubmit)
a goroutine is created to process it and the message is acked. Otherwise, the
message is acked without spawning a new thread.

### Log Parsing

When a new thread is created, we parse the failed job's build artifacts from GCS
and collect which tests, if any, caused the failure. If the failure was caused
due to failed tests (i.e. no build issues), we collect the current flaky tests
from the reporter's logs, and cross-reference the failed presubmit tests with
the current flaky tests. The result is passed on to the Github commenter.

### Github Commenting

The Github comment bot is what keeps track of retries, as well as triggering the
retries themselves. The number of previous retries attempted is determined by
parsing the comment history of the PR itself, and retries are attempted up to a
number set
[here](https://github.com/knative/test-infra/blob/master/tools/flaky-test-retryer/github_commenter.go#L35).
There are a number of different comments that can be posted, based on the failed
tests and existing retry comments. They all follow a similar format:

> The following tests are currently flaky. Running them again to verify...
>
> | Test name        | Retries |
> | ---------------- | ------- |
> | presubmitJobName | x/3     |

and have different footers, depending on the cross-reference result and the
number of attempted retries:

> Automatically retrying... /test presubmitJobName

if all tests that failed are currently flaky, triggering a retry.

> Failed non-flaky tests preventing automatic retry of
> pull-knative-serving-integration-tests:
>
> ```
> test/that/failed.andIsNotFlaky
> ```

if there are tests that failed and are not flaky, listing those that prevent an
automatic retry.

> Job presubmitJobName expended all 3 retries without success.

if all retries are expended without a success.

## Troubleshooting

For troubleshooting questions please ping the oncall. Current oncall can be
found [here](https://knative.github.io/test-infra/)
