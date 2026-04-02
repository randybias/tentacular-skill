# Sidecar HTTP Hook Pattern

Wrap a public Docker image with an inline HTTP server so Tentacular nodes
can call it via `globalThis.fetch("http://localhost:PORT/...")`. No custom
images required — the hook script is injected at deploy time via the
`command:` and `args:` fields in `workflow.yaml`.

## When to Use This Pattern

Use the HTTP hook pattern when:

- The public image contains the native binary you need (ffmpeg, ImageMagick,
  etc.) but does NOT expose an HTTP API
- The image includes a scripting runtime (Python3, Perl, or bash) that can
  host a lightweight HTTP server
- You want to avoid building and maintaining a custom sidecar image

Do NOT use this pattern when:

- The image already has a built-in HTTP API (e.g., `pandoc/core` ships
  `pandoc-server` on port 3030). Just use the image directly.
- The image has no scripting runtime at all (rare — most Debian/Ubuntu-based
  images include at least Perl and bash). Consider a different image or an
  init container approach.

## How It Works

1. Declare the sidecar in `workflow.yaml` with a public image
2. Override `command:` and `args:` to start an HTTP server script instead of
   the image's default entrypoint
3. The script handles `/health` (GET) for readiness probes and custom
   endpoints (POST) that shell out to the native binary
4. Nodes call the sidecar via `globalThis.fetch("http://localhost:PORT/...")`
5. Large files go through `/shared/` (emptyDir volume), not HTTP bodies

```yaml
sidecars:
  - name: ffmpeg
    image: linuxserver/ffmpeg:latest          # public image
    port: 9000
    healthPath: /health
    command: ["bash", "-c"]
    args:
      - "echo BASE64_ENCODED_SCRIPT | base64 -d > /tmp/hook.pl && exec perl /tmp/hook.pl"
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
```

### Base64 Encoding Requirement

`tntc validate` rejects newlines in sidecar `args` values. Multiline scripts
must be base64-encoded and decoded at runtime. The pattern:

1. Write the hook script as a normal file (for readability and testing)
2. Base64-encode it: `cat hook.pl | base64 | tr -d '\n'`
3. Use `bash -c` to decode and exec:

```yaml
command: ["bash", "-c"]
args:
  - "echo ENCODED | base64 -d > /tmp/hook.pl && exec perl /tmp/hook.pl"
```

The `exec` replaces bash with Perl so the process tree stays clean and
signal handling works correctly. The script lands in `/tmp` (an emptyDir
volume auto-provisioned by the builder).

**Keep the source script alongside your workflow** (e.g., `hooks/ffmpeg.pl`)
for version control and readability. Only the base64-encoded form goes into
`workflow.yaml`.

## Image Compatibility

Before choosing an image, verify it includes a scripting runtime. Use the
highest available tier:

| Tier | Runtime | JSON Support | Suitability | Check Command |
|------|---------|-------------|-------------|---------------|
| 1 | Python3 | Built-in (`json` module) | Best | `docker run --rm --entrypoint python3 IMAGE --version` |
| 2 | Perl 5 | Manual (simple for sidecar payloads) | Good | `docker run --rm --entrypoint perl IMAGE -v` |
| 3 | bash + nc | Manual (fragile, single-connection) | Last resort | `docker run --rm --entrypoint bash IMAGE -c "which nc"` |

### Known Image Runtimes

| Image | Use Case | Python3 | Perl | bash | nc | Multi-arch |
|-------|----------|---------|------|------|----|------------|
| `linuxserver/ffmpeg:latest` | Video/audio processing | -- | 5.38 | yes | yes | arm64+amd64 |
| `dpokidov/imagemagick:latest` | Image conversion | -- | yes | yes | -- | TBD |
| `ghcr.io/browserless/chromium` | Headless browser | yes | yes | yes | -- | arm64+amd64 |
| `pandoc/core:3.6` | Document conversion | -- | -- | -- | yes | arm64+amd64 |
| `alpine:latest` | Minimal (testing/probes) | -- | -- | yes | yes | arm64+amd64 |

**Note:** `pandoc/core` does NOT need the hook pattern — it ships `pandoc-server`
with a built-in HTTP API. It is listed here only for completeness.

Always verify before committing to an image. Image contents change across
versions. Pin to a specific tag or digest in production, not `:latest`.

### Binary Path Discovery

Some images install tools in non-standard locations. The binary may not be
on `PATH` even though it exists in the image. Common pitfalls:

