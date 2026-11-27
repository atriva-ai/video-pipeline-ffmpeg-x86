# Sharing decoded frames with inference containers

The video pipeline writes JPEGs to `/app/frames` inside the producer container.
Use one volume for the producer and its inference consumer. The mount point
can differ inside the consumer.

## Frame layout

```text
/app/frames/
└── camera_1/
    ├── frame_0001.jpg
    ├── frame_0002.jpg
    └── frame_0003.jpg
```

Each configured camera ID has its own directory. FFmpeg uses
`frame_%04d.jpg`, so the number has at least four digits (it grows past
`9999`). The number is a sequence within a decode, not a timestamp or a
durable globally unique ID. A new decode for a camera can reuse a filename.

## Compose mounts

Add these mounts to the services in your Compose file. This is a fragment for
existing services, not a complete Compose project. Configure the inference
process to read the consumer mount point, `/shared/frames` in this example.
For [Atriva AI Inference](https://github.com/atriva-ai/ai-inference-ov/blob/1985836d90a4cbcfddb75e5de3f295a6619d4e97/app/shared_data.py#L6-L12),
set `SHARED_FRAMES_PATH` as shown because it defaults to `/app/frames`.

```yaml
services:
  video-pipeline:
    volumes:
      - shared-frames:/app/frames
  ai-inference:
    environment:
      SHARED_FRAMES_PATH: /shared/frames
    volumes:
      - shared-frames:/shared/frames:ro
volumes:
  shared-frames:
```

## Reading frames

A consumer can poll the directory and look at modification times. Do not rely
on lexicographic filename order after frame 9999. A frame can disappear between
listing and reading during a restart or stop, or fail to decode while FFmpeg
is writing it. Skip it and try on the next poll. If the consumer needs to
remember what it has processed, include the modification time, not only the
filename.

This example uses Pillow to read from the mounted volume and returns a detached
image, or `None` if no readable frame is present:

```python
from pathlib import Path

from PIL import Image


def latest_frame(camera_id: str, root: Path = Path("/shared/frames")):
    candidates = []
    for path in (root / camera_id).glob("frame_*.jpg"):
        try:
            candidates.append((path.stat().st_mtime_ns, path))
        except FileNotFoundError:
            continue
    for _, path in sorted(candidates, reverse=True):
        try:
            with Image.open(path) as image:
                return image.copy()
        except OSError:
            continue
    return None
```

## Cleanup and retention

- Starting a new decode clears prior JPEGs for that camera. Stopping a decode
  clears that camera's JPEGs too. The stale-frame restart path also clears the
  old set before relaunching the decoder.
- Cleanup removes the JPEG files, not the camera directory itself. An existing
  directory may have zero frames, including before the first write.
- There is no per-frame TTL or maximum count during a healthy decode. Treat the
  volume as working storage, monitor its free space, and store anything that
  must be retained separately before stopping or restarting the producer.
