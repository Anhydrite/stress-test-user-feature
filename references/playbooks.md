# Playbooks

Use the playbook that matches the feature. Each one lists the minimum proof that the feature is usable, not merely present. Adapt the steps; do not treat the lists as exhaustive.

## UI web

**Proof focus:** the user can complete the journey, the state survives, and failures are visible and recoverable.

- Walk the real flow in a browser, not just the API behind it.
- Reload the page and confirm the state persists.
- Open the feature in a second tab or session and check consistency.
- Test the empty state, the loading state, and the error state.
- Test the permission boundary: a lower role must not see or trigger the action.
- Check the console and network tabs for errors, failed requests, and leaked data.
- Test keyboard navigation and the smallest supported viewport if the feature is user-facing.
- Verify that destructive actions require confirmation and can be cancelled.
- Capture a screenshot of the final state plus the network response that proves the action.

**Common false positives:** optimistic UI that never reconciles, a toast shown before the request resolves, a list that only updates after a manual refresh, a disabled button that hides a server error.

## API / backend

**Proof focus:** the contract holds, the side effects are real, and the state can be read back.

- Read the route, handler, schema, and validation before calling the endpoint.
- Test the success path with a real payload and inspect the full response, not only the status code.
- Read the created or updated resource back through a separate endpoint or a direct query.
- Test authentication and authorization for each relevant role and tenant.
- Test invalid, missing, extra, and wrong-type fields.
- Test empty, maximum, oversized, unicode, and duplicate inputs.
- Test idempotency: repeat the same write and check for duplicates or double effects.
- Test concurrency: two writes at the same time, and a write during a read.
- Test dependency failure: timeout, 5xx, malformed response.
- Check pagination, filtering, sorting, and total counts with a known dataset.
- Check that errors do not leak stack traces, SQL, secrets, or internal IDs.
- Verify DB state, cascades, and orphan rows after delete.

**Common false positives:** `200` with an empty body, a write that returns the input instead of the persisted row, a background effect that never runs, a validation error returned with a success status.

## CLI

**Proof focus:** the command produces a usable artifact and behaves correctly when it fails.

- Run `--help` and verify the documented flags match the implementation.
- Run the happy path and inspect stdout, stderr, and the exit code separately.
- Use the produced file, directory, or resource; do not stop at "command exited 0".
- Test missing arguments, unknown flags, invalid values, and conflicting flags.
- Test a read-only or permission-denied target.
- Test idempotency: run the command twice and check the second result.
- Test interruption and cleanup: kill the process mid-run and check for leftovers.
- Test paths with spaces, unicode, symlinks, and a read-only parent.
- Verify the exit code is non-zero on failure and that the error is actionable.

**Common false positives:** exit code 0 on a partial failure, output written to the wrong stream, a file created but empty or invalid, a temp directory left behind.

## Infra / VM

**Proof focus:** the resource is reachable, usable, persistent, and fully released.

Minimum sequence for a VM-like resource:

1. Create it and capture the resource ID.
2. Wait for the real ready state, not just the API response.
3. Reach it: SSH, console, or the documented access path.
4. Run a real command inside it and capture the output.
5. Write a marker, reboot, and verify the marker survived.
6. Stop and start it; verify it comes back with the same identity and data.
7. Take a snapshot or backup if the feature supports it; restore and verify.
8. Check network, disk, and quota side effects.
9. Delete it and verify the resource, disks, IPs, and snapshots are gone.
10. Verify the quota or billing entry is released.

Also test:
- creation with invalid size, image, network, or name
- creation at the quota limit
- concurrent create and delete
- access before ready
- delete while running
- orphaned resources after a failed create

**Common false positives:** the VM is listed but never boots, the IP is allocated but unreachable, delete removes the record but leaves the disk, a snapshot exists but cannot be restored.

## Async jobs / queues

**Proof focus:** the job reaches a terminal state and its effect happens exactly once.

- Enqueue a job and capture its ID.
- Wait for a terminal state; do not assume "queued" means "will run".
- Inspect the output or the downstream effect.
- Test retry on a transient failure.
- Test a permanent failure and the dead-letter or error path.
- Send the same job twice and check idempotency.
- Kill a worker mid-job and verify recovery or safe failure.
- Check ordering if the feature depends on it.
- Check that the queue drains and no job is stuck.
- Verify cleanup of temporary files, locks, and partial writes.

**Common false positives:** the job is marked complete before the effect, the retry duplicates the effect, a failed job disappears without a trace, the queue is processed but the output is never read.

## Cross-cutting checks

Run these for every domain when relevant:

- **Restart** — does the feature survive a process, container, or browser restart?
- **Concurrency** — do two actors corrupt each other's state?
- **Idempotency** — does a retry create a duplicate?
- **Authorization** — can a lower role or another tenant perform the action?
- **Cleanup** — are all created resources released?
- **Observability** — can you tell from logs and metrics that it worked or failed?
- **Regression** — does the surrounding behavior still work?
