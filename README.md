# multipart-nv

`multipart/form-data` is the body of an HTTP request that carries a form: text fields and files, each in a part of its own, separated by a boundary. It is defined by RFC 7578, on top of the multipart syntax of RFC 2046 section 5.1. This package writes such a body, and reads a complete one for tests.

A server that receives `multipart/form-data` uses [form-nv](https://novo-lang.org/packages/form-nv), which reads a body as it arrives, with limits. The standard library's HTTP client sends a body it is given; it does not build one.

## What it is

A body is a sequence of parts. Each part is a delimiter line, its headers, an empty line and its content.

```
--AaB03x
Content-Disposition: form-data; name="submit-name"

Larry
--AaB03x
Content-Disposition: form-data; name="files"; filename="file1.txt"
Content-Type: text/plain

... contents of file1.txt ...
--AaB03x--
```

Every line ends with a carriage return and a line feed. The word after `--` is the **boundary**. The delimiter that separates parts is a line break, two hyphens and the boundary. The body ends with the boundary followed by two more hyphens. A part whose content held the delimiter would end early, so the boundary must be one that no part holds.

## Install

```
novo pkg add multipart-nv
```

## Example

```novo
use std.bytes
use mpboundary
use mpwrite

fn main() [io]
    let note = mpwrite.field("note", "hello")
    let upload = mpwrite.file("upload", "a.txt", "text/plain", bytes.from_str("file text"))
    // The seed comes from a random source of the caller's.
    let sealed = mpwrite.sealed(20261123, [note, upload])
    println(mpboundary.content_type(sealed.boundary))
    println("${bytes.len(sealed.body)} bytes")
```

The program prints a `Content-Type` header value of the form `multipart/form-data; boundary=novo-` followed by 32 hexadecimal digits, and the length of the body, 283 bytes. A request sends that header and the body:

```
let client = HttpClient.new("http://printer.local").with_header("Content-Type", mpboundary.content_type(sealed.boundary))
client.send("POST", "/api/files/local", sealed.body)
```

## What the package contains

| Module | Contents |
| --- | --- |
| `mpwrite` | `MpPart`, `field` and `file` to make parts, `body` to write them, `sealed` to choose a boundary and write them, and the pieces of a body for streaming: `head_of`, `tail`, `closing` and `overhead_of`. |
| `mpboundary` | `MpBoundary`, a boundary that passed the RFC 2046 check; `choose` to pick one that no content holds; `content_type` and `in_content_type` for the header value. |
| `mpread` | `parse`, a reader of a complete body. |
| `mperr` | `MpFault`, with a message and a code. |

## How to choose an entry point

Use `mpwrite.sealed` when every part is in memory. It picks the boundary and writes the body.

Use `mpwrite.body` when the boundary was given to you, as it is when a protocol fixes it. It refuses a part that holds the delimiter.

Use the pieces when a file is too large to hold. The caller writes, to a connection or a file, in this order:

1. `head_of` of each part, then that part's content in as many chunks as it reads them, then `tail`.
2. `closing`, once, after the last part.

The `Content-Length` is `overhead_of(boundary, parts)` plus the size of every content. For a streamed file the part given to `overhead_of` has empty data, and the size of the file is added.

## The rules a user needs

1. A boundary is 1 to 70 characters from letters, digits, space and `'()+_,-./:=?`, and does not end in a space (RFC 2046 section 5.1.1). `mpboundary.parse` refuses anything else, and every function that writes a delimiter takes the checked `MpBoundary`.
2. `choose` and `sealed` take a seed and give a boundary of `novo-` and 32 hexadecimal digits. The seed comes from the caller, because this package reads no random source. If the delimiter for that seed occurs in a part's content, name, file name or media type, the next seed is tried.
3. A part streamed from a file cannot be searched. The 128 bits of the boundary make a clash improbable, and a caller that needs a certainty reads the file once with `first_clash` or `choose_by`.
4. A text field has no `Content-Type` header, which RFC 7578 section 4.4 reads as `text/plain`. A file part has one, and `application/octet-stream` when the caller gives none.
5. In a name or a file name, a quote is written `%22`, a carriage return `%0D` and a line feed `%0A`, as the HTML specification's multipart encoding does. Other bytes, UTF-8 included, are written as they are.
6. `mpread.parse` reverses those three sequences. A name that holds one of them as text reads back changed, because the format cannot tell the two apart.
7. `mpread.parse` ends a part at a line break and the boundary only when the boundary is followed by a line break or by two hyphens. Text after the closing delimiter is ignored.
8. A part is a text field when its `Content-Disposition` has no `filename` parameter, and a file part otherwise, including a file part with an empty file name.

## Design

The pieces of a body are separate byte strings, so a file is never copied into the body. `body` is those pieces in one buffer of the exact length.

Two choices have another reasonable answer.

- **Names are percent-escaped.** RFC 7578 section 4.2 names two other ways: a `filename*` parameter with an RFC 5987 encoding, and an RFC 2231 form. Browsers write the percent form, and servers read it, so this package does too.
- **The boundary is a parameter, not a byte string that the writer repairs.** A writer that silently changed a boundary it was given would send a `Content-Type` header that disagrees with the body, so `body` refuses instead.

## What is not included

- **A socket and a file.** The caller reads the file and sends the bytes. Every function declares no effect.
- **A streaming reader with limits.** That is [form-nv](https://novo-lang.org/packages/form-nv).
- **Nested `multipart/mixed` parts.** RFC 7578 section 4.3 deprecates them.
- **Content-Transfer-Encoding.** RFC 7578 section 4.7 deprecates it; every part is sent as raw bytes.
- **MIME email.** A message of `multipart/mixed` or `multipart/alternative` has other headers and encodings. See [mail-nv](https://novo-lang.org/packages/mail-nv).

## Tests

```bash
novo test tests/multipart_tests.nv         # the RFC example, urllib3's bytes, round trips, streaming
novo test tests/multipart_edge_tests.nv    # the boundary grammar and every refusal
bash tests/coverage.sh                     # line coverage over src/, merged across the suites
```

The body of RFC 7578 section 4.1 is written byte for byte. Two more bodies were produced by an independent encoder, `urllib3.filepost.encode_multipart_formdata` of urllib3 2.0.7: one with a text field and a binary file, and one with a quote and a line break in a field name, a quote and a line feed in a file name, and non-ASCII text. The first and a body from the same encoder with the same names are compared byte for byte, and the second is read back, its file content holding `--XX ` and `\r\n--XXY`, which are not delimiters for the boundary `XX`.

## Licence

Apache-2.0. See `LICENSE`.
