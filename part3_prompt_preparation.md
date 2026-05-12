# Selected PR for Prompt Preparation: beets/pull/3478 (Fetchart Concurrency Fix)

## 3.1.1 Repository Context

`beetbox/beets` is an extensible command-line music library manager developed in Python. The application helps users organize audio collections, automatically fetch metadata from sources such as MusicBrainz and Discogs, and arrange files according to customizable directory templates. The project is built around a SQLite-based library model and a flexible plugin system that allows additional functionality through event-driven hooks.

### Intended Users

The software is mainly used by music collectors, archivists, DJs, and developers who need an automated and scriptable way to manage large music libraries. While some users maintain small personal collections, others manage very large archives containing terabytes of media files.

### Problem Domain

The main focus of beets is reliable digital asset management. The project emphasizes correctness, data safety, and non-destructive file handling. Since imports and metadata updates may happen simultaneously, the system must safely manage concurrent file operations without corrupting user data or causing inconsistent library states.

### Architecture Overview

The repository follows a layered architecture:

- **UI Layer**
  - `beets.ui` handles command-line interaction and user prompts.

- **Application Layer**
  - `beets.importer` manages the import workflow, including metadata fetching and user selection logic.

- **Domain Layer**
  - `beets.library` models albums and tracks.
  - `beets.autotag` communicates with metadata providers.

- **Infrastructure Layer**
  - `beets.util` contains filesystem utilities, database helpers, and OS-specific functionality.

- **Plugin System**
  - Plugins extend `beets.plugins.BeetsPlugin` and register listeners for events such as:
    - `import_task_choice`
    - `album_imported`
    - `write`

The `fetchart` plugin, located in `beetsplug/`, is responsible for downloading and managing album artwork from multiple sources.

This pull request mainly affects both the plugin layer and the infrastructure layer because it introduces safer concurrent filesystem operations while updating the artwork download workflow.

---

## 3.1.2 Pull Request Description

### Specific Changes Introduced

This pull request improves concurrency safety in the `fetchart` plugin by introducing secure temporary file handling and protected atomic file moves.

The previous implementation generated temporary filenames using a simple hash-based approach, which often produced collisions when multiple imports happened at the same time. The PR replaces this logic with `tempfile.mkstemp()`, creating unique temporary files that include the album database ID in the filename.

Additionally, a new utility called `atomic_move_with_lock` is introduced inside `beets/util`. This helper performs atomic file moves while holding an operating-system-level lock on the destination path.

### Why These Changes Were Needed

Earlier, temporary artwork files were generated using:

```python
hash(str(album)) % 10000
```

Because of the limited randomness in this approach, concurrent imports could accidentally use the same temporary filename. This caused several issues, including:

- Artwork files being overwritten
- `OSError` exceptions during file moves
- Incorrect album art assigned to albums
- Race conditions during parallel imports

These problems became more common in automated setups where multiple beets processes imported albums simultaneously.

### Previous vs New Behavior

#### Previous Behavior

- Temporary artwork files used predictable paths such as:
  ```text
  /tmp/1234.jpg
  ```
- Multiple imports could write to the same file.
- Final file moves had no locking mechanism.
- Parallel imports sometimes failed or produced corrupted artwork.

#### New Behavior

- Temporary files are created with:
  ```python
  tempfile.mkstemp()
  ```
- Paths now look similar to:
  ```text
  /tmp/beets-42-xxxxxx.art
  ```
- Final artwork moves are protected using OS-level file locks.
- If a lock cannot be obtained, beets logs a warning and safely falls back to sequential behavior instead of crashing.

The changes improve reliability while maintaining backward compatibility and avoiding additional dependencies.

---

## 3.1.3 Acceptance Criteria

### Unique Temporary Paths

- Album artwork downloads must use `tempfile.mkstemp()`.
- Temporary filenames should contain the album database ID.
- Tests should mock `mkstemp()` and verify that generated prefixes include the correct album ID.

### Atomic Move with Lock

- `atomic_move_with_lock()` must lock a `.lock` file near the destination path before moving files.
- The system should correctly handle lock contention situations and gracefully retry or fall back when necessary.

### Cross-Platform Compatibility

- POSIX systems should use `fcntl.lockf`.
- Windows systems should use `msvcrt`-based locking or an equivalent fallback.
- CI tests must pass on Linux, macOS, and Windows environments.

### Resource Cleanup

