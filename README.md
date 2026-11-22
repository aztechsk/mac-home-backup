# mac-home-backup

A small, explicit, macOS-only command-line utility for **selective backup and
restore of personal files**. It is written in Perl and uses the macOS APFS
snapshot mechanism and Apple Archive (`/usr/bin/aa`).

A successful backup run produces **one compressed `.aar` file** in
`/private/tmp`. That file can then be copied to a separately managed server or
other storage. Restore works on explicitly selected files, symbolic links, or
whole directory trees; it does not attempt to reinstall macOS or reconstruct an
entire user account automatically.

## Why this tool exists

The intended recovery procedure is deliberately simple:

1. Install a clean copy of macOS, if necessary.
2. Install the required applications and configure the machine deliberately.
3. Recover selected personal files and selected application settings from
   archived backups.

For this workflow, a full machine migration is not the primary goal. A broad,
automatic restoration of `~/Library` may also bring back obsolete or unwanted
application state. Source repositories and other centrally maintained data can
have their own backup and recovery procedures.

The desired backup artifact is a single, portable-in-the-operational-sense
file: easy to identify, checksum, copy, retain, and retrieve from an ordinary
server. The server does not need to understand Apple Archive; archives are
expected to be **opened and restored on macOS**.

### Comparison with Time Machine

Time Machine is an excellent general-purpose macOS backup facility, but it
serves a different workflow.

| Property | mac-home-backup | Time Machine |
| --- | --- | --- |
| Primary goal | Deliberate selection and controlled restoration | Continuous, general-purpose Mac backup and recovery |
| Execution | Explicit CLI run | Automatic scheduled backups after setup; manual runs also available |
| History | One standalone archive per run; retention is external | Managed history of previous file versions |
| Coverage | HOME excluding Library, or an explicitly selected Library set | Broad user-data and application coverage with configurable exclusions |
| Output | One compressed `.aar` file in `/private/tmp` | Time Machine-managed backup destination |
| Off-Mac storage | User or external process copies the archive | Time Machine manages backups on a configured destination |
| File restore | Explicit `-p` paths; exact object/tree replacement | Interactive recovery of selected files and previous versions |
| Whole-account migration | Not provided | Supported through macOS Migration Assistant |
| Design emphasis | Small, inspectable CLI with explicit decisions | Convenience, automation, and recovery history |

**This is not a claim that Time Machine cannot restore individual files.** It
can. The distinction is control of scope, output format, and recovery workflow.
The two tools can be complementary; this utility is **not** a substitute for a
complete disaster-recovery strategy.

## Scope and requirements

- macOS with APFS snapshot support for the Data volume. The backup source is
  `/System/Volumes/Data`, accessed through a read-only snapshot.
- macOS-provided `/usr/bin/aa` and system utilities, plus Perl and the modules
  imported by `mac-home-backup` (including `JSON::PP`, `Digest::SHA`,
  `Unicode::Normalize`, `Encode`, `File::Temp`, and `File::Path`).
- A valid absolute `$HOME`; ordinary operation runs as the account owner,
  **without `sudo`**.
- Sufficient space for the compressed archive, temporary manifest, and (during
  restore) the selected extracted objects and the old objects retained for
  rollback.
- Access to the archive on the Mac performing the restore.

The program does **not** provide encryption, scheduling, incremental/differential
backup, archive retention, automatic upload, network transfer, a GUI, a custom
restore destination option, or bare-metal/system recovery. It does not promise
cross-platform archive readability. There is no automatic merge of restored
directories.

`/private/tmp` is **temporary staging**, not durable backup storage. Its
contents may disappear during restart or system cleanup. **Copy successful
archives to separate, persistent storage promptly**; an archive left only on
the source Mac is not an off-device backup.

## Quick start

The program has one mandatory operation switch. There is no default operation.

```sh
# Backup: exactly one of these modes
./mac-home-backup -b -m home
./mac-home-backup -b -m library

# Restore: provide the archive and one or more selected objects
./mac-home-backup -r \
    -a /path/to/host-user-home-20261119070033-0123456789abcdef.aar \
    -p 'Documents/example.txt' \
    -p 'Projects/example-project'

# Help and version
./mac-home-backup -h
./mac-home-backup -v
```

