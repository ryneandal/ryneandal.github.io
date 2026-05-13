---
title: "Exporting Book Metadata from Apple Books"
date: 2026-05-07
categories: 
  - macOS
  - Apple Books
  - metadata
  - export
  - Rust
excerpt: >
  I've always used Apple's Books (formerly iBooks) for my reading. One issue I've always run into is no export functionality. I wanted a list of books I already owned, and there was no easy way to do it since I always purchase DRM-free ebooks. This post describes how and why I wrote a simple program to export the metadata for my books.
author: Ryne Andal
tags: ["macOS", "Apple Books", "metadata", "export", "Rust"]
canonical_url: https://ryneandal.com/2026/05/07/exporting-book-metadata-from-apple-books
permalink: /2026/05/07/exporting-book-metadata-from-apple-books
seo:
  description: "I've always used Apple's Books (formerly iBooks) for my reading. One issue I've always run into is no export functionality. I wanted a list of books I already owned, and there was no easy way to do it since I always purchase DRM-free ebooks. This post describes how and why I wrote a simple program to export the metadata for my books."
  keywords: "macOS, Apple Books, metadata, export, Rust"
image: /assets/img/2026-05-07-exporting-book-metadata-from-apple-books/old-book-spines.jpg
---

## Exporting Book Metadata from Apple Books

I've always used Apple's Books (formerly iBooks) for my reading. One issue I've always run into is no export functionality. I've been binging the Horus Heresy series and recently found out they were moving from DRM-free downloads to purchase management via their own app. I wanted to identify which books I did not already import into my library so I did not lose out on any past purchases. There was no easy way to generate a list of books in my library, so I used it as an excuse for learning to use Rust. This post describes how and why I wrote a simple program to export the metadata for my books.

### The Problem

Apple Books does not give you your data in a usable format.

You can:

* browse your library
* see reading progress per book
* sync across devices

But you cannot:

* export your library
* access progress in bulk
* analyze anything outside their UI

There’s no API, no export, no structured access. There isn't even clear documentation on where the data is stored, so I needed to do some digging and refer to my limited existing knowledge of MacOS application architecture.

### Finding the Bundle ID

Per Apple's dev documentation, app data is sandboxed `~/Library/Containers/{app-bundle-id}`, so I needed to find that bundle id. MacOS applications use `plist` files to store their metadata (notably the `Info.plist` file), and the developer documentation provides a list of valid keys, which made it a simple exercise in critical thinking to determine which one I wanted ([CFBundleIdentifier](https://developer.apple.com/documentation/coreservices/kmditemcfbundleidentifier)). Most first-party apps these days are stored in `/System/Applications/`, so running `rg -A 1 "CFBundleIdentifier" /System/Applications/Books/Contents/Info.plist` spat out:

```shell
rg "CFBundleIdentifier" -A 1 /System/Applications/Books.app/Contents/Info.plist
235: <key>CFBundleIdentifier</key>
236- <string>com.apple.iBooksX</string>
```

An alternative method is the `mdls` command, which is a command-line tool for querying metadata attributes of files and directories:

```shell
mdls -name kMDItemCFBundleIdentifier /System/Applications/Books.app
kMDItemCFBundleIdentifier = "com.apple.iBooksX"
```

And with the bundle id, I can now access the app's data:

```shell
ls -la ~/Library/Containers/com.apple.iBooksX/Data/Library/Application Support/Books/
```

### Finding the Data

After looking through the directory structure of the app's sanbox container, I found the data I was looking for in a sqlite database file at `~/Library/Containers/com.apple.iBooksX/Data/Documents/BKLibrary/BKLibrary-1-{created-timestamp}.sqlite`.

The next step was to verify this was what I wanted, so I inspected the schema:

```shell
sqlite3 ~/Library/Containers/com.apple.iBooksX/Data/Documents/BKLibrary/BKLibrary-1-091020131601.sqlite ".schema"

CREATE TABLE ZBKLIBRARYASSET ( Z_PK INTEGER PRIMARY KEY, Z_ENT INTEGER, Z_OPT INTEGER, ZAUTHOR VARCHAR, ZTITLE VARCHAR, ZISFINISHED INTEGER, ... );
```

A lot of meaningless table names and columns, but sifting through the noise you can see that the `ZBKLIBRARYASSET` table contains columns for author, title, series, etc. Pulling a row verified my hypothesis:

```shell
sqlite3 ~/Library/Containers/com.apple.iBooksX/Data/Documents/BKLibrary/BKLibrary-1-091020131601.sqlite "select * from ZBKLIBRARYASSET LIMIT 1"
2|...|Steven Erikson||||||com.apple.ibooks.datasource.jalisco.purchases||||Epic Fantasy||||||||||Book 5|719884766||Erikson, Steven|Midnight Tides|385981116|||Midnight Tides|||https://books.apple.com/us/book/midnight-tides/id385981116|||
```

