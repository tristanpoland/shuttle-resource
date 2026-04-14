# Contributing to Shuttle Resource

Thank you for your interest in contributing to Shuttle Resource. This document describes the process for submitting changes, the expectations around code quality and style, and how to collaborate effectively.

---

## Getting Started

1. **Fork the repository** on GitHub.
2. **Clone your fork** locally:
   ```sh
   git clone https://github.com/your-username/shuttle-resource.git
   cd shuttle-resource
   ```
3. **Create a branch** for your change:
   ```sh
   git checkout -b feature/my-improvement
   ```
4. **Build the project** to verify your environment is set up correctly:
   ```sh
   go build ./...
   ```

---

## Opening an Issue First

For non-trivial changes — new features, new backends, or changes to the resource protocol — please open an issue before writing code. This allows maintainers to discuss the approach and avoid duplicated or misaligned effort.

For small bug fixes, documentation corrections, or typos, a pull request without a prior issue is fine.

---

## Code Style

The project follows standard Go conventions:

- Format all code with `gofmt` before committing.
- Keep package-level names unexported unless they are part of a deliberate public API.
- Prefer short, descriptive variable names over long ones.
- Avoid unnecessary abstractions; the codebase is intentionally small and direct.
- Error messages should be lowercase and not end with punctuation, consistent with Go idioms.

---

## Documentation and Tests

- If you are adding a new feature or operation mode, update the relevant documentation in `docs/`.
- If you are adding a new driver backend, add a corresponding section to `docs/extending.md` and `docs/integrations.md`.
- Write tests where practical. The project currently relies on integration-level validation, but unit tests for new logic are welcome.
- Ensure existing behavior is not broken. Run `go build ./...` and `go vet ./...` before submitting.

---

## Pull Request Guidelines

- Keep pull requests focused on a single concern. Split unrelated changes into separate pull requests.
- Write a clear pull request description that explains what the change does and why.
- Reference any related issues with `Fixes #<issue-number>` or `Relates to #<issue-number>`.
- Be responsive to review feedback. Maintainers may request changes before merging.

---

## Commit Messages

Follow the conventional commit style where appropriate:

```
feat: add Redis backend driver
fix: handle missing S3 key gracefully during wait
docs: add integration guide for MinIO
```

For small changes, a concise imperative sentence is sufficient. Avoid vague messages like "fix stuff" or "update code."

---

## Review Process

Pull requests are reviewed by maintainers on a best-effort basis. Reviews typically focus on:

- Correctness and edge case handling
- Consistency with existing code style and conventions
- Impact on backward compatibility
- Documentation coverage

Once a pull request is approved, it will be merged by a maintainer.

---

## Reporting Security Issues

Do not open public issues for security vulnerabilities. Instead, contact the repository maintainers directly via GitHub's private vulnerability reporting feature or by email if a contact address is provided in the repository profile.