The archive filename in the restore example is illustrative, not an existing
file. `--help` and `--version` are also recognized.

### Backup configuration

Backup requires the UTF-8 text file `~/.mac-home-backup`. Restore does not read
this configuration file. Repeated keys are supported; only these keys exist:

```ini
# Relative to $HOME; Library is always excluded automatically.
home_exclude = Downloads
home_exclude = Movies
home_exclude = .Trash

# Relative to $HOME/Library; include only these objects/trees.
library_include = Application Support/Sublime Text
library_include = Preferences/com.sublimetext.4.plist
```

Configuration paths are literal relative paths, not glob patterns. They must
not begin with `/` or contain empty, `.` or `..` components. The program
resolves components against the actual snapshot filesystem spelling and
rejects traversal through symlink ancestors. A configured path that is absent
is an error rather than a silently skipped backup item.

- **`-b -m home`:** traverse `$HOME`, always exclude `Library`, then apply each
  `home_exclude` to a complete file or directory subtree. All other accessible
  supported objects are candidates for the archive.
- **`-b -m library`:** traverse only the listed `library_include` objects
  relative to `$HOME/Library`. An included directory brings its descendants;
  an included file does not automatically include its parent directory as an
  independently archived object. At least one include is required.

Special objects (FIFOs, sockets, character and block devices) are skipped.
Ordinary files, directories, and symbolic links are supported. The source
preflight does not follow symlinks while walking the tree.

### Restoring files

```sh
./mac-home-backup -r -a "$archive" -p 'Documents/notes.txt'
```

### Restoring a complete directory subtree

```sh
./mac-home-backup -r -a "$archive" -p 'Projects/example-project'
```

**Warning:** selecting an archived directory **replaces the entire existing
directory tree**. Files already in that destination that are not in the
archive are removed as part of replacing the old tree. This is not an overlay,
a merge, or an import operation. Select only what you intend to replace.

### Restoring selected Library entries

```sh
./mac-home-backup -r \
    -a "$library_archive" \
    -p 'Preferences/com.sublimetext.4.plist' \
    -p 'Application Support/Sublime Text'
```

`-m` is **not** accepted with `-r`. The archive filename determines whether
`-p` paths are resolved under `$HOME` or `$HOME/Library` (details below).

## 1. Archive creation path

The backup side is intentionally separate from the restore side.

### 1.1 Source consistency: an APFS snapshot

The backup sequence, simplified from `stage_backup()` and
`with_apfs_snapshot()`, is:

1. Parse `~/.mac-home-backup` and validate the chosen `home`/`library` mode.
2. Create an APFS snapshot of `/System/Volumes/Data` with `tmutil localsnapshot`.
3. Mount **that snapshot read-only**, not the mutable live HOME.
4. Resolve configured paths to their real spelling on the snapshot, reject
   unsafe ancestor traversal, and check access to the selected filesystem
   objects.
5. Produce a temporary Apple Archive from the snapshot.
6. List the resulting archive, validate its path/type namespace using the
   restore rules, and compare its entry count against the source preflight count.
7. Build an Apple Archive manifest **from the archive**, then `aa verify` it
   against the same mounted snapshot using the same selection filters.
8. Remove the temporary manifest; unmount and delete the snapshot.
9. Compute SHA-256 of the finished archive and publish it under its final
   filename in `/private/tmp`.

The program attempts snapshot unmount/deletion and temporary-file cleanup on
operational errors and supported interruptions. The snapshot makes the source
view stable during the backup; it does not make the `/private/tmp` output
permanent.

### 1.2 Apple Archive format and attributes

The archive is created with `/usr/bin/aa archive`, using **LZFSE** compression,
a 4 MiB block size, four workers, hole support, and an explicit field set:

```text
typ,pat,lnk,dev,uid,gid,mod,flg,mtm,ctm,btm,xat,acl,dat,siz,duz,sh2,hlc
```

