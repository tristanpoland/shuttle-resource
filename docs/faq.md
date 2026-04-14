# Frequently Asked Questions

---

## What is Shuttle Resource?

Shuttle Resource is a [Concourse CI](https://concourse-ci.org/) custom resource type that enables coordinated handoffs between pipeline jobs. It allows one job to initiate a "shuttle" — a named rendezvous point stored in S3 — and another job to wait for a response. This makes it possible to model asynchronous workflows, cross-pipeline triggers, and remote job coordination directly within Concourse.

---

## Why is it called "Shuttle Resource"?

The name reflects the resource's purpose: it shuttles state and signals between jobs that may not otherwise be able to communicate. One job sends a message; another receives and responds. The resource acts as the carrier between them.

---

## What problem does it solve?

Concourse pipelines are typically linear or fan-in/fan-out within a single pipeline. Shuttle Resource extends this by enabling two separate jobs — potentially in different pipelines or even different teams — to synchronize. For example:

- A deployment job can notify a validation job in another pipeline and wait for an approval or test result.
- A job can signal an external process and block until it receives a response.
- Long-running asynchronous work can be handed off without blocking a worker for the full duration.

---

## Is this project production ready?

Shuttle Resource is a focused, small codebase with a clear scope. It has been used in real Concourse environments. That said, the project is actively maintained and evolving. Before using it in a critical production pipeline, review the open issues on [GitHub](https://github.com/tristanpoland/shuttle-resource/issues) to understand any known limitations.

For highly available use cases, ensure your S3 bucket is configured with appropriate redundancy and that your Concourse workers can reliably reach the S3 endpoint.

---

## What S3 permissions does the resource need?

At minimum, the IAM credentials provided in the source configuration require:

- `s3:GetObject` — to read payloads
- `s3:PutObject` — to create and update payloads
- `s3:DeleteObject` — to clean up after a shuttle is consumed
- `s3:ListBucket` — to enumerate versions during the `check` step

Scope these permissions to the specific bucket and key prefix used by the resource.

---

## Does it work with S3-compatible storage?

Yes. Any S3-compatible object store that supports the above operations should work. Set the `endpoint` field in the source configuration to point to your custom endpoint. This has been used with MinIO and Ceph RADOS Gateway. See [integrations.md](./integrations.md) for configuration examples.

---

## Can I use Shuttle Resource across multiple Concourse teams or deployments?

Yes. Because the coordination state lives in S3, any Concourse job with the correct source configuration (same bucket, path, and credentials) can interact with the same shuttle, regardless of which team or Concourse instance it belongs to.

---

## What happens if the remote job never responds?

If the returning job never calls `put` with `return: true`, the waiting job will poll indefinitely (every 30 seconds). There is currently no built-in timeout. To handle this, use Concourse's built-in job timeout mechanism:

```yaml
- get: coordination-shuttle
  params:
    wait: true
  timeout: 1h
```

---

## How do I suggest a feature or report a bug?

Open an issue on [GitHub](https://github.com/tristanpoland/shuttle-resource/issues). Please include:

- A description of the problem or feature request
- The version of Concourse and the resource image tag you are using
- Relevant pipeline YAML (with credentials redacted)

---

## How can I contribute?

Contributions are welcome. See [contributing.md](./contributing.md) for guidelines on submitting pull requests, code style expectations, and the review process.
