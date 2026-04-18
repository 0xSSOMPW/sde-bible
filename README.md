# 📚 Interview Prep Books

A collection of curated interview preparation notes built with [mdbook](https://rust-lang.github.io/mdBook/). Each language/technology has its own standalone mdbook, linked from a central landing page.

## Structure

```
learn/
├── index.html          # Landing page (open this in a browser)
├── styles.css          # Landing page styles
├── js-ts-qa/           # JavaScript & TypeScript interview notes
├── kafka-qa/           # Apache Kafka interview notes
├── docker-qa/          # Docker & DevOps interview notes
├── rust-qa/            # Rust interview notes
└── java-qa/            # Java interview notes (WIP)
```

## Prerequisites

```bash
cargo install mdbook
```

## Building

```bash
# Build all books
mdbook build js-ts-qa
mdbook build rust-qa
mdbook build java-qa
```

## Serving Locally

```bash
# Serve a specific book with live reload
mdbook serve js-ts-qa --open

# Or serve the landing page with a static server
python3 -m http.server 8000
# then open http://localhost:8000
```

## Adding a New Question

1. Create a markdown file inside the appropriate `src/<topic>/` subdirectory.
2. Add a link to it in `src/SUMMARY.md`.
3. Run `mdbook build <book-dir>` to compile.
# sde-bible