Notable fields and their purpose:

| Field(s) | Role |
| --- | --- |
| `TYP`, `PAT` | Filesystem object type and archive-relative path |
| `DAT`, `SIZ`, `DUZ`, `SH2` | File data, size/storage information, SHA-256-related data field |
| `LNK`, `HLC` | Symbolic-link target and hard-link cluster metadata |
| `UID`, `GID`, `MOD`, `FLG` | Ownership, permissions, and filesystem flags |
| `MTM`, `CTM`, `BTM` | Modification, creation, and backup time metadata |
| `XAT`, `ACL` | Extended attributes and access-control lists |
| `DEV` | Device-related archive metadata |

The field list expresses what the program asks Apple Archive to store; it is
**not a guarantee** that every attribute can be reproduced under every user
account, filesystem, or macOS version. Special object types are excluded with
`-exclude-type spbc`.

The same HOME exclude and Library include filters are used both for archive
creation and for verification. Filters are component-boundary-aware regular
expressions: `^<escaped-path>(/|$)`. Their path arguments are based on actual
source entries, not unresolved user text.

### 1.3 What backup verification proves

`aa convert -manifest` creates a temporary manifest from the newly created
archive. `aa verify` checks that manifest against the read-only snapshot.
This detects missing or differing archive-described filesystem entries and
metadata according to the verification fields and filters.

**Important limit:** this is not an independent read/decompress/check of every
compressed `DAT` payload in the output archive. That extra full archive
readback is deliberately not performed during each backup. On restore, the
selected extracted content is verified after installation.

### 1.4 Output naming and SHA-256

A completed archive is named:

```text
<LocalHostName>-<user>-<home|library>-<YYYYMMDDHHMMSS>-<sha256-prefix>.aar
```

- The timestamp is local time in `YYYYMMDDHHMMSS` form.
- `<sha256-prefix>` contains the first **16 hexadecimal characters** of the
  SHA-256 digest of the complete archive file.
- The program prints the **full** 64-character SHA-256 digest on success.
- Publication uses a hard link followed by removal of the staging name, so
  an existing final filename is not overwritten.
- There is **one** published `.aar` artifact per successful run, not a
  manifest-plus-archive pair.

The filename digest helps detect accidental corruption, truncation, or a bad
copy. It is **not a signature**: someone able to modify the archive and rename
it can also replace the digest prefix. No encrypted or authenticated archive
container is provided by this utility.

## 2. Restore path and selection semantics

### 2.1 Destination root is derived from the archive name

`check_restore_archive()` accepts a filename of the form above and extracts
its mode with an anchored parser. Restore has no independent mode option:

| Archive mode in filename | Root used for restore |
| --- | --- |
| `home` | `$HOME` |
| `library` | `$HOME/Library` |

A HOME archive and a Library archive therefore have **different namespaces**.
For example, a Library selector is
`Preferences/com.sublimetext.4.plist`, **not**
`Library/Preferences/com.sublimetext.4.plist`. A HOME selector does not have
an explicit `$HOME/` prefix.

The archive's host and account labels identify the source in the filename, but
are not an authorization mechanism; this implementation does not require them
to equal the destination machine or account name. The destination is the
**current** `$HOME`, so moving an archive to another Mac calls for deliberate
review of paths, ownership, permissions, and selected state.

### 2.2 Recorded `PAT` versus an implied parent directory

Archive entries are obtained using:

```sh
/usr/bin/aa list -i "$archive" \
    -list-format json -exclude-field all -include-field pat,typ
```

The restore index accepts only recorded object types:

- `D` -- directory;
- `F` -- regular file;
- `L` -- symbolic link.

In observed HOME archives, the first record is a root directory with
`PAT=""` and `TYP="D"`. The code accepts this special first record but does
**not** expose it as a selectable `-p` path. Other selectable paths must be
nonempty, relative and valid.

**An existing `PAT` record is required for every requested `-p`.** A directory
is not selectable merely because it is the common prefix of archived files.
This matters most for selective Library archives:

