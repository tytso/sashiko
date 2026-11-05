+++
title = "Support and Legacy Shared Libraries"
paths = ["lib/e2p/", "lib/et/", "lib/ss/", "lib/support/", "lib/blkid/", "lib/uuid/"]
symbols = ["e2p_*", "com_err*", "quota_*"]
+++
# Support and Legacy Shared Libraries: maintainer notes

ABI policy and internal-vs-exported boundaries for `libe2p`, `libcom_err`,
`libss`, `libblkid`, `libuuid`, and `libsupport`.

- `libe2p`, `libcom_err` (`lib/et/`), `libss` (`lib/ss/`), `libblkid`, and
  `libuuid` are installed shared libraries with external consumers: preserve
  exported function signatures, struct layouts, and symbol versions (`High` if
  broken), and never print errors directly to `stdout`/`stderr` inside
  `libe2p`, `libblkid`, or `libuuid`.
- `libsupport` (`lib/support/`) is an internal static convenience library used
  only by `e2fsprogs` binaries (quota helpers, profile parser, rbtree/dict
  containers, plausible-device checks); `printf`/`com_err` diagnostics are
  permitted in `libsupport`, and its internal APIs may evolve alongside all
  in-tree callers without shared-library ABI wrappers.
- When adding a new filesystem feature bit name to `lib/e2p/feature.c`, verify
  the compat/incompat/ro_compat mask and bit constant match `lib/ext2fs/ext2_fs.h`
  and Linux `fs/ext4/ext4.h` (`High` if mapped to the wrong feature mask or bit).
