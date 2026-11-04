# NitroArchive

A streaming, seekable compressed archive format for large byte streams.

NitroArchive creates independently compressed Zstandard blocks that can be read locally or fetched on demand from HTTP and S3-compatible object storage. Archives can also be mounted with FUSE as a read-only file named `data`.

## Why NitroArchive?

- **Streaming:** compress and decompress without temporary files.
- **Seekable:** independently compressed blocks support efficient random reads.
- **Remote-friendly:** HTTP and S3 readers fetch only the required ranges.
- **Fast:** blocks are compressed concurrently while archive output remains ordered.
- **Recoverable:** complete blocks can be restored from a truncated archive tail.
- **Encrypted:** optional authenticated AES-256-GCM encryption for blocks and indexes.
- **Mountable:** expose the uncompressed stream through a read-only FUSE mount.

> NitroArchive stores one byte stream per archive. It is ideal for disk images, database snapshots, tar streams, and individual large files. It is not a multi-file archive format by itself; package directories as a tar stream when needed.

## Quick start

Compress and restore a file:

```bash
./nitroarchive -c -i source.img -o source.nar
./nitroarchive -d -i source.nar -o restored.img
cmp source.img restored.img
```

Or use standard input and standard output:

```bash
./nitroarchive -c < source.img > source.nar
./nitroarchive -d < source.nar > restored.img
```

## Requirements

- FUSE 3 or a compatible FUSE installation when mounting archives.
- An S3-compatible account only when reading from or writing to S3.

## Common recipes

### Archive a disk image

```bash
nitroarchive -c -i server.img -o server.img.nar
nitroarchive -d -i server.img.nar -o server-restored.img
sha256sum server.img server-restored.img
```

### Archive a directory as a tar stream

NitroArchive stores a single stream, so use `tar` to preserve a directory tree:

```bash
tar -C /srv/data -cf - . | nitroarchive -c -o backup.tar.nar
```

Restore it:

```bash
mkdir -p restored
nitroarchive -d -i backup.tar.nar | tar -C restored -xf -
```

List the embedded tar archive without extracting it:

```bash
nitroarchive -d -i backup.tar.nar | tar -tf -
```

### Show progress

```bash
nitroarchive -verbose -c -i source.img -o source.nar
nitroarchive -verbose -d -i source.nar -o restored.img
```

Progress is written to standard error, so it is safe to use while archive data is written to standard output:

```bash
nitroarchive -verbose -c < source.img > source.nar
```

### Tune block and checkpoint sizes

```bash
nitroarchive -c \
  -block-size 8388608 \
  -checkpoint 524288000 \
  -i source.img \
  -o source.nar
```

This example uses 8 MiB blocks and a 500 MiB checkpoint interval.

- Larger blocks can improve compression but increase the amount fetched for small random reads.
- Smaller blocks improve seek granularity but add metadata and request overhead.
- More frequent checkpoints improve intermediate index availability but increase archive size.

The defaults are 4 MiB blocks and 250 MiB checkpoints.

## Encryption

Encryption is enabled whenever a key is supplied. Keys must contain exactly 32 random bytes encoded as 64 hexadecimal characters or standard base64.

Generate a key and create an encrypted archive:

```bash
export NITROARCHIVE_KEY="$(openssl rand -hex 32)"
printf '%s\n' "$NITROARCHIVE_KEY" > nitroarchive.key
chmod 600 nitroarchive.key

nitroarchive -c -i source.img -o source.img.nar
nitroarchive -d -i source.img.nar -o restored.img
```

Load the key in a later shell:

```bash
export NITROARCHIVE_KEY="$(cat nitroarchive.key)"
nitroarchive -d -i source.img.nar -o restored.img
```

The `-key` option is also supported:

```bash
nitroarchive -c -key 'hex:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef' -i source.img -o source.nar
```

Using `NITROARCHIVE_KEY` is preferred because command-line arguments may be visible in process listings and shell history. Store the key separately from the archive and back it up securely. An encrypted archive cannot be recovered without its key.

## Remote input and S3 output

### Read an archive over HTTPS

HTTP servers must return `Content-Length` and support byte-range requests for random access and FUSE mounting.

Stream-decompress an HTTPS archive:

```bash
nitroarchive -d \
  -i https://downloads.example.com/backups/server.img.nar \
  -o server.img
```

Compress a remote object into a local archive:

```bash
nitroarchive -c \
  -i https://downloads.example.com/images/server.img \
  -o server.img.nar
```

### Use Amazon S3

NitroArchive uses the standard AWS SDK credential chain:

```bash
export AWS_PROFILE=archive-operator
export AWS_REGION=eu-west-1

nitroarchive -c \
  -i source.img \
  -o s3://company-archives/images/source.img.nar

nitroarchive -d \
  -i s3://company-archives/images/source.img.nar \
  -o restored.img
```