```text
PAT: Preferences/com.example.one.plist        (F)
PAT: Preferences/com.example.two.plist        (F)
PAT: Application Support/Example App          (D)

No PAT: Preferences
No PAT: Application Support
```

In that archive:

```text
-p 'Preferences/com.example.one.plist'   ACCEPT
-p 'Application Support/Example App'     ACCEPT; restores its subtree
-p 'Preferences'                         REJECT; no recorded directory PAT
-p 'Application Support'                 REJECT; no recorded directory PAT
```

Why such strictness? `aa extract -include-path Preferences` can extract the
matching children **even when `Preferences` has no recorded PAT**. Treating
that implied parent as a selectable directory and replacing the live
`~/Library/Preferences` would delete unrelated preferences. The index must
reject this request **before extraction**, rather than infer an archive
object from pathname prefixes.

### 2.3 Exact object/tree replacement rules

The `-p` argument is a complete relative filesystem path, not a basename
search or glob. Its meaning is:

| Recorded type | What is selected and installed |
| --- | --- |
| `F` | That one regular file |
| `L` | That one symlink (the link itself, not its referent) |
| `D` | That recorded directory and all of its archived descendants |

A directory replacement is exact, never an overlay. An existing destination
may have a different object type; the selected archived object replaces it
where safe. A selected target may also be absent. Its **parents**, however,
must already exist as actual directories.

Multiple `-p` arguments are supported, but duplicate or overlapping requests
are rejected. For example, `-p a -p a/b` is invalid if `a` covers `a/b`.
Independent files or trees can be selected together. A shared text prefix is
not a path boundary: `a` and `a-old` are different objects.

### 2.4 Path syntax validation

Both requested and archived paths are checked. Selectable paths must be
relative, nonempty, and have no empty, `.` or `..` component; they must not
contain unsafe control characters or invalid Unicode. The archive index
rejects duplicate `PAT` records, unsupported types, Unicode-equivalent or
case-equivalent collisions, and any path whose explicit parent is a file or
symlink rather than a directory.

Implicit parents are indexed for collision detection as well. For example,
`Prefs/a.txt` and `prefs/b.txt` cannot coexist as two different implied
parents on a case-insensitive filesystem. This check applies to implied parents as well as explicit archive entries.

### 2.5 Unicode: NFC, NFD and the actual archive spelling

Unicode normalization must not be confused with filesystem name matching:

- **NFC and NFD** may represent the same visible characters with different
  Unicode code-point sequences and different UTF-8 bytes.
- In real macOS testing, `/usr/bin/aa -include-path` selected an archived NFC
  name when passed NFC, but returned an empty list when passed the equivalent
  NFD spelling. **Both commands exited with code 0.** The corresponding
  `aa extract` NFD test also extracted nothing and returned code 0.
- `mac-home-backup` therefore decodes the `-p` argument as UTF-8, uses
  `Unicode::Normalize::NFC()` to find the unique matching archived record,
  then passes the **archive's actual `PAT` spelling** to `aa extract` and
  `aa verify`.

This is a deliberate, documented interpretation of exact selection:
**the user may supply a canonically equivalent Unicode spelling, but it must
resolve to one actual, recorded `PAT`**. Case folding is *not* used for user
selection. Ambiguous canonically equivalent records are rejected during
indexing.

A real end-to-end restore with an NFD `-p` and an NFC archived filename
succeeded, including post-install verification and content comparison.

### 2.6 Case-insensitive APFS and disk spelling

Default macOS APFS is commonly **case-insensitive but case-preserving**;
APFS also offers a case-sensitive variant. On the tested Mac, `CaseProbe` and
`caseprobe` resolved to the same device and inode. Consequently,
`lstat("root.txt")` alone does not prove that the directory entry is actually
spelled `root.txt`; it could be `Root.txt`.

For an existing destination, the program now compares each path component
against names returned by `readdir()`:

- Unicode canonically equivalent spellings are allowed (`NFC` comparison).
- A case-only alias is **rejected**, not silently renamed or overwritten.
- A genuinely absent final object is allowed, so a deleted file can be
  restored.
