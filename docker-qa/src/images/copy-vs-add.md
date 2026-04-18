# Q: What is the difference between `COPY` and `ADD` in a Dockerfile?

**Answer:**

Both instructions copy files from the build context into the image, but `ADD` has extra (often unwanted) functionality.

### `COPY` — Simple File Copy
Does exactly one thing: copies files or directories from the build context into the image filesystem. It's transparent and predictable.

```dockerfile
COPY requirements.txt /app/
COPY src/ /app/src/
```

### `ADD` — Copy with Extras
`ADD` does everything `COPY` does, plus two additional features:
1. **Auto-extracts compressed archives** (`.tar`, `.tar.gz`, `.tgz`, `.bz2`, `.xz`) into the destination directory.
2. **Fetches files from remote URLs** (like `wget`).

```dockerfile
# Auto-extracts the tarball into /app/
ADD app.tar.gz /app/

# Downloads a file from the internet
ADD https://example.com/config.json /etc/app/config.json
```

### Why You Should Almost Always Use `COPY`

> [!WARNING]
> The Docker official best practices guide explicitly recommends using `COPY` over `ADD` in almost all cases.

**Reasons:**
1. **Predictability**: `COPY` has no hidden side effects. With `ADD`, a developer might not realize their `.tar.gz` file will be auto-extracted. If you *want* the archive as-is (e.g., to extract it manually later), `ADD` will silently break your intent.
2. **Security**: `ADD` from a URL does not verify SSL certificates and doesn't support authentication. Use `RUN curl` or `RUN wget` instead for better control.
3. **Cache invalidation**: Remote URL fetches with `ADD` can cause unpredictable cache behavior since Docker cannot know if the remote file has changed.

### Rule of Thumb
- Use **`COPY`** for all local file copies (99% of cases).
- Use **`ADD`** only when you explicitly need tar auto-extraction.
- Use **`RUN curl`** or **`RUN wget`** for downloading remote files.