- **browserless/chromium**: Chromium lives at
  `/usr/local/bin/playwright-browsers/chromium-*/chrome-linux/chrome`, NOT as
  `chromium` on PATH. Use `glob` in Python or `find` in bash to locate it.
- **linuxserver/ffmpeg**: `ffmpeg` is on PATH (works directly).
- **pandoc/core**: `pandoc` and `pandoc-server` are on PATH.

**Always probe the image before writing the hook script:**

```bash
# Find a binary by name
docker run --rm --entrypoint sh IMAGE -c "find / -type f -executable -name '*chromium*' 2>/dev/null | head -10"

# Check what runtimes are available
docker run --rm --entrypoint sh IMAGE -c "which python3 perl bash nc 2>/dev/null"
```

When the binary path is version-dependent (contains a version number in the
directory), use glob patterns in your hook script rather than hardcoding:

```python
# Python: glob for versioned paths
import glob
matches = glob.glob("/usr/local/bin/playwright-browsers/chromium-*/chrome-linux/chrome")
chrome = matches[0] if matches else None
```

## Template: Python3 HTTP Wrapper (Tier 1)

Use when the image includes Python3. Cleanest option with built-in JSON.

Save as `hooks/hook.py` in your tentacle directory:

```python
import http.server, json, subprocess, os, time, glob

class Handler(http.server.BaseHTTPRequestHandler):
    def log_message(self, fmt, *args):
        print(fmt % args, flush=True)

    def do_GET(self):
        if self.path == "/health":
            self._json_response(200, {"status": "ok"})
        else:
            self._text_response(404, "not found")

    def do_POST(self):
        if self.path == "/run":
            length = int(self.headers.get("Content-Length", 0))
            body = json.loads(self.rfile.read(length)) if length else {}

            # --- Replace this block with your binary invocation ---
            cmd = ["my-tool", body.get("input", "")]
            start = time.time()
            result = subprocess.run(cmd, capture_output=True, text=True)
            duration_ms = int((time.time() - start) * 1000)

            if result.returncode != 0:
                self._json_response(500, {"error": result.stderr[:500]})
                return

            self._json_response(200, {
                "stdout": result.stdout,
                "duration_ms": duration_ms
            })
            # --- End replaceable block ---
        else:
            self._text_response(404, "not found")

    def _json_response(self, code, obj):
        body = json.dumps(obj).encode()
        self.send_response(code)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def _text_response(self, code, msg):
        body = msg.encode()
        self.send_response(code)
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

httpd = http.server.HTTPServer(("0.0.0.0", 9000), Handler)
print("HTTP wrapper listening on :9000", flush=True)
httpd.serve_forever()
```

Encode and reference in `workflow.yaml`:

```bash
cat hooks/hook.py | base64 | tr -d '\n'
```

```yaml
sidecars:
  - name: my-tool
    image: some/image-with-python:tag
    port: 9000
    healthPath: /health
    command: ["bash", "-c"]
    args:
      - "echo ENCODED | base64 -d > /tmp/hook.py && exec python3 -u /tmp/hook.py"
```

## Template: Perl HTTP Wrapper (Tier 2)

Use when the image has Perl but not Python3. Most Ubuntu/Debian-based images
include `perl-base`. JSON is handled with simple string formatting — adequate
for the fixed-shape request/response payloads sidecars use.

Save as `hooks/hook.pl` in your tentacle directory:

```perl
use strict;
use warnings;
use IO::Socket::INET;

$| = 1;  # autoflush

my $server = IO::Socket::INET->new(
    LocalAddr => '0.0.0.0', LocalPort => 9000,
    Proto => 'tcp', Listen => 5, ReuseAddr => 1,
) or die "Cannot bind :9000: $!\n";

print "HTTP wrapper listening on :9000\n";

while (my $client = $server->accept()) {
    my $req = '';
    while (my $line = <$client>) {
        $req .= $line;
        last if $line =~ /^\r?\n$/;
    }
    my ($method, $path) = $req =~ /^(\w+)\s+(\S+)/;
    $method //= ''; $path //= '';

    my $cl = 0;
    $cl = $1 if $req =~ /Content-Length:\s*(\d+)/i;
    my $body = '';
    if ($cl > 0) { read($client, $body, $cl); }

    if ($method eq 'GET' && $path eq '/health') {
        http_json($client, 200, '{"status":"ok"}');
    }
    elsif ($method eq 'POST' && $path eq '/run') {
        # --- Replace this block with your binary invocation ---
        my ($input) = $body =~ /"input"\s*:\s*"([^"]*)"/;
        $input //= '';

        my $start = time();
        my $output = `my-tool "$input" 2>&1`;
        my $rc = $? >> 8;
        my $duration_ms = (time() - $start) * 1000;

        if ($rc != 0) {
            my $err = substr($output, 0, 500);
            $err =~ s/"/\\"/g;
            http_json($client, 500, qq({"error":"$err"}));
        } else {
            $output =~ s/"/\\"/g;
            $output =~ s/\n/\\n/g;
            http_json($client, 200,
                qq({"stdout":"$output","duration_ms":$duration_ms}));
        }
        # --- End replaceable block ---
    }
    else {
        http_json($client, 404, '{"error":"not found"}');
    }
    close $client;
}

sub http_json {
    my ($fh, $code, $json) = @_;
    my $status = $code == 200 ? 'OK' : $code == 404 ? 'Not Found' : 'Error';
    print $fh "HTTP/1.1 $code $status\r\n";
    print $fh "Content-Type: application/json\r\n";
    print $fh "Content-Length: " . length($json) . "\r\n";
    print $fh "Connection: close\r\n";
    print $fh "\r\n";
    print $fh $json;
}
```

