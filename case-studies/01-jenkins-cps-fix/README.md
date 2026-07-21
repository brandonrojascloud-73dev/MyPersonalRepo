# Jenkins Pipeline Crash: CPS Serialization Fix

## The Problem

Production Jenkins instance started crashing during customer hibernation workflows. The pipeline would fail immediately with:

```
java.io.NotSerializableException: org.jenkinsci.plugins.workflow.job.WorkflowJob
```

This wasn't a minor bug—it blocked all customer hibernation operations, which are critical for cost management. The team had to revert the change and manually hibernate customers for two weeks while we figured out what went wrong.

## Context

We were adding a safety guard to prevent `CustomerHibernate` and `CustomerUnHibernate` jobs from running while `UpdateCustomers` (a long-running job that touches all customers) was active. The logic seemed simple:

```groovy
// Inside the pipeline's node {} block
def blockingJob = Jenkins.instance.getItemByFullName('UpdateCustomers')
if (blockingJob.isBuilding()) {
    // Skip this run
}
```

The code worked in our heads. It failed in production.

## Root Cause

Jenkins Pipeline uses **CPS (Continuation-Passing Style)** transformation. When a pipeline pauses (like waiting for user input), it serializes all local variables to disk so it can resume later.

`WorkflowJob` objects are **not serializable**. When we stored the job reference in a variable inside the `node {}` block, Jenkins tried to serialize it during the next pause point and crashed.

The stack trace told the story:
```
HashMap.writeObject → CpsThreadGroup.saveProgram → NotSerializableException: WorkflowJob
```

## The Fix

Move the Jenkins API lookup into a `@NonCPS` annotated method. This tells Jenkins: "Don't try to serialize anything inside this method. Just return a simple value."

```groovy
@NonCPS
def isJobBuilding(String jobName) {
    def job = Jenkins.instance.getItemByFullName(jobName)
    return job != null && job.isBuilding()
}
```

Then call it from the pipeline:
```groovy
if (isJobBuilding('UpdateCustomers')) {
    currentBuild.result = 'NOT_BUILT'
    echo 'Skipped: UpdateCustomers is running'
    return
}
```

The pipeline only sees a boolean—fully serializable.

## Validation

We tested with Jenkins Replay in the dev environment:
- Replayed build #992 against `CustomerHibernate`
- Passed the original crash point
- Reached the destructive confirmation stage (where we stopped intentionally)

The fix was applied to both the primary and multiregion Jenkins configuration repositories.

## Lessons Learned

1. **CPS serialization is invisible until it bites you.** The code looks fine, compiles fine, but fails at runtime during pause points.

2. **`@NonCPS` is a boundary, not a workaround.** It's the correct pattern for Jenkins API calls that return non-serializable objects.

3. **Replay is your friend.** Jenkins Replay lets you test pipeline changes without triggering real side effects. Critical for destructive operations.

4. **Guardrails matter.** The original change was meant to prevent concurrent operations that could corrupt customer state. The fix preserved that safety without breaking the pipeline.

## Why This Matters

This isn't just about fixing a bug. It's about understanding **how Jenkins actually works under the hood**. Most people write pipelines that work until they don't. Understanding CPS serialization lets you write pipelines that are robust by design.

The fix took 2 hours to implement, 2 weeks of production impact to justify, and 6 months of stability to validate. That's the reality of infrastructure work.