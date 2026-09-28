[English](README.md) | [Polski](README.pl.md)

# Safe macOS and GitHub backups

A public guide to creating, checking, and restoring backups. Example commands assume Zsh on macOS. First define the scope of the data, then perform read-only diagnostics. Writing and deletion are separate, deliberate steps.

<a id="index"></a>
## Go to

| Stage | Section |
| --- | --- |
| Preparation | [How to use](#how-to-use) · [Security](#security) · [Structure and categories](#structure) |
| Execution | [Backup procedure](#backup) · [Verification](#verification) · [Projects](#projects) |
| Sources | [GitHub](#github) · [SSH](#ssh) · [Application and Safari settings](#safari) |
| Further steps | [Restoration](#restore) · [Troubleshooting](#troubleshooting) · [Publishing](#publishing) · [Example](#example) |

<a id="how-to-use"></a>
## How to use this guide

The notation `<...>` marks a placeholder: your own value to substitute before running a command. Do **not type** the `<` and `>` characters in final commands; the shell might interpret them as redirection. For example, replace `<BACKUP_VOLUME>` with the actual path of a mounted disk. Quote paths that contain spaces. The code in this guide is a template, not a ready-made script to run without review.

All data below is **fictional**:

```text
<HOME> = /Users/alex
<BACKUP_VOLUME> = /Volumes/Backup
<GITHUB_USERNAME> = alex-dev
<REPOSITORY_NAME> = ExampleApp
```

Other placeholders name a category, project, file, repository, or selected snapshot. Before copying, check that the substituted source and destination paths point to the intended locations.

`<BRANCH_NAME>` denotes a branch name, `<COMMIT_SHA>` a commit identifier, `<PROFILE_ID>` a profile identifier, and `<BUNDLE_IDENTIFIER>` an application identifier. Treat them as example labels, never as values from someone else's environment.

[↑ Back to contents](#index)

<a id="security"></a>
## Security

- **Never publish** a private SSH key, tokens, passwords, or credential files. A `*.pub` file is the public part of a key pair, but consider its publication deliberately as well.
- Store backups containing secrets on encrypted media; restrict access to the media and the backups. A public GitHub repository is no place for such a backup.
- Before publishing a README or guide, review its text and commit history for secrets and identifying data.
- Start with read-only diagnostics. Write only after checking the source, destination, filters, and free space. **DELETE always comes last**, after full verification and confirmation that you are not deleting the only copy.
- Do not use `curl | sh`. Use `sudo` only when truly necessary, with an unambiguous path and scope of operation.

If an error occurs, stop deletion and preserve the working source. See the [procedure](#backup) and [diagnostics](#troubleshooting).

[↑ Back to contents](#index)

<a id="structure"></a>
## Backup structure and categories

```text
<BACKUP_VOLUME>/
├── CURRENT/
│   ├── Codex/
│   ├── Ollama/
│   ├── AI-Tools/
│   ├── Zsh/
│   ├── Homebrew/
│   ├── Projects/
│   ├── iTerm2/
│   ├── SSH/
│   └── Safari/
└── ARCHIVE/
    └── <CATEGORY>/<YYYY-MM-DD>/
```

`CURRENT` is the latest **verified** copy needed for quick restoration. `ARCHIVE` holds older, verified historical points. Choose the date of the state being archived; do not mix different snapshots in one directory. Before replacing `CURRENT/<CATEGORY>/`, preserve the previous important state in `ARCHIVE` and check it. Do not overwrite the only good copy.

| Category | What to keep and why | What usually not to copy and why | How to verify and restore |
| --- | --- | --- | --- |
| Codex | Configuration in use, your instructions and scripts; they restore your workflow. | Cache, sessions, and authentication files without a separate decision; they may be reproducible or sensitive. | SHA256, file comparison, and running the tool; restore only needed settings after comparing them with active settings. |
| Ollama | Modelfiles, your templates and settings; they describe model configuration. | Downloaded weights if they can be downloaded again; they take up substantial space. | SHA256 and Modelfile review; restore files, then download the weights or restore them from a separate backup. A Modelfile alone does not contain weights. |
| AI-Tools | Your scripts, definitions and documentation; they may be unique. | Virtual environments, cache, logs and secrets; they are usually reproducible or sensitive. | SHA256, syntax check and a small functional test; restore selected scripts and dependencies. |
| Zsh | Configuration files in use, functions and your scripts; they restore the shell. | Command history and cache without a specific need; they may expose data. | SHA256 and `zsh -n` on the file being restored; after comparison, restore only needed entries. |
| Homebrew | Package, cask and tap lists or a Brewfile; they allow installations to be restored. | Download cache and bottles if available; they take up space. | SHA256 and review of the lists; after restoration, install selected items and check versions. |
| Projects | Code, `.git`, project files and unique artifacts; they may not exist elsewhere. | Only confirmed builds and cache; details in [Projects](#projects). | SHA256, checksum dry-run and Git checks; restore the project in a separate location, then check history and the build. |
| iTerm2 | Preferences, profiles, snippets, shell integration and your scripts; they restore the workspace layout. | Cache, history and tool environments; they can be large or sensitive. | SHA256, `plutil -lint` for plist files and checking profile visibility; restore only needed files. |
| SSH | Needed keys and configuration; they may be impossible to regenerate with the same identity. | Agent sockets and runtime files; they are not persistent configuration. | SHA256, permissions and the fingerprint of **your own** public key; restore with restrictive permissions. |
| Safari | Needed preferences, profiles, extension settings or snippets; they restore settings. | Cache, history, runtime files and whole containers without review; they may be huge or sensitive. | SHA256, `plutil -lint` and checking settings in the application; restore a specific entry or file. |

The category list is a template. Before including data in a backup, assess its confidentiality, size, and whether it can be downloaded again.

[↑ Back to contents](#index)

<a id="backup"></a>
## Backup procedure

The example `<SOURCE>` and `<DEST>` respectively mean the source directory and staging area. Substitute them only after diagnostics. Commands with `--delete` in this section also have `-n`, so they delete nothing; review the list of changes. Actual deletion is not part of copying.

1. **Check that the disk is mounted.** `mount` and `df -h <BACKUP_VOLUME>` must show the expected volume, not an empty local directory.
2. **Check the source.** `ls -ld <SOURCE>` and a list of needed files confirm its existence and scope. For repositories, check [Git](#github).
3. **Check free space.** Compare `du -sh <SOURCE>` with `df -h <BACKUP_VOLUME>`, accounting for existing snapshots and spare capacity.
4. **Create staging** named `<CATEGORY>.incoming` on the same volume, at the destination. Make sure the name does not refer to an important older backup. Staging keeps an incomplete copy separate from `CURRENT` or `ARCHIVE`.
5. **Copy the data.** After reviewing filters, use, for example, `rsync -a <SOURCE>/ <DEST>/`. On macOS, `-a` preserves structure, permissions and timestamps to the extent supported by `rsync`; assess the need for ACL/xattr separately.
6. **Resolve special-file problems.** If copying reports a socket or another runtime file, identify it and exclude only that item; run `rsync` again. See the [socket issue](#troubleshooting).
7. **Run a checksum dry-run.** `rsync -acn --delete --itemize-changes <SOURCE>/ <DEST>/` should report zero differences in content and file lists after applying the same filters on both sides. Do not proceed with an unexplained difference.
8. **Create `SHA256SUMS`** in the staging directory using the [template](#verification).
9. **Verify SHA256.** Every entry must return `OK`, and the number of entries must equal the number of data files.
10. **Run additional Git checks** if the copy contains a repository: check branches, worktrees, unpushed commits and `git fsck --full` for critical repositories. See [GitHub](#github).
11. **Only then rename `.incoming` to the final name.** Before `mv`, confirm that the final path does not exist or that the previous state has been properly archived separately. Renaming within the volume quickly exposes the finished copy.
12. **Delete the old source only after full verification** of `CURRENT` or `ARCHIVE`, matching hashes, assessment of unique data and a deliberate decision. Deletion is the last, separate operation.

If `rsync` interrupts copying, usually keep `.incoming`, fix the specific problem and run the copy again. After finalization, check the manifest again from the final directory. The [finalization issue](#troubleshooting) describes further conditions.

[↑ Back to contents](#index)

<a id="verification"></a>
## Backup verification

From the snapshot or `.incoming` directory, generate a manifest with **relative** paths:

```zsh
find . -type f ! -name SHA256SUMS -print0 \
  | sort -z \
  | xargs -0 shasum -a 256 >| SHA256SUMS
shasum -a 256 -c SHA256SUMS
```

The manifest does not include itself. With 100 data files, it has 100 entries, while the whole directory then contains 101 regular files. If you exclude other files from the manifest, document that explicitly and adjust the count accordingly. Symbolic links, sockets and metadata are not regular files covered by the `find` above; check them separately. For filenames containing a newline, use a tool that handles that case and test reading the manifest.

Compare content and file lists before creating the manifest:

```zsh
rsync -acn --delete --itemize-changes <SOURCE>/ <DEST>/
```

Zero output means no differences in content and no extra or missing files within the compared scope. After creating the manifest, exclude it from the comparison, for example with `--exclude=/SHA256SUMS`, applying the same other filters. A dry-run does not confirm permissions, ACL, xattr or correct source selection; check them separately. Run `shasum -c` from the directory to which the manifest paths refer.

[↑ Back to contents](#index)

<a id="projects"></a>
## Projects

Keep source code, `.git`, local branches, unpushed commits, worktree metadata, project files and unique artifacts that cannot be recreated. A complete `.git` is often the only copy of local history. See [GitHub](#github).

You may consider omitting `build/`, `builds/`, `DerivedData/`, `node_modules/`, `.gradle/`, `.godot/`, `cache/`, `dist/` and `export_templates/`. **Never exclude them automatically without review.** A finished APK, IPA or other artifact may be the only surviving version.

First inspect sizes and candidates for reproducible data:

```zsh
du -sh <PROJECT>/*
find <PROJECT> -type d \( -name build -o -name builds -o -name DerivedData -o -name node_modules -o -name .gradle -o -name .godot -o -name cache -o -name dist -o -name export_templates \) -print
```

The first command omits hidden items, so inspect those separately too. `find` only points to candidates; review every path before adding a filter. After copying, perform the [checksum comparison](#verification) with the same filters.

[↑ Back to contents](#index)

<a id="github"></a>
## GitHub and backups

A local repository contains a working directory and `.git`. GitHub as a `remote` stores only data pushed to the service. An independent GitHub backup is a separate copy of hosted data, including needed metadata obtained through an API or export. None of these three scopes automatically replaces the others.

`git clone` may fail to restore local branches and commits not pushed to the server, worktrees, dangling objects, issues, pull requests, repository settings or branch protections. This is why a complete `.git` can matter. If you use worktrees, check the connections between directories and preserve all metadata; a single working directory may not be enough.

Local repository diagnostics are read-only:

```zsh
git -C <PROJECT> status --short --branch
git -C <PROJECT> branch -a -vv
git -C <PROJECT> worktree list
git -C <PROJECT> log --branches --not --remotes --oneline
git -C <PROJECT> fsck --full
```

Record the check results without publishing private remote addresses. `dangling` does not automatically mean corruption. Do not run `git gc` or `prune`, or delete packs, before confirming that needed data has been restored elsewhere.

[↑ Back to contents](#index)

<a id="ssh"></a>
## SSH

**NEVER publish the contents of a private SSH key.** `<SSH_PRIVATE_KEY>` denotes the private file and `<SSH_PUBLIC_KEY>` the corresponding `*.pub` file. The public key can be given to a service, but the private key allows authentication as its owner and must remain secret.

Copy needed keys and SSH configuration only to encrypted media. Before copying, check that the files have the correct owner and permissions. After restoration, the SSH directory should be accessible only to its owner, a private key should usually have mode `600`, and a public key mode `644`; set permissions only on your specific files.

```zsh
ls -ld <HOME>/.ssh
ls -l <SSH_PRIVATE_KEY> <SSH_PUBLIC_KEY>
ssh-keygen -lf <SSH_PUBLIC_KEY>
```

The last command calculates the fingerprint of **your own public key**. Compare it locally with a previously trusted record or account setting; do not put a real fingerprint in a public guide. After restoration, check file SHA256 hashes, permissions and the connection to the correct service.

[↑ Back to contents](#index)

<a id="safari"></a>
## Application and Safari settings

Do not copy entire multi-gigabyte containers without review. Preferences, profiles, extension settings and snippets are often sufficient. Cache, history and runtime files may be unnecessary, large or sensitive. Determine where the relevant application version stores its settings and test restoration on a small scope.

When modifying a plist, change **only the specific entry**. First preserve a verified copy, then write the change through a temporary file while preserving permissions. Finally, always run `plutil -lint` on the changed file. For Safari and extensions, also check the result of `pluginkit -m -A -D`; an entry may point to an extension embedded in an application rather than a separate application. See [troubleshooting](#troubleshooting).

[↑ Back to contents](#index)

<a id="restore"></a>
## Restoration procedure

1. Choose `CURRENT/<CATEGORY>/` or a specific `ARCHIVE/<CATEGORY>/<YYYY-MM-DD>/`. Confirm that the disk is mounted and the snapshot date is correct.
2. From the snapshot directory, run `shasum -a 256 -c SHA256SUMS`; every item must return `OK`. Also check that needed files and metadata are complete.
3. Compare the snapshot with the current configuration: file versions, permissions, data scope and local Git history. Preserve the current state so you can roll back if needed.
4. Do not overwrite a working environment without comparison. Choose the smallest required scope; restore it to a separate location first, if possible.
5. Restore only the needed file, directory or repository. Transfer secrets only in a secure environment and restore the appropriate permissions.
6. After restoration, run the relevant application or service and check its operation: syntax for Zsh, branches and history for Git, `plutil -lint` for plist files, permissions and connection for SSH. Record the result and any missing items.

If SHA256 verification fails, do not consider the snapshot verified. Preserve it for diagnosis and select another verified point.

[↑ Back to contents](#index)

<a id="troubleshooting"></a>
## Troubleshooting

Start with the [backup procedure](#backup), [verification](#verification) and [restoration](#restore). The diagnostics below do not delete data; commands that write are described with conditions.

### 1. `rsync -E`: `Permission denied` or `._*`

**Symptom:** Copying with `-E` reports access denied or a problem with an AppleDouble `._*` file.

**Cause:** The system macOS `openrsync` handles metadata, ACL and xattr through `-E`; AppleDouble is a way to store such metadata. The destination or file may not accept it.

**Safe diagnostics:** `rsync -an --checksum <SOURCE_FILE> <DESTINATION>` and `rsync -anE --checksum <SOURCE_FILE> <DESTINATION>`.

**Solution:** If the variant without `-E` works, use `rsync -a` for code and projects **provided that** ACL/xattr are not required. If they are required, determine exactly which metadata fails and use a compatible destination.

**Verification:** Checksum dry-run, SHA256 and a separate check of required metadata; see [verification](#verification).

### 2. `rsync: mkstempsock: Invalid argument`

**Symptom:** `rsync` stops at a special file.

**Cause:** A UNIX socket or other special file cannot be recreated on the destination file system.

**Safe diagnostics:** `find <SOURCE> -type s -print` and `find <SOURCE> \( -type s -o -type p -o -type b -o -type c \) -print`.

**Solution:** Identify the specific runtime socket and exclude **only** that path from copying. Do not exclude the whole `.git` or project directory.

**Verification:** Rerun `rsync` with the same filter and perform a checksum dry-run; see [backup](#backup).

### 3. Git fsmonitor socket

**Symptom:** Copying a repository stops at `.git/fsmonitor--daemon.ipc`.

**Cause:** This is an IPC socket of a running process, not a persistent part of repository history.

**Safe diagnostics:** `find <PROJECT>/.git -type s -print` and inspection of the indicated path.

**Solution:** Exclude the specific runtime socket; keep `.git`, branches, objects and worktree metadata.

**Verification:** Checksum dry-run with the same filter and Git checks from the [GitHub section](#github).

### 4. Interrupted copy to `.incoming`

**Symptom:** `rsync` ended with an error after copying some files.

**Cause:** Partial staging is the natural result of an interruption.

**Safe diagnostics:** Check the error message, `ls -ld <DEST>` and `rsync -acn --delete --itemize-changes <SOURCE>/ <DEST>/` with the appropriate filters.

**Solution:** Do not automatically delete `.incoming`. Fix the filter or the source of the problem and run `rsync` again; it will finish copying missing data.

**Verification:** The checksum dry-run shows zero differences, followed by SHA256 and any needed Git checks; see [backup](#backup).

### 5. Zsh: `PATH` suddenly contains one file

**Symptom:** `zsh: command not found: shasum`, `awk` or `find` after a loop.

**Cause:** In Zsh, `path` is a special array tied to `PATH`. Code such as `for path in ...` can replace the program search path.

**Safe diagnostics:** `print -r -- "$PATH"` and `typeset -p path` in the broken session.

**Solution:** Start a fresh session: `exec /usr/bin/env -u PATH /bin/zsh -l`. In scripts, use names like `FILE`, `NAME`, `ITEM` instead of `path`.

**Verification:** `command -v shasum awk find` points to programs, and the original command works.

### 6. `zsh: file exists: SHA256SUMS`

**Symptom:** Regenerating the manifest does not work.

**Cause:** The `noclobber` option blocks an ordinary `>` when the file already exists.

**Safe diagnostics:** `setopt | rg noclobber` and `ls -l SHA256SUMS`.

**Solution:** After confirming the correct directory, generate the manifest with the `>| SHA256SUMS` command from [verification](#verification). An old manifest after file changes may give a false mismatch.

**Verification:** `shasum -a 256 -c SHA256SUMS` returns `OK` for every item, and the entry count matches.

### 7. SHA256: `FAILED open or read` after moving

**Symptom:** The file exists at the new location, but the manifest check cannot open it.

**Cause:** The manifest contains an old absolute path.

**Safe diagnostics:** Read the path in the manifest and compare the recorded hash with `shasum -a 256 <DESTINATION>` for the corresponding file. Do not publish hashes of sensitive files.

**Solution:** If the recorded and actual hashes match, rebuild the manifest with relative paths as in [verification](#verification). If they differ, explain the discrepancy before changing the manifest.

**Verification:** From the snapshot directory, `shasum -a 256 -c SHA256SUMS` succeeds after moving.

### 8. `rsync` shows `.d..t.... ./`

**Symptom:** The dry-run reports only the root directory even though the files match.

**Cause:** The directory `mtime` differs; that code does not mean file contents differ.

**Safe diagnostics:** `rsync -acnO --delete --itemize-changes <SOURCE>/ <DEST>/`.

**Solution:** Use `-O` for content comparison; it omits directory timestamps. If the directory timestamp matters, inspect and correct it separately after analysis.

**Verification:** The dry-run with `-O` shows no file differences, and SHA256 succeeds; see [verification](#verification).

### 9. PlistBuddy: `Delete: Entry ... Does Not Exist`

**Symptom:** Deleting an entry fails even though the key is in the file.

**Cause:** Key names with spaces or parentheses may be misinterpreted by PlistBuddy syntax.

**Safe diagnostics:** Read the plist with Python `plistlib` and print **only key names**, without values that may contain secrets.

**Solution:** After preserving a copy, use `plistlib` to change one exactly identified key, write to a temporary file in the same directory, preserve permissions and only then replace the file. Do not delete the whole plist.

**Verification:** `plutil -lint <PLIST_FILE>` and reading the specific key; see [settings](#safari).

### 10. Removing an application: `Permission denied`

**Symptom:** Removing an application from `/Applications` is blocked.

**Cause:** The application may be owned by `root` or restricted by permissions and flags.

**Safe diagnostics:** `ls -ldOe "/Applications/<APP_NAME>.app"` and `stat -f 'owner=%Su group=%Sg mode=%Sp flags=%Sf' "/Applications/<APP_NAME>.app"`.

**Solution:** First confirm that it is the correct application, that its data and settings are backed up, and that the user deliberately wants to remove it. If `root` owns it and the permissions require it, use `sudo` **only** for the specific application path. No wildcards.

**Verification:** Check that this exact application is gone and other programs still work; see [security](#security).

### 11. A removed Safari extension still appears

**Symptom:** An old extension entry remains in the configuration.

**Cause:** The extension registry or plist still contains the identifier.

**Safe diagnostics:** `pluginkit -m -A -D` and a search for `<BUNDLE_IDENTIFIER>` in the relevant plist files without printing sensitive values.

**Solution:** Determine whether the identifier is stale; remove only the specific entry from the appropriate plist after preserving a copy. Do not delete whole plist files.

**Verification:** `plutil -lint <PLIST_FILE>`, reread the entry and check Safari; see [settings](#safari).

### 12. Two entries look like two extensions

**Symptom:** The system shows an application and an extension separately.

**Cause:** One application may contain an embedded extension:

```text
<APP_NAME>.app
└── Contents/PlugIns/<EXTENSION>.appex
```

**Safe diagnostics:** Check the application bundle structure and identifiers in `pluginkit -m -A -D`.

**Solution:** Treat the pair as an application and its embedded extension if the paths confirm it; do not remove one entry based solely on the number of entries.

**Verification:** The application and extension work, and the configuration points to the expected pair; see [Safari](#safari).

### 13. Two backups look similar

**Symptom:** It is unclear whether the older copy is a duplicate.

**Cause:** Names and sizes do not prove identical content.

**Safe diagnostics:** Calculate SHA256 for corresponding files or manifests for both directories.

**Solution:** The same SHA256 indicates duplicate file content; a different SHA256 indicates a unique version. Before reducing whole snapshots, also consider file lists, metadata and Git history.

**Verification:** Compare all required hashes, file counts and snapshot scopes; see [verification](#verification).

### 14. A project occupies several GB because of build and cache files

**Symptom:** The project backup is disproportionately large.

**Cause:** Build, cache or dependency directories contain automatically generated data.

**Safe diagnostics:** `du -sh <PROJECT>/*` and search for directories from [Projects](#projects).

**Solution:** After review, add precise exclusions only for reproducible data. A finished APK, IPA or other artifact may be the only copy and must not be excluded automatically.

**Verification:** The omission list is justified, the checksum dry-run succeeds with the same filters, and unique artifacts are present.

### 15. A local branch does not exist on GitHub

**Symptom:** `git clone` does not provide a branch visible locally.

**Cause:** The branch or its commits have not been pushed.

**Safe diagnostics:** `git -C <PROJECT> branch -a -vv`, `git -C <PROJECT> worktree list` and `git -C <PROJECT> log --branches --not --remotes --oneline`.

**Solution:** Before reducing `.git`, keep the complete repository and associated worktrees if anything exists only locally.

**Verification:** The copy shows the same branches, commits and worktrees; see [GitHub](#github).

### 16. `git fsck` shows a dangling commit/tree/blob

**Symptom:** The report contains `dangling commit`, `dangling tree` or `dangling blob`.

**Cause:** The objects are not reachable from current references; this does not automatically mean corruption.

**Safe diagnostics:** `git -C <PROJECT> fsck --full` and review of history and references.

**Solution:** Keep the complete `.git` in a historical backup. Do not delete dangling objects merely because they are dangling; they may hold the only version of data.

**Verification:** The copy passes the integrity check and retains the expected objects; see [GitHub](#github).

### 17. The manifest includes itself

**Symptom:** The entry count exceeds the number of data files, or manifest verification is unstable.

**Cause:** `SHA256SUMS` was included when calculating its own hash.

**Safe diagnostics:** Compare the entry count with the number of regular data files, excluding `SHA256SUMS`.

**Solution:** After checking the directory, regenerate the manifest:

```zsh
find . -type f ! -name SHA256SUMS -print0 \
  | sort -z \
  | xargs -0 shasum -a 256 >| SHA256SUMS
```

**Verification:** `shasum -a 256 -c SHA256SUMS` succeeds; with 100 data files there are 100 entries and 101 files in total.

### 18. Can `.incoming` be finalized?

**Symptom:** The copy looks ready, but its completeness is uncertain.

**Cause:** Completion of copying alone does not prove a match or repository correctness.

**Safe diagnostics:** Check in order: the source exists; staging exists; `rsync` ended without error; checksum dry-run shows zero differences; SHA256 passes; `git fsck` passes for critical repositories; file counts match.

**Solution:** Only after satisfying the checklist and preserving the previous final state, run `mv` from `.incoming` to the final name; see the [procedure](#backup).

**Verification:** The final directory exists, there is no competing partial copy, and the manifest succeeds from the new location.

### 19. Can the old source be deleted?

**Symptom:** Migration seems complete, but the old data uses space.

**Cause:** An apparent match may hide unique versions, branches, keys or artifacts.

**Safe diagnostics:** Confirm that `CURRENT`/`ARCHIVE` exists, SHA256 passes, source ↔ archive checksum passes, history is accounted for, duplicates are identified by hashes, and no unique branches, keys or artifacts remain.

**Solution:** Perform DELETE last, only for the precisely identified and verified old source, after a deliberate decision by the data owner.

**Verification:** Check that `CURRENT` and `ARCHIVE` still pass SHA256 and that needed data can be restored; see [restoration](#restore).

### 20. Quick message diagnostics

**Symptom:** One of the common messages below appears.

**Cause:** The table lists the most common causes; the specific case needs confirmation.

**Safe diagnostics:** Choose the appropriate row and run the indicated check before writing.

**Solution:** Apply the solution only after confirming the cause and the conditions in the relevant issue.

**Verification:** Repeat the check from the row, and for copies run [checksum and SHA256](#verification).

| Message / symptom | Most common cause | What to check | Solution |
| --- | --- | --- | --- |
| `Permission denied` with `rsync -E` | AppleDouble, ACL or xattr | Dry-run with `-E` and without `-E` | For code without required metadata, use `-a`; see issue 1. |
| `mkstempsock: Invalid argument` | Socket/runtime | `find <SOURCE> -type s -print` | Exclude only the identified socket; issues 2–3. |
| `command not found` after a Zsh loop | Overwritten `path`/`PATH` | `print -r -- "$PATH"` | New session and rename the variable; issue 5. |
| `file exists: SHA256SUMS` | `noclobber` | `setopt` and the correct directory | `>\| SHA256SUMS`; issue 6. |
| `.d..t.... ./` | Directory timestamp | Dry-run with `-O` | Check time separately from content; issue 8. |
| `FAILED open or read` | Old absolute path | Recorded and current hash | Relative manifest after confirming hashes; issue 7. |
| `Delete: Entry Does Not Exist` | Plist key syntax | Exact key name | `plistlib` and one entry; issue 9. |
| `Permission denied` when removing an application | Owner/flags | `ls -ldOe`, `stat` | Exact path, `sudo` if necessary; issue 10. |
| `dangling commit/tree/blob` | Object outside references | `git fsck --full` | Keep objects for assessment; issue 16. |
| Checksum dry-run shows differences | Missing file, different content or extra manifest | `rsync` codes, filters and file list | Explain every difference, rerun copying; issues 4 and 8. |

[↑ Back to contents](#index)

<a id="publishing"></a>
## What can safely be published on GitHub

**YES:** directory structure, example commands, placeholders, general principles, example exclusions and troubleshooting without private data.

**NO:** private keys, tokens, passwords, credentials, private IPs, UUIDs, personal paths, names of private projects and repositories, or contents of files that may contain secrets. Before `git add` and publication, review the file and the secret scanner results, and before a public push, also review commit history.

This file is the main public document. If additional public guides are created later, add links to them here; each should link back to `README.md`, and related documents should link to each other. Do not duplicate whole guides.

[↑ Back to contents](#index)

<a id="example"></a>
## Configuration example

**THIS IS ONLY AN EXAMPLE. All data is fictional.**

```text
<HOME> = /Users/alex
<BACKUP_VOLUME> = /Volumes/Backup
<GITHUB_USERNAME> = alex-dev
<REPOSITORY_NAME> = ExampleApp
<PROJECTS_DIR> = /Users/alex/Projects
<CATEGORY> = Projects
<YYYY-MM-DD> = 2026-01-15
```

The example result after substituting values shows the division between the latest state and a historical point:

```text
/Volumes/Backup/CURRENT/Projects/ExampleApp/
/Volumes/Backup/ARCHIVE/Projects/2026-01-15/
```

Before using this yourself, replace every value with your own, check that the volume is mounted and follow the [backup procedure](#backup).

[↑ Back to contents](#index)