- A symlink at the final target is allowed; an intermediate symlink is not.

The destination-spelling control runs during preflight and is rechecked
before and after installation. On real APFS, the full restore correctly
refused `Root.txt` when the archive specified `root.txt`; an NFD-input Unicode
restore still succeeded afterwards.

The archive's case-fold check helps detect ambiguous namespaces; it is not a
claim that Perl Unicode casefold reproduces every possible APFS naming rule
on all macOS versions.

### 2.7 `aa -include-path` uses path-component boundaries

A static-audit concern was that selecting `main.c` might also match
`main.c.test18-save` by plain text prefix. Testing against an archive that
actually contained **both** entries showed otherwise:

- `aa list -include-path .../main.c` listed only `main.c`.
- `aa extract -include-path .../main.c` extracted only `main.c`.

For the tested macOS `aa`, selection honored the path-component boundary. The
post-extraction staging inventory is kept as an independent safeguard.

## 3. Restore execution: from archive to installed object

The restore uses the following sequence (primarily `restore_preflight()`,
`restore_selected()`, and `restore_replace_one()`):

1. **Validate the input archive:** require a nonempty regular file, reject an
   input symlink, compare path and opened-file metadata, and validate the
   filename format and mode.
2. **Check archive integrity:** calculate the full archive SHA-256 and compare
   its first 16 hex digits with the filename suffix.
3. **Build the index:** run `aa list` for `PAT/TYP`; validate all records,
   actual selectors, duplicate/overlap rules, ancestor types, and Unicode/case
   collisions.
4. **Preflight the destination:** determine `$HOME` or `$HOME/Library`, verify
   all selected targets' existing ancestor directories with `lstat()`, reject
   missing/non-directory/symlink ancestors, and check real disk spelling.
5. **Record state and make private staging:** capture target state, create a
   private `.mac-home-backup-restore-XXXXXXXX` directory under the selected
   destination root, then build a temporary archive manifest in
   `/private/tmp`.
6. **Extract once:** run `aa extract -i ARCHIVE -d STAGE -no-ignore-eperm`
   with one `-include-path` for each selected archived `PAT`.
7. **Check staging:** walk the extracted tree without following symlinks;
   require exactly the selected recorded objects and permitted implicit parent
   directories, with matching `D/F/L` types and no extra or missing records.
8. **Replace and verify each selected root separately:** reserve a private
   sibling directory next to its target, move the original there if present,
   move the extracted object into the final name, verify the installed object
   against the archive manifest, and delete the saved original only after
   verification succeeds.
9. **Cleanup:** remove the temporary manifest and leftover staging. Report
   errors rather than falsely reporting a fully successful restore.

The staging root is created on the **same filesystem as the target parent**;
this is checked using device identities before relying on `rename()`. Private
staging and reservation directories use restrictive permissions. No implicit
parent directories created *inside staging* are installed at the destination.

### 3.1 Parent-directory policy

For `-p x/y/file`, the existing destination root and `x` and `x/y` must be
real directories. The program will not create missing parent directories,
even if those names appear as separate `PAT` records in the archive.

If `x` is absent, choose the recorded `-p x` **if it is a directory** and you
intend to replace that whole subtree. You cannot select both `x` and `x/y`
in the same command because their selections overlap.

Symlinks are never followed as destination ancestors. The *target object*
itself may be a symlink and can be replaced as an object, not through its
referent. The real-macOS test suite confirmed rejection of a symlink parent,
a missing parent, and an ordinary file used as a parent.

### 3.2 Post-install verification

A fresh temporary manifest is generated using:

```sh
/usr/bin/aa convert -manifest -i "$archive" -o "$manifest"
```

After installation, each selected root is verified separately:

```sh
/usr/bin/aa verify -v -i "$manifest" -d "$destination_root" \
    -include-path "$archive_pat" -exclude-field xat
```