Copy from one S3 object to another while transforming the stream:

```bash
nitroarchive -c \
  -i s3://source-bucket/images/source.img \
  -o s3://archive-bucket/images/source.img.nar
```

S3 outputs use multipart uploads. Inputs generally require `s3:GetObject`; outputs require `s3:PutObject` and the multipart-upload permissions applicable to the bucket policy.

### Use an S3-compatible provider

Set a custom endpoint and, if required, path-style addressing:

```bash
export AWS_ACCESS_KEY_ID='your-access-key'
export AWS_SECRET_ACCESS_KEY='your-secret-key'
export AWS_REGION='us-east-1'
export NITROARCHIVE_S3_ENDPOINT='https://s3.example.com'
export NITROARCHIVE_S3_PATH_STYLE=true

nitroarchive -c \
  -i source.img \
  -o s3://archive-bucket/source.img.nar
```

Wasabi example:

```bash
export AWS_ACCESS_KEY_ID='your-access-key'
export AWS_SECRET_ACCESS_KEY='your-secret-key'
export AWS_REGION='us-east-1'
export NITROARCHIVE_S3_ENDPOINT='https://s3.us-east-1.wasabisys.com'
export NITROARCHIVE_S3_PATH_STYLE=false

nitroarchive -d \
  -i s3://archive-bucket/source.img.nar \
  -o source.img
```

OVHcloud example for the `gra` region:

```bash
export AWS_ACCESS_KEY_ID='your-access-key'
export AWS_SECRET_ACCESS_KEY='your-secret-key'
export AWS_REGION='gra'
export NITROARCHIVE_S3_ENDPOINT='https://s3.gra.io.cloud.ovh.net'
export NITROARCHIVE_S3_PATH_STYLE=false

nitroarchive -d \
  -i s3://archive-bucket/source.img.nar \
  -o source.img
```

Use the endpoint and region assigned to your bucket.

## FUSE mounts

A FUSE mount exposes the uncompressed archive stream as one read-only file named `data`.

### Mount a local archive

```bash
sudo mkdir -p /mnt/nitro
nitroarchive --fuse ./source.img.nar /mnt/nitro
```

In another terminal:

```bash
ls -lh /mnt/nitro/data
sha256sum /mnt/nitro/data
dd if=/mnt/nitro/data of=sample.bin bs=1M skip=100 count=4 status=progress
```

Unmount it:

```bash
fusermount3 -u /mnt/nitro
```

On systems without `fusermount3`:

```bash
umount /mnt/nitro
```

### Mount an HTTPS archive

```bash
mkdir -p "$HOME/mnt/nitro"
nitroarchive --fuse \
  https://downloads.example.com/backups/source.img.nar \
  "$HOME/mnt/nitro"
```

### Mount an S3 archive

```bash
mkdir -p "$HOME/mnt/nitro"
nitroarchive --fuse \
  s3://company-archives/images/source.img.nar \
  "$HOME/mnt/nitro"
```

For verbose FUSE statistics:

```bash
nitroarchive -verbose --fuse source.img.nar /mnt/nitro
```

The mount uses `allow_other`. Non-root users may need this line in `/etc/fuse.conf`:

```text
user_allow_other
```

FUSE mode requires the complete final index and footer. A truncated archive may still be stream-recoverable, but it cannot be mounted for random access.

## Configuration file

Start the interactive configuration wizard:

```bash
nitroarchive -wizardconfig
```

NitroArchive first checks `~/.nitroarchiverc`. If that file does not exist, it searches for `.nitroarchiverc` from the current directory upward through parent directories.

Example configuration:

```json
{
  "block_size": 4194304,
  "checkpoint": 262144000,
  "aws_access_key_id": "your-access-key",
  "aws_secret_access_key": "your-secret-key",
  "aws_region": "us-east-1",
  "s3_endpoint": "https://s3.example.com",
  "s3_path_style": true
}
```

Protect configuration files that contain credentials:

```bash
chmod 600 ~/.nitroarchiverc
```

Precedence is generally:

1. Command-line options.
2. Relevant environment variables.
3. Configuration file values.
4. Built-in defaults.

Useful environment variables include:

| Variable | Purpose |
| --- | --- |
| `NITROARCHIVE_KEY` | Archive encryption key |
| `AWS_ACCESS_KEY_ID` | AWS or S3-compatible access key |
| `AWS_SECRET_ACCESS_KEY` | AWS or S3-compatible secret key |
| `AWS_SESSION_TOKEN` | Temporary credential session token |
| `AWS_REGION` | AWS or provider region |
| `AWS_PROFILE` | Shared AWS profile |
| `AWS_SHARED_CREDENTIALS_FILE` | Alternate credentials file |
| `AWS_CONFIG_FILE` | Alternate AWS configuration file |
| `NITROARCHIVE_S3_ENDPOINT` | Custom S3-compatible endpoint |
| `NITROARCHIVE_S3_PATH_STYLE` | Force path-style bucket addressing |

