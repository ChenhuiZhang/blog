---
layout: title
title: Cargo build timing
date: 2021-08-17 13:59:33
tags: rust
---

1. Switch to nightly version
```
rustup default nightly-x86_64-unknown-linux-gnu
```

2. Run Cargo build with "-Z timings"
```
cargo build -Z timings
```

3. Check the output report
Example: [cargo-timing.html](/html/cargo-timing.html)