Encode and reference in `workflow.yaml`:

```bash
cat hooks/hook.pl | base64 | tr -d '\n'
```

```yaml
sidecars:
  - name: my-tool
    image: some/image-with-perl:tag
    port: 9000
    healthPath: /health
    command: ["bash", "-c"]
    args:
      - "echo ENCODED | base64 -d > /tmp/hook.pl && exec perl /tmp/hook.pl"
```

### Perl JSON Helpers

The Perl template uses manual JSON formatting because `JSON::PP` is not
always installed. For sidecar payloads this is fine — the shapes are simple
and fixed. Rules:

- **Producing JSON:** Use `qq()` interpolation for known fields. Escape
  double quotes and newlines in dynamic values.
- **Parsing JSON:** Use regex extraction (`/"key"\s*:\s*"([^"]*)"/"`) for
  string fields, (`/"key"\s*:\s*(\d+)/`) for numbers. This is adequate for
  the flat JSON objects nodes send to sidecars.
- **Complex JSON:** If you need nested objects or arrays, consider whether
  the image has `JSON::PP` (`perl -MJSON::PP -e1`) or switch to a
  shared-volume command pattern instead.

## Shared Volume Pattern

For operations on large files, use `/shared/` instead of HTTP bodies:

1. **Node** writes input to `/shared/input/` (e.g., video file)
2. **Node** POSTs a JSON command to the sidecar with file paths (not file contents)
3. **Sidecar** reads from `/shared/input/`, processes, writes to `/shared/output/`
4. **Sidecar** responds with JSON metadata (frame paths, file sizes, durations)
5. **Node** reads results from `/shared/output/`

This avoids memory pressure from large HTTP bodies and is the standard
pattern for video, audio, and image processing sidecars.

```
Node                          Sidecar
 |                              |
 |-- write video to /shared --> |
 |-- POST {input: path} -----> |
 |                              |-- ffmpeg reads /shared/input
 |                              |-- ffmpeg writes /shared/output
 |<--- {frames: [...]} --------|
 |-- read frames from /shared   |
```

## E2E Validated Examples

These combinations have been tested end-to-end on a real cluster:

| Tier | Image | Hook Language | Binary Tested | Result |
|------|-------|---------------|---------------|--------|
| 1 | `ghcr.io/browserless/chromium:latest` | Python3 | chromium --version | Chromium 147.0.7727.0 |
| 2 | `linuxserver/ffmpeg:latest` | Perl | ffmpeg -version | ffmpeg 8.0.1 |
| 3 | `alpine:latest` | bash+nc | (none — fixed response) | OK |
| -- | `pandoc/core:3.6` | (no hook) | pandoc-server | Built-in HTTP API |

## Verification Checklist

Before deploying a sidecar with the hook pattern:

1. **Runtime exists:** Verified the image contains the scripting runtime
   (ran `docker run --rm --entrypoint <runtime> IMAGE --version`)
2. **Binary exists:** Verified the native binary is on PATH in the image
   (ran `docker run --rm --entrypoint <binary> IMAGE --help`)
3. **Port is correct:** Port in `workflow.yaml` matches the port in the
   wrapper script, and is not 8080 (reserved for engine)
4. **Health endpoint works:** The wrapper script responds to
   `GET /health` with 200
5. **Security context compatible:** The wrapper runs as uid 65534 with
   readOnlyRootFilesystem — no writing outside `/tmp` and `/shared`
6. **Multi-arch:** The image supports the target cluster's architecture
   (check with `docker manifest inspect IMAGE`)