## Recovery from an incomplete archive

Streaming decompression emits every complete block and tolerates a truncated trailing record. This can recover data written before an interrupted transfer:

```bash
nitroarchive -d -i incomplete.nar -o recovered.img
```

Check how much was recovered:

```bash
stat -c '%n: %s bytes' recovered.img
```

Authentication failures, damaged record headers, and invalid compressed blocks remain errors. Recovery does not bypass encryption or integrity checks.

## Command reference

```text
nitroarchive -c [options]              Compress
nitroarchive -d [options]              Decompress
nitroarchive --fuse SOURCE MOUNT_POINT Mount an archive
nitroarchive -wizardconfig             Configure defaults interactively
```

Common options:

| Option | Description |
| --- | --- |
| `-c` | Compress input |
| `-d` | Decompress an archive |
| `-i PATH` | Input file, URL, S3 URL, or `-` for stdin |
| `-o PATH` | Output file, S3 URL, or `-` for stdout |
| `-block-size BYTES` | Uncompressed bytes per seekable block |
| `-checkpoint BYTES` | Uncompressed bytes between index snapshots |
| `-key KEY` | AES-256 key in hex or base64 |
| `-verbose` | Print progress and strategy details to stderr |
| `-fuse` | Mount an archive read-only |
| `-wizardconfig` | Run the configuration wizard |

Display the options supported by the installed version:

```bash
nitroarchive -h
```

## How the format works

A `.nar` stream contains:

1. A fixed-size header.
2. Independently compressed Zstandard block records.
3. Periodic complete index snapshots.
4. A complete final index.
5. A fixed-size footer pointing to the final index.

```text
input stream
     |
     v
split into blocks
     |
     v
parallel Zstandard compression
     |
     v
ordered block records
     |
     +--> checkpoint indexes
     |
     v
final index + footer
```

When encryption is enabled, every compressed block and index payload is independently encrypted and authenticated with AES-256-GCM. Public framing remains available for transport and record discovery, while the detailed index requires the key.

## Troubleshooting

### `archive is encrypted; provide a 32-byte AES-256 key`

Set the same key used when creating the archive:

```bash
export NITROARCHIVE_KEY="$(cat nitroarchive.key)"
nitroarchive -d -i encrypted.nar -o restored.img
```

### HTTP source has no valid `Content-Length`

The remote server must provide a valid `Content-Length`. For mounting and efficient random access, it must also honor `Range` requests.

Test the server:

```bash
curl -I https://downloads.example.com/source.nar
curl -i -H 'Range: bytes=0-63' https://downloads.example.com/source.nar
```

The range request should normally return `206 Partial Content`.

### FUSE reports an `allow_other` error

Enable `user_allow_other` in `/etc/fuse.conf`, or run under an account and system configuration permitted to create `allow_other` mounts.

### FUSE cannot mount an incomplete archive

Streaming decompression can recover complete blocks, but mounting requires the final footer and index. Recover the stream first:

```bash
nitroarchive -d -i incomplete.nar -o recovered.img
```

### S3 requests use the wrong URL style

Try path-style addressing for providers that do not support virtual-hosted bucket URLs:

```bash
export NITROARCHIVE_S3_PATH_STYLE=true
```

### Preserve sparse disk images when restoring

NitroArchive outputs a byte stream and does not recreate sparse extents automatically. On supported systems, pipe through a sparse-aware copy:

```bash
nitroarchive -d -i disk.img.nar | cp --sparse=always /dev/stdin disk.img
```

## Security notes

- Prefer `NITROARCHIVE_KEY` over `-key` to reduce exposure in process listings.
- Do not commit `.nitroarchiverc`, `.env` files, keys, or cloud credentials.
- Use short-lived cloud credentials where possible.
- Keep encryption keys in a secret manager and back them up separately.
- Treat mounted remote archives as dependent on the availability and integrity of the remote service.
- Verify important restores with a cryptographic checksum.

Example verification workflow:

```bash
sha256sum source.img > source.img.sha256
nitroarchive -c -i source.img -o source.img.nar
rm source.img
nitroarchive -d -i source.img.nar -o source.img
sha256sum -c source.img.sha256
```

## Current limitations

- Each archive represents one byte stream.
- FUSE exposes only one file, `data`.
- HTTP output is not supported; use a local file, stdout, or S3 output.
- Random access requires a complete final index and footer.
- FUSE availability and permissions depend on the host operating system.