That lines up with the first purchase I ever made on Apple Books, continuing my first read of the [Malazan Book of the Fallen](https://www.goodreads.com/series/43493-malazan-book-of-the-fallen) series after reading the first four physical books. It's a great series, and I highly recommend it. Some of the best high fantasy, with incredible worldbuilding, characters, and realistic moral ambiguity. Anyway, now that I knew the data was in the `ZBKLIBRARYASSET` table, I needed to figure out how to export it. Since it was a sqlite database, the task was trivial enough to automate, and I always try to use these simple tasks as an opportunity to learn a new language or tool so I started a Rust project.

### Exporting the Data

The task was simple enough, so I started with a simple Rust program to make sure I could access the data:

```rust

use std::path::Path;
use rusqlite::Connection;

fn main() {
       /// Column indices:
      const AUTHOR_IDX: usize = 73;
      const TITLE_IDX: usize = 101;
      const CREATION_DATE_IDX: usize = 55;
      const IS_FINISHED_IDX: usize = 21;
      let path = Path::new("/Users/ryne/Library/Containers/com.apple.iBooksX/Data/Documents/BKLibrary/BKLibrary-1-091020131601.sqlite");
      let db = Connection::open(path).unwrap();
      let mut stmt = db.prepare("SELECT * FROM ZBKLIBRARYASSET LIMIT 5 OFFSET 1").unwrap();
      let rows = stmt.query_map([], |row| {
          Ok((
              row.get::<_, Option<String>>(AUTHOR_IDX)?,
              row.get::<_, Option<String>>(TITLE_IDX)?,
              row.get::<_, Option<f32>>(CREATION_DATE_IDX)?,
              row.get::<_, Option<i8>>(IS_FINISHED_IDX)?,
          ))
      }).unwrap();
      for (idx, row) in rows.enumerate() {
          let (author, title, creation_date, is_finished) = match row {
              Ok(val) => val,
              Err(e) => {
                  eprintln!("Error reading row {}: {}", idx + 1, e);
                  continue;
              }
          };
     
          println!("{}. author={:?}, title={:?}, creation_date={:?}, is_finished={:?}", idx + 1, author, title, creation_date, is_finished);
      }
  }
```

with the output:

```shell
cargo run --example first_five
   Compiling apple-books-data-export v0.1.0 (/Users/ryne/code/GitHub/apple-books-exporter)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.12s
     Running `target/debug/examples/first_five`
1. author=Some("Steven Erikson"), title=Some("Midnight Tides"), creation_date=Some(742489600.0), is_finished=Some(1)
2. author=Some("Arthur Schopenhauer"), title=Some("The Essays of Arthur Schopenhauer: the Wisdom of Life"), creation_date=Some(742489600.0), is_finished=Some(1)
3. author=Some("Steven Erikson"), title=Some("Toll the Hounds"), creation_date=Some(742489600.0), is_finished=None
4. author=Some("Virgil"), title=Some("The Aeneid"), creation_date=Some(742489600.0), is_finished=Some(1)
5. author=Some("Jason Schreier"), title=Some("Blood, Sweat, and Pixels"), creation_date=Some(742489600.0), is_finished=Some(1)
```

So I had a proof of concept, all I needed to do was export the data to a CSV file. With some assistance from Cursor, I quickly threw together a simple Rust binary to export the data to a CSV file, using the [clap crate](https://docs.rs/clap/latest/clap/) which made parsing command line arguments trivial via derive, feeling a lot like Typer annotations in Python.

The code is available under the MIT license on [GitHub](https://github.com/ryneandal/apple-books-exporter), and can be easily installed via homebrew:

```shell

brew tap ryneandal/books
brew install apple-books-data-export

apple-books-data-export export --format json --output books.json --pretty
```

Each row is one library asset. JSON and CSV use the same logical columns (JSON uses RFC 3339 datetimes where present):

| Field | Notes |
|-------|-------|
| title | Required string |
| author | Optional |
| status | finished, in_progress, or not_started_or_unknown (derived from Apple Books flags and progress) |
| reading_progress | Optional fraction |
| high_watermark_progress | Optional fraction |
| finished_at | Optional UTC datetime |
| last_opened_at | Optional UTC datetime |
| last_engaged_at | Optional UTC datetime |
| library_record_created_at | Optional UTC datetime |
| asset_guid | Optional string |
| genre | Optional string |

Some of the fields are derived from the Apple Books flags and progress, and some are not present in the database. The `status` field is derived from the `ZISFINISHED` column, and the `reading_progress` and `high_watermark_progress` fields are derived from the `ZREADINGPROGRESS` and `ZHIGHWATERMARKPROGRESS` columns. The `finished_at`, `last_opened_at`, and `last_engaged_at` fields are derived from the `ZLASTOPENEDDATE` and `ZLASTENGAGEDDATE` columns. The `library_record_created_at` field is derived from the `ZCREATEDDATE` column. The `asset_guid` field is derived from the `ZASSETGUID` column. The `genre` field is derived from the `ZGENRE` column. There are a ton of other columns in the database, but only selected a few that I wanted. If you're looking for a full list of columns, the entire schema [documented](https://github.com/ryneandal/apple-books-exporter/blob/main/docs/BKLibrary-schema.md) in the codebase, with best guess as inferred purpose for the asset table.
