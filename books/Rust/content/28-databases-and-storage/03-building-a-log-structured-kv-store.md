---
title: Building a Log-Structured Key-Value Store (Mini-Bitcask)
---

# Building a Log-Structured Key-Value Store (Mini-Bitcask)

Nothing teaches storage-engine engineering faster than writing one from scratch. A **Bitcask-style** engine (the same architecture used in Riak and many embedded Rust engines) turns random key-value writes into fast sequential appends to a **Write-Ahead Log (WAL)** while keeping an in-memory `HashMap<String, LogPointer>` index that maps every live key to its exact byte offset on disk.

## Why Sequential Appends Beat Random Disk Writes

- **O(1) writes:** Appending `[key_len: u32][val_len: u32][key][val]` to the end of a file takes a single `write()` syscall + `sync_data()`.
- **O(1) reads:** One hash lookup in memory gives `(offset, len)`, followed by a single `seek` + `read_exact` from disk.
- **Crash recovery:** On startup, scanning the append-only log sequentially rebuilds the in-memory index. Truncating any partial trailing record handles power loss cleanly.
- **Compaction:** Because updates and tombstones (`val_len = u32::MAX`) append new records, background compaction rewrites only live keys into a fresh segment file and atomically swaps it in.

## Complete Runnable Mini-Bitcask Engine in Pure Rust

```rust
use std::collections::HashMap;
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Seek, SeekFrom, Write};
use std::path::Path;

pub struct MiniKv {
    file: File,
    index: HashMap<String, (u64, usize)>,
}

impl MiniKv {
    pub fn open(path: impl AsRef<Path>) -> io::Result<Self> {
        let mut file = OpenOptions::new()
            .create(true)
            .read(true)
            .append(true)
            .open(path)?;

        let mut index = HashMap::new();
        file.seek(SeekFrom::Start(0))?;

        loop {
            let mut hdr = [0u8; 8];
            if file.read_exact(&mut hdr).is_err() {
                break; // Reached EOF or incomplete tail record
            }
            let klen = u32::from_be_bytes(hdr[0..4].try_into().unwrap()) as usize;
            let vlen = u32::from_be_bytes(hdr[4..8].try_into().unwrap()) as usize;

            let mut kbuf = vec![0u8; klen];
            if file.read_exact(&mut kbuf).is_err() {
                break;
            }
            let val_offset = file.stream_position()?;
            if file.seek(SeekFrom::Current(vlen as i64)).is_err() {
                break;
            }
            let key = String::from_utf8_lossy(&kbuf).into_owned();
            index.insert(key, (val_offset, vlen));
        }

        Ok(Self { file, index })
    }

    pub fn set(&mut self, key: &str, val: &[u8]) -> io::Result<()> {
        let end = self.file.seek(SeekFrom::End(0))?;
        self.file.write_all(&(key.len() as u32).to_be_bytes())?;
        self.file.write_all(&(val.len() as u32).to_be_bytes())?;
        self.file.write_all(key.as_bytes())?;
        let val_offset = end + 8 + key.len() as u64;
        self.file.write_all(val)?;
        self.file.sync_data()?;
        self.index.insert(key.to_string(), (val_offset, val.len()));
        Ok(())
    }

    pub fn get(&mut self, key: &str) -> io::Result<Option<Vec<u8>>> {
        let Some(&(offset, len)) = self.index.get(key) else {
            return Ok(None);
        };
        self.file.seek(SeekFrom::Start(offset))?;
        let mut buf = vec![0u8; len];
        self.file.read_exact(&mut buf)?;
        Ok(Some(buf))
    }
}
```

> [!TIP]
> Production engines add a **CRC32 checksum** to every record header (`[crc32][klen][vlen][key][val]`) so silent bit-rot or half-flushed disk sectors are detected immediately during startup replay.

## Hands-On Lab: Verify Persistence Across Restarts

Add this `main()` to `MiniKv`, run it, and confirm that reopening the log file recovers the latest value for overwritten keys:

```rust
fn main() -> io::Result<()> {
    let path = "/tmp/rust-minikv.wal";
    let _ = std::fs::remove_file(path);

    {
        let mut db = MiniKv::open(path)?;
        db.set("user:1", b"alice")?;
        db.set("user:1", b"alice-v2")?;
        db.set("role:1", b"admin")?;
    } // File dropped and closed

    let mut reopened = MiniKv::open(path)?;
    assert_eq!(reopened.get("user:1")?, Some(b"alice-v2".to_vec()));
    assert_eq!(reopened.get("role:1")?, Some(b"admin".to_vec()));
    println!("Recovered {} keys from WAL cleanly!", reopened.index.len());
    Ok(())
}
```