Verification is against the **final installed path**, not just the private
extraction staging. It checks the selected archived structure/content and
non-excluded metadata. Real tests detected modified data (`DIFFER ... (DAT)`),
new/unexpected files, and directory metadata changes.

**`-exclude-field xat` excludes ALL extended attributes from this verification,
not just `com.apple.provenance`.** This exception was accepted because macOS
can change the provenance attribute during extraction. The extractor may
preserve xattrs, but this verification does **not** attest to their integrity.
Do not claim an exhaustive metadata guarantee from `aa verify` success.

The extractor uses `-no-ignore-eperm`: permission failures are not knowingly
silenced. UID/GID, ACL, flags, xattrs, resource forks, and cross-Mac restores
still depend on filesystem capabilities, account privileges, and macOS
behavior. Their full portability has not been established by the existing
tests.

### 3.3 Replace, commit and rollback

For one selected object, the intended progression is:

```text
existing TARGET (if any)
    | rename
    v
private sibling vault/original

extracted STAGE/PAT
    | rename
    v
TARGET
    | aa verify against archive manifest
    +-- success --> remove vault/original, then vault; object committed
    |
    +-- failure --> move unverified TARGET aside and restore vault/original
```

If the target did **not** exist, a verification failure removes the unverified
new object from the target location and restores the original *absent* state.
On rollback failure, the program reports the failure and intentionally retains
the reservation directory for manual recovery; do not delete that directory
until its contents have been inspected.

After successful verification, removal of the saved old object is post-commit.
If that cleanup fails, the program reports an error but must **not** pretend
that the newly verified object was rolled back.

The two `rename()` operations are **not a single atomic transaction**. A crash
or power loss between them is not automatically recovered. `SIGINT` handling
protects the tested commit/cleanup boundaries, but does not convert the
sequence into a crash-proof transaction.

**Multiple selections are not a global transaction.** Each `-p` selection is
installed and verified on its own. A later failure does not undo an earlier
successfully committed selection.

## 4. Security model and operational contract

The utility favors a small, inspectable implementation. Its safety checks are
specific rather than a general-purpose filesystem isolation system.

### Implemented safeguards

- Strongly constrained CLI modes and relative path syntax.
- Archive filename parsing and a SHA-256 corruption check before extraction.
- Full archive `PAT/TYP` validation before any selected target is changed.
- Duplicate, overlapping, non-directory-ancestor, Unicode, and case-collision
  rejection within the archive namespace.
- Real destination name-spelling checks on case-insensitive filesystems.
- `lstat()` ancestor validation and identity rechecks; no symlink ancestors.
- Private staging and per-object old-object reservations on the target
  filesystem; explicit staging inventory check.
- Verification after installation and rollback attempts on failure.
- Checked filesystem operations, external-command exit status, and cleanup.
- No shell-string interpolation for external command arguments.

### Deliberate limits and trust assumptions

- **No concurrent modification contract:** while restore runs, the user is
  responsible for keeping the selected destination objects and the input
  archive unchanged. The tool does limited state/identity rechecks, but does
  not detect every in-place write to a file or change under a directory.
  It does not offer a universal TOCTOU or multi-process transaction guarantee.
- **Not a sandbox for hostile archives:** validation and staging checks reduce
  risks, but checking extracted files *after* `aa extract` is not the same as
  containing the extractor in an OS sandbox. Exhaustive malicious-archive
  tests are not complete. Use archives from trusted sources and understand
  that the filename digest does not authenticate a sender.
- **No artificial size limits:** entry-index memory consumption and extracted
  disk usage scale with archive size. Normal resource errors must be handled;
  a fixed RAM/entry/expansion quota is intentionally not implemented.
- **No full xattr verification:** `XAT` is explicitly ignored in restore
  verification as described above.
- **No cross-Mac metadata guarantee:** restore under a different account or
  on another Mac may fail or reproduce only a subset of privileged metadata.
- **No automatic crash recovery or global transaction:** already described in
  the rollback section.
- **No automatic restore of missing ancestors:** a safety policy, not an
  unfinished feature.

