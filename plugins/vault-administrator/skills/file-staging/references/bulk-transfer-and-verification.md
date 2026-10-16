# Bulk Folder Transfer, Delta Sync, and Proof of Upload

Builds on `references/api-endpoints.md`: read that first for the individual endpoint shapes.
This file covers moving a whole tree, and the three workflows layered on top of a transfer.

## Folder Tree Upload (core)

1. Walk the local directory recursively. Build two ordered lists: folders (parent before child)
   and files (with their size).
2. For each folder in the walk, generate a `Create Folder or File` call with `kind=folder`,
   mapping the local relative path onto the Vault File Staging (VFS) target root. Folders must
   exist before files are placed inside them, and the API creates one item per request, so a tree
   needs one call per subfolder.
3. For each file, branch on size using the rule in `references/api-endpoints.md`: a small file is
   one `Create Folder or File` call with `kind=file`, a large one is the full three-call resumable
   sequence. A tree with mixed file sizes needs both paths generated, not one picked for the whole
   run.
4. Preserve the local relative path under the VFS target path unless the developer asked for a
   flattened layout.

## Folder Tree Download (core)

1. Call `List Items at a Path` with `recursive=true` on the VFS root being pulled.
2. From the response, recreate the local subfolder structure from every `kind: folder` entry
   before downloading anything.
3. For each `kind: file` entry, download via `Download Item Content`. A file past the `Range` cap
   in `references/api-endpoints.md` comes down as sequential `Range: bytes={min}-{max}` requests
   appended locally in order. There is no server-side "give me the whole large file in one
   response" option once it exceeds that cap.
4. Preserve the VFS relative path under the local target directory unless the developer asked for
   a flattened layout.

## Delta Sync

Delta sync is only partially supported, because `List Items at a Path` returns `kind`, `path`,
`name`, `size`, and `modified_date` per file and nothing more. That leaves size and modified date
as the whole comparison. Don't invent a hash, and don't borrow one from an upload response
elsewhere, since the listing endpoint this depends on doesn't carry it.

Design:
1. Pull the remote listing with `recursive=true` on the VFS target root. Build a map of
   `remote_relative_path -> {size, modified_date}`.
2. Walk the local tree. For each local file, look up its remote counterpart by relative path.
3. Upload the file if: no remote counterpart exists (new file), OR remote `size` differs from
   local size, OR local mtime is newer than remote `modified_date`.
4. Skip the file otherwise.

State this caveat to the developer every time this branch generates a script: it is a
size-plus-timestamp heuristic, not a byte-for-byte comparison. A file edited in place that lands
at the same size, with an mtime that isn't newer than what's already on VFS, will not be detected
as changed and will be silently skipped. Certainty means diffing the content locally first, since
there is no remote checksum to compare against afterward.

## Proof of Upload

Proof of upload is partially supported as well, and only while the transfer is happening. The
single-shot `Create Folder or File` response carries a whole-file `file_content_md5`, and
`Upload to a Session` carries `part_content_md5` for the one part. Both endpoints accept an
optional `Content-MD5` request header if the developer wants Vault to validate the upload on the
way in. Once those responses are gone the evidence is gone with them.

What a report can honestly claim:
- A single-shot file whose `file_content_md5` matches a locally computed hash is proven intact. A
  mismatch there is real transfer corruption, so fail that file loudly rather than logging it.
- A resumable file has per-part checksums and no whole-file one. Mark it not verifiable and say
  why. Never let it read as checksum-verified when the API never returned that evidence.
- Remote `size` from a recursive listing, compared against the local manifest, is a second and
  weaker signal, and it is the only after-the-fact check available.

Build the local manifest before the transfer (the developer's own `md5sum` or PowerShell
`Get-FileHash -Algorithm MD5`, drafted by this skill and run by them), capture the upload
responses during it, and pull the listing after. Let the developer choose the report format; the
columns fall out of the three claims above. The whole artifact is produced locally by their own
script, and Vault issues no transfer-completion certificate of its own.
