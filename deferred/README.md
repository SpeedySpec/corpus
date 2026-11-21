# Deferred findings

Problems found in review that were judged too small to fix at the time, kept so they are not lost.

## Layout

```
deferred/
└── <repository>/          One directory per repository, named as the repository is
    ├── README.md          Where the findings came from, and the index
    └── <area>.md          One file per area of the code
```

## Each finding

- **Where:** the crate, file or function.
- **What happens:** what someone using the software gets, in plain terms.
- **Suggested fix:** when the review proposed one.

When a finding is fixed, delete it and name the pull request in the commit message. When a whole file empties, delete
the file.
