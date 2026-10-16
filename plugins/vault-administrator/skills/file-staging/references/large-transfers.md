# Planning a very large transfer

For moving tens of gigabytes or tens of thousands of files into Vault File Staging (VFS). The
mechanics of the tree walk live in `bulk-transfer-and-verification.md` and the size rules in
`api-endpoints.md`. This file is about sizing the job before anyone commits to a finish time.

## What bounds the job

Vault meters the API in a burst window and slows down rather than failing: past the limit it
delays every call for the rest of the window and reports the delay on the response instead of
erroring. The current limit, the header names, and the listing and path-length ceilings that
migration trees with long titles run into all come from the docs MCP
(`mcp__vault-docs-mcp__search_documentation`, source `vault_api_reference`).

Two different things can be the bottleneck. Many small files are call-bound, one call per file
against a fixed per-window budget, and extra parallelism stops helping once that budget is spent.
A few huge files are bandwidth-bound, where the call budget barely matters and the link speed
decides. Work out both floors, plan against the larger one, and the other comes free.

## Sizing worksheet

Get these from the user, or from a listing of the source:

1. Total bytes.
2. File count, and how many are over 50 MB.
3. Deepest path length and longest file name.
4. Where the data is: this laptop, a file share, another server, cloud storage.
5. Who uploads: an Admin (root paths) or a user (`u{id}` paths).

Estimate calls as one per file under 50 MB, plus `1 + ceil(size / part size) + 1` for each larger
file (create, parts, commit). Divide that by the burst budget for the call-bound floor. Total
bytes over a measured upload rate gives the bandwidth-bound floor. Rates swing with the link, the
region, and what else is running on the Vault's POD, so measure one rather than quoting one.

## Parallelism and resume

Parallel streams help until the call budget or the link saturates, and 5 to 10 concurrent uploads
is the usual sweet spot for files under 50 MB. Large files parallelize across files and never
within a session, because the parts of one session have to go in order. Treat every upload as
retryable: a failed single-shot upload can simply be re-sent, and a resumable session can be
listed to see which parts landed and picked up from the next part number. Verify at the end with
a recursive listing against the local manifest (`bulk-transfer-and-verification.md`, Proof of
Upload).

## Shared platform effects

Vault runs on shared PODs. Staging uploads are direct calls and are not queued, but anything that
turns the staged files into Vault content runs as a job and queues behind other jobs on the POD.
A transfer that finishes on schedule can still be followed by a load that waits, so plan the two
as separate phases with separate estimates. The clocks on Vault-written files and on uncommitted
upload sessions do not pause for a queue.
