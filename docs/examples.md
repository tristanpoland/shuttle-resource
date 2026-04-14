# Usage Examples

The following examples illustrate common patterns for using Shuttle Resource in Concourse CI pipelines. All examples assume the resource type and resource have been declared as shown in [integrations.md](./integrations.md).

---

## Basic Example: Triggering a Remote Job and Waiting

This is the most common pattern: one job initiates a shuttle and another job waits for it to complete.

### Pipeline Definition

```yaml
resource_types:
  - name: shuttle
    type: registry-image
    source:
      repository: tristanpoland/shuttle-resource
      tag: latest

resources:
  - name: coordination-shuttle
    type: shuttle
    source:
      bucket: my-concourse-bucket
      access_key_id: ((aws_access_key_id))
      secret_access_key: ((aws_secret_access_key))
      region: us-east-1
      path: shuttles/coordination

jobs:
  - name: caller
    plan:
      - put: coordination-shuttle

  - name: remote-worker
    plan:
      - get: coordination-shuttle
        trigger: true
      - task: do-work
        config:
          platform: linux
          image_resource:
            type: registry-image
            source: {repository: alpine}
          run:
            path: sh
            args: ["-c", "echo 'work done'"]
      - put: coordination-shuttle
        params:
          return: true

  - name: waiter
    plan:
      - get: coordination-shuttle
        passed: [caller]
      - get: coordination-shuttle
        params:
          wait: true
      - task: continue-after-response
        config:
          platform: linux
          image_resource:
            type: registry-image
            source: {repository: alpine}
          run:
            path: sh
            args: ["-c", "echo 'remote job completed'"]
```

### How It Works

1. The `caller` job runs a `put` step, creating a new shuttle version in S3 stamped with the job's identity.
2. The `remote-worker` job is triggered by the new version, performs its work, and then runs `put` with `return: true`, marking the shuttle as complete.
3. The `waiter` job retrieves the shuttle version and uses `wait: true` to poll until the shuttle is marked as done. It then continues with its own tasks.

---

## Advanced Example: Failure Propagation

If the remote job encounters an error, it can signal failure back to the waiting job.

```yaml
  - name: remote-worker
    plan:
      - get: coordination-shuttle
        trigger: true
      - task: do-work
        config:
          platform: linux
          image_resource:
            type: registry-image
            source: {repository: alpine}
          run:
            path: sh
            args: ["-c", "exit 1"]  # Simulate failure
        on_failure:
          put: coordination-shuttle
          params:
            return: true
            fail: true
      - put: coordination-shuttle
        params:
          return: true

  - name: waiter
    plan:
      - get: coordination-shuttle
        params:
          wait: true
          continue_on_failure: true
      - task: handle-result
        config:
          platform: linux
          image_resource:
            type: registry-image
            source: {repository: alpine}
          run:
            path: sh
            args: ["-c", "echo 'checked result'"]
```

When `fail: true` is passed by the returning job, the shuttle payload is marked as not passed. The waiting job reads this state. Without `continue_on_failure: true`, the wait step will fail the build; with it, the build continues and downstream tasks can inspect the result.

---

## Advanced Example: Skipping Coordination

In some scenarios, a job may want to skip the shuttle step entirely — for example, when running in a development environment where the remote job does not exist.

```yaml
  - name: optional-caller
    plan:
      - get: coordination-shuttle
        params:
          skip: true
      - task: always-runs
        config:
          platform: linux
          image_resource:
            type: registry-image
            source: {repository: alpine}
          run:
            path: echo
            args: ["Skipped shuttle coordination"]
```

The `skip` parameter causes the `in` step to return immediately without reading from or writing to S3, which is useful for local testing or no-op pipeline paths.

---

## Advanced Example: Bootstrapping with seed_version

For a fresh pipeline with no prior versions, Concourse's `check` step would return no versions and jobs would never trigger. The `seed_version` flag injects a synthetic version to bootstrap the pipeline:

```yaml
resources:
  - name: coordination-shuttle
    type: shuttle
    source:
      bucket: my-concourse-bucket
      access_key_id: ((aws_access_key_id))
      secret_access_key: ((aws_secret_access_key))
      region: us-east-1
      path: shuttles/coordination
      seed_version: true
```

After the initial trigger, real versions take over and `seed_version` has no further effect.

---

## Parameter Reference

### `out` Parameters

The `out` step takes no parameters. It creates a new version and records the calling job's identity automatically using Concourse's built-in environment variables.

### `in` Parameters

| Parameter             | Type    | Default | Description |
|-----------------------|---------|---------|-------------|
| `wait`                | boolean | false   | Poll until the shuttle is marked done, then clean up |
| `return`              | boolean | false   | Mark the shuttle as done with the current job's result |
| `fail`                | boolean | false   | When used with `return`, marks the result as failed |
| `skip`                | boolean | false   | No-op; return immediately without touching S3 |
| `continue_on_failure` | boolean | false   | When used with `wait`, do not fail the build if the shuttle was marked as failed |

Only one of `wait`, `return`, or `skip` may be specified at a time.
