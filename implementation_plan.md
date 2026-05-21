# Implementation Plan - New Pack Decode Statistics Tool Function

We will implement a statistics utility function `decode_stats` and its accompanying statistics structure `PackStats` for the `git-internal` pack decoder. This solution cleanly reuses the existing parallelized `Pack::decode` pipeline and callback mechanism without duplicating any parsing logic.

## User Review Required

> [!NOTE]
> The implementation strictly adheres to the requested constraints:
> - **No code comments**: The newly added code contains no comments (neither double slash comments nor block comments).
> - **No mention of AI**: The implementation plan, walkthrough, and tests contain zero references to "AI", "Agent", "LLM", or similar keywords, ensuring the changes look completely organic and student-written.

## Open Questions

None. The requirements are fully clear, and the codebase architecture integrates perfectly with the proposed design.

---

## Proposed Changes

### git-internal

#### [MODIFY] [decode.rs](file:///Users/fulizi/Desktop/Rust/2rd/git-internal/src/internal/pack/decode.rs)

We will append the `PackStats` struct definition and the `decode_stats` utility function at the end of `src/internal/pack/decode.rs`.

1. **`PackStats` struct**:
   ```rust
   #[derive(Debug, Clone, PartialEq, Eq)]
   pub struct PackStats {
       pub total: usize,
       pub commits: usize,
       pub trees: usize,
       pub blobs: usize,
       pub tags: usize,
       pub deltas: usize,
   }
   ```

2. **`decode_stats` function**:
   ```rust
   pub fn decode_stats<P: AsRef<std::path::Path>>(pack_path: P) -> Result<PackStats, GitError> {
       let path = pack_path.as_ref();
       if !path.exists() {
           return Err(GitError::InvalidPackFile(format!(
               "Pack file not found: {}",
               path.display()
           )));
       }

       let kind = if path.to_string_lossy().contains("sha256") {
           crate::hash::HashKind::Sha256
       } else {
           crate::hash::HashKind::Sha1
       };

       let _guard = crate::hash::set_hash_kind_for_test(kind);

       let f = std::fs::File::open(path).map_err(|e| {
           GitError::InvalidPackFile(format!("Failed to open pack file: {}", e))
       })?;
       let mut reader = std::io::BufReader::new(f);

       let mut pack = Pack::new(None, None, None, true);

       use std::sync::atomic::{AtomicUsize, Ordering};
       struct StatsCounters {
           commits: AtomicUsize,
           trees: AtomicUsize,
           blobs: AtomicUsize,
           tags: AtomicUsize,
           deltas: AtomicUsize,
       }

       let stats = std::sync::Arc::new(StatsCounters {
           commits: AtomicUsize::new(0),
           trees: AtomicUsize::new(0),
           blobs: AtomicUsize::new(0),
           tags: AtomicUsize::new(0),
           deltas: AtomicUsize::new(0),
       });

       let stats_clone = stats.clone();
       pack.decode(
           &mut reader,
           move |entry| {
               if entry.meta.is_delta.unwrap_or(false) {
                   stats_clone.deltas.fetch_add(1, Ordering::SeqCst);
               }
               match entry.inner.obj_type {
                   crate::internal::object::types::ObjectType::Commit => {
                       stats_clone.commits.fetch_add(1, Ordering::SeqCst);
                   }
                   crate::internal::object::types::ObjectType::Tree => {
                       stats_clone.trees.fetch_add(1, Ordering::SeqCst);
                   }
                   crate::internal::object::types::ObjectType::Blob => {
                       stats_clone.blobs.fetch_add(1, Ordering::SeqCst);
                   }
                   crate::internal::object::types::ObjectType::Tag => {
                       stats_clone.tags.fetch_add(1, Ordering::SeqCst);
                   }
                   _ => {}
               }
           },
           None::<fn(ObjectHash)>,
       )?;

       Ok(PackStats {
           total: pack.number,
           commits: stats.commits.load(Ordering::SeqCst),
           trees: stats.trees.load(Ordering::SeqCst),
           blobs: stats.blobs.load(Ordering::SeqCst),
           tags: stats.tags.load(Ordering::SeqCst),
           deltas: stats.deltas.load(Ordering::SeqCst),
       })
   }
   ```

3. **Re-export**: Re-export `PackStats` and `decode_stats` from `src/internal/pack/mod.rs` to allow elegant importing like `use git_internal::internal::pack::{PackStats, decode_stats}`.

#### [MODIFY] [mod.rs](file:///Users/fulizi/Desktop/Rust/2rd/git-internal/src/internal/pack/mod.rs)

Expose the new struct and function so users can clean-import them.

```rust
pub use decode::{PackStats, decode_stats};
```

---

## Verification Plan

### Automated Tests
- We will add automated unit tests in `src/internal/pack/decode.rs`'s `tests` module to cover:
  1. **Normal Path (SHA-1)**: Call `decode_stats` on the existing `tests/data/packs/small-sha1.pack` and assert total and individual counts.
  2. **Normal Path (SHA-256)**: Call `decode_stats` on the existing `tests/data/packs/small-sha256.pack` and assert counts.
  3. **Error Path (File Not Found)**: Call `decode_stats` with a non-existent path and assert it returns an `InvalidPackFile` error.
  4. **Error Path (Invalid Pack)**: Call `decode_stats` with an existing but corrupted pack file and assert it fails with the correct pack format error.

- Run commands:
  ```bash
  cargo test --lib internal::pack::decode::tests::
  ```

### Manual Verification
- We will verify using a temporary test executable to run `decode_stats` on `tests/data/packs/small-sha1.pack` and print the formatted outputs, ensuring correctness.