- File descriptors returned by `mkstemp()` must always be closed.
- Temporary files should be deleted if an exception occurs during processing.
- Tests should confirm that no descriptor leaks remain after failed operations.

### Backward Compatibility

- Existing `fetchart` configurations must continue working without changes.
- Users not using parallel imports should observe identical behavior to earlier versions.
- Existing tests in `test/test_fetchart.py` should continue passing.

---

## 3.1.4 Edge Cases

### 1. Exhausted Temporary Directory

Under heavy concurrency, the system temporary directory may run out of available space or inodes, causing `mkstemp()` to fail.

The implementation should:

- Catch `OSError`
- Log a clear warning message
- Skip artwork fetching for that album without terminating the entire import process

---

### 2. Read-Only or Network Filesystems

Some users may store music libraries on read-only drives or network-mounted filesystems where file locking behaves differently.

The implementation should:

- Catch `PermissionError` and `OSError`
- Warn the user if locking is unavailable
- Attempt a best-effort atomic rename operation without crashing

---

### 3. Unicode and Path Encoding Issues

Album metadata may contain non-ASCII characters that create filesystem compatibility issues on some systems or NFS mounts.

The implementation should:

- Sanitize temporary filename prefixes
- Restrict prefixes to safe ASCII characters where necessary
- Handle `UnicodeEncodeError` exceptions during path generation

---

## 3.1.5 Initial Prompt

### Prompt

You are tasked with implementing a concurrency safety fix for the `fetchart` plugin in the `beetbox/beets` music library manager.

---

### Repository Context

Beets is a Python-based command-line application for managing music collections. The project uses a plugin architecture where the `fetchart` plugin (`beetsplug/fetchart.py`) downloads album artwork from multiple sources and stores it in the user’s library.

Currently, temporary artwork files are generated using a weak hash-based naming approach:

```python
hash(str(album)) % 10000
```

This creates race conditions and filename collisions during concurrent imports.

Core filesystem utilities are located in:

- `beets/util/__init__.py`
- `beets/util/fshandler.py`

The import workflow is managed by:

- `beets/importer.py`

The project intentionally avoids external dependencies beyond the Python standard library and `unidecode`.

---

### Task Requirements

#### 1. Refactor Temporary File Creation in `beetsplug/fetchart.py`

- Replace manual temporary filename generation.
- Use `tempfile.mkstemp()` for secure temporary files.
- Include the album database ID in the filename prefix:
  ```text
  beets-{album.id}-
  ```
- Ensure all file descriptors are properly closed.

---

#### 2. Add `atomic_move_with_lock` in `beets/util/__init__.py`

Implement:

```python
atomic_move_with_lock(src, dst)
```

Requirements:

- Perform atomic file moves while holding an OS-level lock.
- POSIX:
  - Use `fcntl.lockf`
- Windows:
  - Use `msvcrt` locking or a standard-library alternative
- Add a timeout mechanism (recommended: 5 seconds).
- If locking fails:
  - Log a warning
  - Fall back to a normal move operation
- Ensure cleanup using `try...finally`.

---

#### 3. Update `beets/util/fshandler.py`

- Add internal platform-specific locking helpers.
- Prefix helper names with `_` to keep them private.

---

#### 4. Testing Requirements

##### `test/test_fetchart.py`

- Simulate concurrent downloads for the same album.
- Verify:
  - Temporary paths are unique
  - `atomic_move_with_lock()` is called

##### `test/test_util.py`

- Verify:
  - Locks are acquired and released correctly
  - No `.lock` files remain after execution

---

#### 5. Documentation

Update:

```text
docs/plugins/fetchart.rst
```

Document:

- Safer parallel imports
- Atomic artwork installation behavior

---

### Constraints and Edge Cases

- Do not add external dependencies.
- Support both POSIX and Windows systems.
- Handle read-only or unsupported network filesystems gracefully.
- Preserve backward compatibility.
- Do not modify the public plugin API or configuration schema.

---

### Acceptance Checklist

- [ ] `tempfile.mkstemp()` is used with album-ID prefixes.
- [ ] `atomic_move_with_lock()` supports both POSIX and Windows.
- [ ] Tests verify locking and path uniqueness.
- [ ] No new external dependencies are introduced.
- [ ] CI passes on Linux, macOS, and Windows.

Provide the implementation as either:

- Git diff patches, or
- Complete modified file contents.
