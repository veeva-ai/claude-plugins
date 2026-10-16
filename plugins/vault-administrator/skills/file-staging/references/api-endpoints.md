# File Staging API Endpoints

All paths below are relative to `https://$VAULT_DNS`, and every request needs `Authorization:
$AUTH`. This reference never fills that in with a real value. Draft commands with only
what Step 2 gathered filled in and leave everything else as a placeholder.

Calling any of them requires the *Application: File Staging: Access* permission on top of the
general *API: Access API*. Only the standard Vault Owner and System Admin security profiles
carry it by default, though an Admin can grant it to other profiles.

## Path Semantics

Non-Admin and Admin users address Vault File Staging (VFS) paths differently. Get this wrong and
every other command below 404s or hits the wrong file.

- **Non-Admin users** see only their own directory and address it as if it were the root: a file
  at `/Folder/File.txt` is under their own user directory.
- **Admin users** (Vault Owner, System Admin) see the whole VFS tree, rooted at `/`, including
  every user's directory. To reach their own files they must still go through their own user
  folder: `/u{their_user_id}/Folder/File.txt`. A path like `/Folder/File.txt` for an Admin user is
  a folder sitting at VFS's root, not inside any user's directory.
- User directories are always named `u{user_id}`.
- When nobody is sure which of the two applies, list `/` after signing in. An Admin gets the root
  listing with the per-user `u{id}` folders in it; a non-Admin only ever sees their own.
- Each user's root has an `Inbox` directory: `/Inbox` (non-Admin) or `/u{user_id}/Inbox` (Admin).
  Uploading a file there creates a Vault *Staged* document as a side effect, not just a file
  placement. Renaming a file in Inbox creates a *new* Staged document; it does not rename or
  remove the one already created for the original file. Deleting a file from Inbox does not
  delete the Staged document Vault already created from it.

## Which upload a file gets

A file of 50MB or less goes up in one call. Anything larger needs a resumable upload session,
which tops out at a 500GB file. That ceiling is VFS's own. Direct Upload, which puts one file
straight onto a document or record, stops at 4GB instead, so a file between the two sizes can be
staged but cannot be attached directly. Part sizes, part counts, session expiry, and how many
sessions a Vault allows at once come from the docs MCP
(`mcp__vault-docs-mcp__search_documentation`, source `vault_api_reference`, "resumable upload
sessions").

## Endpoints

All under `/api/{version}/services/file_staging/`. For bodies, headers, and response fields, ask
the docs MCP (source `vault_api_reference`) with the endpoint name below in quotes instead of
guessing a parameter.

- **Create Folder or File**, `POST items`, `multipart/form-data`, one item per call and 50MB per
  file. Its response is the only place a whole-file MD5 comes back. Commas, semicolons, and
  spaces in a file name stage fine, but curl reads `,` and `;` inside `-F "file=@..."` as its own
  separators and fails to open the file. Quote the name inside the part,
  `-F 'file=@"Okafor, Maya - Resume.docx"'`, or use a multipart library, and URL-encode the
  path on list and download calls.
- **Create Resumable Upload Session**, `POST upload`.
- **Upload to a Session**, `PUT upload/{upload_session_id}`. Parts go in numerical order, and
  re-using a part number overwrites that part.
- **Commit Upload Session**, `POST upload/{upload_session_id}`. Returns a `job_id` to poll and no
  whole-file checksum for the reassembled file.
- **List Upload Sessions**, `GET upload`, where Admins see every session in the Vault. **Get
  Upload Session Details**, `GET upload/{upload_session_id}`. **Abort Upload Session**, `DELETE
  upload/{upload_session_id}`, which purges the parts already uploaded.
- **List Items at a Path**, `GET items/{item}`. No checksum or hash field is returned on any item,
  and nothing else hands one back for a file already on VFS.
- **Download Item Content**, `GET items/content/{item}`, capped at 50MB per `Range` request, so a
  larger file comes down in windows and is reassembled locally.
- **Update Folder or File**, the move and rename call, `PUT items/{item}`. Async, returns a
  `job_id`.
- **Delete File or Folder**, `DELETE items/{item}`. Async, returns a `job_id`, and without
  `recursive=true` deleting a non-empty folder does nothing.