Treat a reported `RC=0` as successful completion of the program's implemented
checks, **not** proof of cryptographic authenticity, full metadata fidelity,
or a race-free operation under concurrent writes.

## 5. Developer and auditor reference

| Function | Responsibility |
| --- | --- |
| `parse_options()` / `main()` | Exclusive backup/restore CLI, dispatch, top-level errors |
| `parse_config()` | UTF-8 backup configuration and path syntax |
| `stage_backup()` | Snapshot-based backup lifecycle and final publication |
| `with_apfs_snapshot()` | Snapshot acquisition, read-only mount, cleanup |
| `resolve_filter_paths()` / `resolve_source_path()` | Real source-path spelling and safe backup selection |
| `check_source_access()` / `check_source_tree()` | Source traversal and counts; special types skipped |
| `aa_filter_args()` / `archive_snapshot()` | Apple Archive selection filters and archive fields |
| `check_staged_archive()` / `publish_staged_archive()` | Structural/count check and no-overwrite publication |
| `restore_preflight()` | Archive, digest, index, selected paths, destination preflight |
| `check_restore_archive()` / `check_restore_archive_digest()` | Input file and filename integrity checks |
| `restore_archive_index()` / `validate_restore_path()` | Archive `PAT/TYP` namespace and ancestor validation |
| `select_restore_paths()` | Unique selected archive records, NFC-equivalent input handling |
| `check_restore_target_parents()` | Required parent directories, symlink and identity checks |
| `check_restore_target_spelling()` | Exact on-disk case, NFC-equivalent Unicode matching |
| `restore_selected()` | Private staging, one extraction, independent installation units |
| `check_restore_stage()` | Exact extracted-object inventory and type checks |
| `restore_replace_one()` / `restore_rename()` | Per-object swap, verification, rollback, reservation cleanup |
| `restore_path_state()` / `assert_restore_state()` | Limited change detection between stages |
| `assert_restore_identity()` | Device/inode/type identity checks |
| `remove_restore_object()` | Non-symlink-following cleanup for discarded objects |
| `exec_cmd()` / `run_cmd()` | Shell-free child execution, status/signal handling |

The source keeps the main user-facing path in a single Perl script. There is
no separate restore daemon, transfer client, archive database, or persistent
manifest index.

## 6. Troubleshooting and recovery notes

| Symptom | Meaning / action |
| --- | --- |
| `Exactly one operation is required` | Use one of `-b` or `-r`, not both or neither |
| `Requested restore path ... has no archive PAT record` | Request the recorded file/subtree, not an implied directory; inspect `aa list` |
| `Restore destination spelling mismatch` | Existing disk entry differs in case from archived `PAT`; inspect/rename manually only after deciding what is correct |
| `Restore destination parent ...` error | An ancestor is absent, symlinked, or not a directory; no ancestor will be created implicitly |
| `Restore archive SHA-256 mismatch` | Archive or filename no longer matches its checksum suffix; do not use the file until investigated |
| `Restore rollback FAILED` | Inspect the **reported retained reservation path** immediately; do not erase it or repeat a destructive restore blindly |
| Successful verify with `ignore fields: XAT` | Selected non-XAT fields passed verification; this does **not** check extended attributes |
| Archive missing after restart | `/private/tmp` is temporary storage; retrieve a previously transferred durable copy |

Before deliberately restoring an existing directory, inspect the archive's
contents and ensure no application is modifying the destination. Prefer a
fresh, clean macOS installation plus selected restoration if that is your
intended recovery workflow.

## References

- Apple Support, [Back up your Mac with Time Machine](https://support.apple.com/en-us/104984)
- Apple Support, [Restore your Mac from a backup](https://support.apple.com/en-gb/102551)
- Apple Developer, [Apple Archive](https://developer.apple.com/documentation/applearchive)
- Apple Archive `aa(1)` command documentation (also available locally with
  `man aa` on macOS)

---

**Design principle:** keep backup artifacts simple and independent, make
restoration explicit, verify what is installed, and avoid unnecessary machinery
for scenarios outside this tool's stated operational contract.
