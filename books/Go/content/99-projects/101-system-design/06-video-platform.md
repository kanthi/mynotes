---
title: 'Design Lab: A Video Upload and Streaming Platform'
---

# Design Lab: A Video Upload and Streaming Platform

Build the Go control plane around an adaptive-bitrate media pipeline: accept resumable source uploads, fan out transcoding jobs across a bounded worker DAG, publish immutable [HLS (RFC 8216)](https://www.rfc-editor.org/info/rfc8216/) manifests atomically, and serve segments through a CDN. Go orchestrates storage, state transitions, and external encoders (`ffmpeg`); it does not implement video codecs.

![Video platform upload and adaptive bitrate transcoding DAG pipeline to CDN edge](../../../images/diagram-go-video-transcoding-dag.svg)

## State machine, endpoints, and schema

Expose `POST /v1/videos` (returns presigned upload session), `POST /v1/videos/{id}/complete`, `GET /v1/videos/{id}`, and an authorized playback manifest endpoint `GET /v1/videos/{id}/master.m3u8`.

```text
created -> uploaded -> processing (gen N) -> ready (manifest vN)
                           |
                           +-> failed -> processing (gen N+1)
```

### Authoritative schema (`migrations/001_videos.sql`)

```sql
CREATE TABLE videos (
    id                UUID PRIMARY KEY,
    owner_id          UUID NOT NULL,
    visibility        TEXT NOT NULL DEFAULT 'PRIVATE', -- PUBLIC | UNLISTED | PRIVATE
    source_blob_id    TEXT NOT NULL DEFAULT '',
    processing_gen    BIGINT NOT NULL DEFAULT 0,
    status            TEXT NOT NULL DEFAULT 'CREATED',
    active_manifest   TEXT NOT NULL DEFAULT '',
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE renditions (
    video_id       UUID NOT NULL REFERENCES videos(id),
    processing_gen BIGINT NOT NULL,
    profile        TEXT NOT NULL, -- 1080p | 720p | 480p | 360p
    status         TEXT NOT NULL DEFAULT 'PENDING',
    playlist_key   TEXT NOT NULL DEFAULT '',
    bytes_written  BIGINT NOT NULL DEFAULT 0,
    PRIMARY KEY (video_id, processing_gen, profile)
);
```

Only a verified upload completion inserts the transcode outbox event and increments `processing_gen`. Every rendition writes segments to a **generation-scoped immutable prefix** (`videos/{id}/gen-{N}/{profile}/seg-0001.ts`). When all required renditions finish, the coordinator writes `master.m3u8` and flips `active_manifest` with `WHERE id = $1 AND processing_gen = $2`—so a slow worker from generation `N-1` can never overwrite generation `N`.

## Go worker boundary: Safe `exec.CommandContext` and bounded DAG pool

Launch `ffmpeg` with fixed binary paths, explicit argument slices, and **zero shell interpolation** (`sh -c` is a command-injection vulnerability). Enforce per-job CPU/wall-clock deadlines, disk quotas, and clean process-group teardown on timeout.

```go
package transcoder

import (
	"bytes"
	"context"
	"fmt"
	"os/exec"
	"sync"
	"time"
)

type Profile struct {
	Name       string // e.g. "720p"
	Resolution string // e.g. "1280x720"
	Bitrate    string // e.g. "2800k"
}

type Result struct {
	Profile     string
	PlaylistKey string
	Err         error
}

// RunDAG fans out rendition encodes across at most maxWorkers concurrent ffmpeg
// subprocesses, enforcing a hard per-rendition timeout and bounded stderr capture.
func RunDAG(ctx context.Context, srcPath, outDir string, profiles []Profile, maxWorkers int) []Result {
	jobs := make(chan Profile, len(profiles))
	for _, p := range profiles {
		jobs <- p
	}
	close(jobs)

	var (
		mu      sync.Mutex
		results = make([]Result, 0, len(profiles))
		wg      sync.WaitGroup
	)
	for i := 0; i < maxWorkers; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for p := range jobs {
				err := encodeHLS(ctx, srcPath, outDir, p)
				mu.Lock()
				results = append(results, Result{
					Profile:     p.Name,
					PlaylistKey: fmt.Sprintf("%s/%s/index.m3u8", outDir, p.Name),
					Err:         err,
				})
				mu.Unlock()
			}
		}()
	}
	wg.Wait()
	return results
}

func encodeHLS(parent context.Context, srcPath, outDir string, p Profile) error {
	ctx, cancel := context.WithTimeout(parent, 10*time.Minute)
	defer cancel()

	playlist := fmt.Sprintf("%s/%s/index.m3u8", outDir, p.Name)
	cmd := exec.CommandContext(ctx, "ffmpeg",
		"-y", "-i", srcPath,
		"-vf", "scale="+p.Resolution,
		"-b:v", p.Bitrate,
		"-hls_time", "4",
		"-hls_playlist_type", "vod",
		playlist,
	)
	var stderr bytes.Buffer
	cmd.Stderr = &stderr
	if err := cmd.Run(); err != nil {
		return fmt.Errorf("ffmpeg %s failed: %w (tail: %.200s)", p.Name, err, stderr.String())
	}
	return nil
}
```

## Capacity math and delivery architecture

- **Storage**: One 10-minute 1080p upload (~500 MB source) produces 4 HLS renditions (`1080p` 5 Mbps + `720p` 2.8 Mbps + `480p` 1.4 Mbps + `360p` 800 kbps ≈ 10 Mbps combined ≈ `750 MB` total). At 10,000 uploads/day, new storage grows at **~12.5 TB/day** before lifecycle tiering.
- **Egress**: 100,000 concurrent viewers averaging 3 Mbps require **300 Gbps** egress—impossible from a single origin without a multi-tier CDN caching immutable `.ts`/`.m4s` segments (`Cache-Control: public, max-age=31536000, immutable`).
- **Private video authorization**: Serve manifests via short-lived signed CDN cookies or presigned segment URLs so authorization checks stay consistent between playlists and segments and revocations take effect within the token TTL.

## Failure evidence

Run `go test -race ./...`:

1. **Mid-rendition crash**: Kill a worker halfway through `720p`, redeliver the job, and verify `master.m3u8` is never published with missing segments.
2. **Generation fencing**: Run generation 1 and generation 2 concurrently; delay generation 1 so it finishes last and assert the `WHERE processing_gen = $2` guard rejects its stale publication.
3. **Timeout & temp cleanup**: Inject a stalled encoder and verify `exec.CommandContext` terminates it cleanly and removes scratch files.
4. **Authorization & cache boundary**: Switch a video from `PUBLIC` to `PRIVATE` and verify expired signed URLs return `403`. Measure time-to-ready, transcode queue age, and playback startup latency.

## Reference materials

- [RFC 8216 — HTTP Live Streaming (HLS)](https://www.rfc-editor.org/info/rfc8216/)
- [Amazon S3 Multipart Upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Subprocess orchestration (`os/exec`)](../../09-cli-tools-in-go/307-os-exec-orchestration.md)
- [S3-like object storage lab](05-object-storage.md)
