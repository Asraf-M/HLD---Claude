# System Design: YouTube (Beginner-Friendly Guide)

---

## What Are We Building?

A platform where users can **upload videos** and **watch videos** — at massive scale.

**Real-world context (2020 numbers):**
- 2 billion monthly active users
- 5 billion videos watched per day
- 500 hours of video uploaded every minute
- 37% of all mobile internet traffic is YouTube

> **Why is this hard?**  
> Storing and serving video is orders of magnitude harder than text or images. A single 1080p video can be 1 GB. Multiply that by millions of uploads. Then think about billions of people watching, on different devices, at different network speeds, from every country on earth. Every piece of this needs a different design.

---

## Step 1: Understand What We're Building

### Core Features

| Feature | Detail |
|---------|--------|
| Upload videos | Max 1 GB per video |
| Watch videos | Smooth streaming on any device |
| Change quality | 360p, 720p, 1080p, 4K |
| Platforms | Mobile, Web, Smart TV |
| Scale | 5 million DAU |
| Daily storage needed | ~150 TB |
| CDN cost estimate | ~$150,000/day |

### Two Main Flows

Everything in YouTube boils down to two flows:

1. **Upload Flow** — Creator uploads a video → system processes it → makes it available
2. **Watch Flow** — Viewer clicks play → video streams smoothly to their device

---

## Step 2: High-Level Design

<img src="../../system-design-notes/14. Youtube/images/high-level-design.png" alt="High Level Design" width="450">

The key insight in the diagram: **video streaming and everything else go through separate paths**.

```
User
 ├── [CDN]          ← for streaming video (fast, cached at edge)
 └── [API Servers]  ← for everything else (uploads, likes, comments, search)
```

> **Question:** Why can't the API servers also serve video?  
> **Answer:** A 10-minute 1080p video is ~300 MB. If you serve it from your API servers, each concurrent viewer consumes 300 MB of bandwidth from your server cluster. With millions of concurrent viewers, you'd need petabytes of bandwidth from your own data centers — impossibly expensive. A CDN distributes videos to edge nodes worldwide, so each viewer gets video from a server near them, and you pay the CDN instead of running your own bandwidth infrastructure.

### Components at a Glance

| Component | Role |
|-----------|------|
| **Client** | Smartphones, browsers, smart TVs |
| **CDN** | Streams videos from edge nodes near users |
| **API Servers** | Handle uploads, metadata, auth, comments |
| **Original Storage** | Raw uploaded videos (S3 blob storage) |
| **Transcoding Servers** | Convert raw video into multiple formats/resolutions |
| **Transcoded Storage** | Processed videos ready for CDN distribution |
| **Metadata DB** | Video title, description, duration, status |
| **Metadata Cache** | Fast access to frequently-read metadata |

---

## Step 3: Video Upload Flow — Step by Step

<img src="../../system-design-notes/14. Youtube/images/video-uploading-flow.png" alt="Video Uploading Flow" width="500">

When a creator uploads a video, **two things happen in parallel**:

### Track 1: The Video File

```
① User uploads raw video → Original Storage (S3)
② Transcoding Servers pull the raw video and process it
③ Two things happen simultaneously:
   ③a. Transcoded videos → Transcoded Storage → CDN (ready to stream!)
   ③b. "Transcoding done" event → Completion Queue → Completion Handler
④ Completion Handler updates Metadata DB + notifies the user
```

### Track 2: The Metadata (Runs Parallel to Track 1)

<img src="../../system-design-notes/14. Youtube/images/metadata-upload.png" alt="Metadata Upload" width="350">

```
Simultaneously while the video uploads:
User also sends metadata → Load Balancer → API Servers
                                              ↓
                                      Metadata Cache + Metadata DB
                                      (title, description, tags, etc.)
```

> **Question:** Why update metadata and upload video at the same time?  
> **Answer:** Speed. The raw video upload takes minutes. If you waited for the video to finish before saving the title and description, the whole operation would feel slow. By running them in parallel, the metadata is saved instantly while the large file transfer continues in the background.

### Pre-Signed URLs — How Upload Authorization Works

<img src="../../system-design-notes/14. Youtube/images/pres-signed-urls.png" alt="Pre-Signed URLs" width="450">

> **Question:** Why not just let the user upload directly to S3? And why not route the upload through your API servers?

**Problem with direct S3 upload:** Anyone could upload anything to your S3 bucket without authentication.

**Problem with routing through API servers:** A 1 GB video passing through your API servers wastes their resources and bandwidth.

**Solution — Pre-Signed URLs:**

```
① User → POST /upload (just a small request to API servers)
② API Server validates the user, generates a temporary signed URL for S3
③ API Server → returns pre-signed URL to user
④ User uploads the video DIRECTLY to S3 using the pre-signed URL
   (API servers are not in the upload path — just S3 and the user)
```

> **Analogy:** You want to send a large package to a warehouse. Instead of having the shipping company (API server) carry it through their office, they give you a direct delivery pass (pre-signed URL) to drive it straight to the warehouse (S3). The office verified who you are, but the physical package never goes through them.

---

## Step 4: Video Streaming Flow

<img src="../../system-design-notes/14. Youtube/images/video-streaming-flow.png" alt="Video Streaming Flow" width="350">

Watching a video is simple from a design perspective: **videos are served directly from the CDN**.

```
User clicks play
      ↓
Nearest CDN edge node
      ↓
Video streams to device
```

No API servers involved. No database queries. Just CDN → user.

### How Does Quality Switching Work? (Adaptive Bitrate Streaming)

> **Question:** How does YouTube automatically switch from 1080p to 360p when your connection gets slow?

This is called **Adaptive Bitrate Streaming (ABR)** and uses protocols like **HLS** or **DASH**.

The key idea: the video is not one big file. It's split into **small chunks** (2-10 seconds each), and each chunk exists in **multiple quality versions**.

```
video.m3u8 (manifest file — tells the player what chunks are available)
├── chunk_001_1080p.ts
├── chunk_001_720p.ts
├── chunk_001_360p.ts
├── chunk_002_1080p.ts
├── chunk_002_720p.ts
├── chunk_002_360p.ts
...
```

The player monitors your network speed. If it drops, it requests the next chunk in 360p instead of 1080p — without pausing playback.

> **Analogy:** Instead of downloading the whole book before reading, you get it one page at a time. If the delivery gets slow, you temporarily get a simplified version of the page (fewer images) — but you never stop reading.

---

## Step 5: Video Transcoding — The Hard Part

This is the most complex piece. When you upload a raw `.mov` file from your iPhone, YouTube can't just serve that file directly. It needs to:

1. **Convert to standard formats** (H.264, VP9) — different devices support different codecs
2. **Create multiple resolutions** (4K, 1080p, 720p, 480p, 360p)
3. **Generate thumbnails**
4. **Add watermarks**
5. **Separate audio and encode it**

All of this is called **transcoding**. It's CPU-intensive and takes time.

> **Question:** Why can't we just keep the original file and serve it?  
> **Answer:** Three reasons: (1) Raw formats (MKV, MOV) aren't supported by all browsers and devices; (2) A raw 4K file is too large to stream on a 3G connection; (3) The original file has no thumbnails or watermarks. Transcoding solves all three simultaneously.

### DAG Model — How Transcoding Tasks Are Organized

<img src="../../system-design-notes/14. Youtube/images/dag-video-transcoding.png" alt="DAG Video Transcoding" width="500">

Transcoding isn't a single task — it's many tasks, some of which can run in **parallel**.

**DAG = Directed Acyclic Graph** — a way to define tasks where:
- Some tasks depend on others (must happen in order)
- Some tasks are independent (can happen simultaneously)

```
Original Video
    ├── Video stream →  [Video Encoding (1080p)]
    │                   [Video Encoding (720p)] ← all run in parallel
    │                   [Video Encoding (360p)]
    │                   [Thumbnail generation]
    │                   [Watermark]
    ├── Audio stream → [Audio Encoding]
    └── Metadata      → [Metadata extraction]
                                   ↓
                           [Assemble all outputs]
```

> **Question:** Why not just do these one by one?  
> **Answer:** A 10-minute video might take 5 minutes to encode in 1080p. If you also had to encode 720p and 360p sequentially, total time = 15 minutes. By running all 3 encodings in parallel, total time = ~5 minutes. Parallel processing is essential at YouTube's scale.

---

## Step 6: Video Transcoding Architecture — All the Components

<img src="../../system-design-notes/14. Youtube/images/video-transcoding-architecture.png" alt="Video Transcoding Architecture" width="550">

The transcoding pipeline has 5 components that work together:

### 1. Preprocessor — Split the Video into Chunks

<img src="../../system-design-notes/14. Youtube/images/video-split.png" alt="Video Split" width="500">

The raw video is split into small chunks called **GOPs (Group of Pictures)**.

> **Why split?**  
> (1) **Parallel encoding:** Multiple workers can encode different chunks simultaneously → much faster total encoding time  
> (2) **Resumable processing:** If a worker crashes mid-encoding, only the chunk it was working on needs to be re-done — not the entire video  
> (3) **Adaptive streaming:** HLS/DASH needs the video pre-chunked anyway

The preprocessor also reads a **DAG config file** (written by engineers) that defines what tasks to run:

<img src="../../system-design-notes/14. Youtube/images/dag-config.png" alt="DAG Config" width="450">

```
task { name: 'download-input', type: 'Download' ... next: 'transcode' }
task { name: 'transcode', type: 'Transcode' ... }
```

> Think of the DAG config like a recipe. The preprocessor reads it and decides: "For this video, I need to download it, then transcode it, then generate a thumbnail, then add a watermark."

### 2. DAG Scheduler — Plan the Work

<img src="../../system-design-notes/14. Youtube/images/dag-scheduler.png" alt="DAG Scheduler" width="450">

The DAG Scheduler takes the list of tasks from the preprocessor and organizes them into **stages**:

- **Stage 1:** Split the video into video stream, audio stream, and metadata (must happen first)
- **Stage 2:** All encoding tasks run in parallel (video encoding, thumbnail, audio encoding)

```
Stage 1 (sequential):
  Original video → video stream, audio stream, metadata

Stage 2 (parallel — all run simultaneously):
  video stream → video encoding (1080p, 720p, 360p)
  video stream → thumbnail
  audio stream → audio encoding
  metadata    → (saved directly)
```

The scheduler puts each task into the task queue for the Resource Manager to execute.

### 3. Resource Manager — Assign Work to Workers

<img src="../../system-design-notes/14. Youtube/images/resource-manager.png" alt="Resource Manager" width="550">

The Resource Manager is like a **job dispatcher**. It has 3 queues:

| Queue | What it holds |
|-------|-------------|
| **Task Queue** | Tasks waiting to be executed (priority queue — urgent tasks first) |
| **Worker Queue** | Available workers and their current utilization |
| **Running Queue** | Currently running tasks + which worker is doing them |

The **Task Scheduler** inside the Resource Manager:
1. Picks the highest-priority task from the Task Queue
2. Picks the least-busy worker from the Worker Queue
3. Assigns the task to that worker
4. Moves it to the Running Queue

> **Analogy:** The Task Scheduler is like a manager in a call center. They look at the queue of waiting calls (Task Queue), see which agents are free (Worker Queue), and route the call to the best available agent.

### 4. Task Workers — Do the Actual Work

<img src="../../system-design-notes/14. Youtube/images/task-worker.png" alt="Task Workers" width="250">

Different workers specialize in different tasks:

| Worker type | What it does |
|-------------|-------------|
| **Encoder** | Convert video to H.264/VP9 at different resolutions |
| **Thumbnail** | Extract frames, generate preview images |
| **Watermark** | Overlay the channel logo/identifying info |
| **Merger** | Combine encoded video + audio + metadata into final file |

Workers run on GPU-enabled servers (encoding is heavily GPU-accelerated).

### 5. Temporary Storage — Handle Failures Gracefully

Between each step, intermediate data (chunks, partially encoded files) is saved to temporary storage. If a worker crashes mid-way:
- We don't restart from scratch
- We restart from the last saved checkpoint

> **Analogy:** Like saving your progress in a video game. If the game crashes, you reload from the last save point, not from the beginning.

### Complete Transcoding Flow

```
Raw video → [Preprocessor] (split into GOPs + read DAG config)
                  ↓
           [DAG Scheduler] (organize tasks into stages)
                  ↓
           [Resource Manager] (assign tasks to workers)
                  ↓
           [Task Workers] (encode, thumbnail, watermark, merge)
                  ↓  ↑ (save intermediate data)
           [Temporary Storage]
                  ↓
           [Encoded Video] → Transcoded Storage → CDN
```

---

## Step 7: Message Queues — Decoupling the Pipeline

<img src="../../system-design-notes/14. Youtube/images/message-queue1.png" alt="Message Queue 1" width="500">

<img src="../../system-design-notes/14. Youtube/images/message-queue2.png" alt="Message Queue 2" width="450">

Each stage in the pipeline is connected by a message queue (like Kafka or SQS). Instead of one stage calling the next directly, it drops a message in a queue and the next stage picks it up.

```
Upload → [Queue] → Download Module → [Queue] → Encoding Module → [Queue] → ...
```

> **Question:** Why all the queues? Why not just call the next service directly?  
> **Answer:** Three reasons:  
> (1) **Resilience:** If the encoding service crashes, the message stays in the queue. When it recovers, it picks up where it left off — nothing is lost  
> (2) **Speed spikes:** If 1,000 videos are uploaded at the same moment, they all go into the queue. Encoding workers process them steadily. Without a queue, 1,000 simultaneous encoding requests would crash the encoding servers  
> (3) **Independent scaling:** You can add more encoding workers when the queue is full, without touching the upload service

---

## Step 8: Cost Optimizations

YouTube's CDN cost is estimated at ~$150,000/day. How do you reduce this?

### Don't Store Everything on the CDN

| Video category | Strategy | Why? |
|---------------|----------|------|
| Popular videos (top 20%) | CDN edge cache | Millions of views/day — worth the CDN cost |
| Less popular (middle 30%) | Regional CDN / high-capacity servers | Fewer viewers, cheaper to serve from origin |
| Long-tail (bottom 50%) | On-demand transcoding from cold storage | Rarely watched — don't waste CDN space or even pre-transcode |

> **The 80/20 rule:** ~20% of videos account for ~80% of all views. CDN caching that 20% gives you most of the benefit at a fraction of the cost.

### Other Cost Strategies

1. **Build your own CDN** — YouTube/Google built their own CDN infrastructure and peering agreements with ISPs, eliminating third-party CDN costs
2. **Regionalize content** — A Korean-language video is unlikely to be watched in Brazil. Only distribute it to CDN nodes in Asia
3. **Encode on-demand** — For a video that's watched once a year, don't pre-encode 5 resolutions. Transcode on the first request, cache the result

---

## Step 9: Error Handling

### Recoverable Errors

> **Scenario:** A transcoding worker fails halfway through encoding chunk 7 of a video.

**Solution:** Retry logic + message queue:
- The worker's message lease expires (it didn't acknowledge completion)
- The queue makes the message visible again
- Another worker picks it up and processes chunk 7 again
- Only that chunk is re-done — not the entire video

### Non-Recoverable Errors

> **Scenario:** User uploads a corrupted video file that can't be decoded.

**Solution:** Detect early + fail fast:
- The Preprocessor validates the file format and codec
- If invalid/corrupt, stop immediately and return an error code to the user
- Don't let a bad file waste hours of transcoding worker time

---

## Step 10: Full Architecture Summary

```
                     UPLOAD PATH
                     ──────────
Creator uploads video
        |
        ├── POST /upload → API Servers → returns pre-signed URL
        │
        └── Creator uploads directly to [Original Storage (S3)]
                        |
                  [Message Queue (Kafka)]
                        |
              ┌─────────────────────────┐
              │  TRANSCODING PIPELINE   │
              │  Preprocessor           │
              │  → DAG Scheduler        │
              │  → Resource Manager     │
              │  → Task Workers         │
              │  → Temporary Storage    │
              └──────────┬──────────────┘
                         |
              [Transcoded Storage]──────→ [CDN]
                         |
              [Completion Queue] → [Completion Handler]
                                         |
                              [Metadata DB + Cache]
                              "Video is ready!" notification

                     WATCH PATH
                     ──────────
Viewer clicks play
        |
        ├── Metadata → API Servers → Metadata Cache/DB
        │              (title, available qualities, description)
        └── Video → [CDN Edge Node] → streams to device
                    (no API servers involved at all)
```

---

## Summary: The "Why" Behind Each Decision

| Decision | Why? |
|----------|------|
| CDN for video delivery | Serve from edge nodes near users; origin can't handle billions of viewers |
| Pre-signed URLs for upload | Don't route large files through API servers; authorize without exposing storage |
| Original + transcoded separate storage | Raw files preserved; CDN-ready files in transcoded storage |
| Transcoding pipeline with DAG | Video, audio, thumbnail tasks can run in parallel; faster processing |
| Message queues between pipeline stages | Resilience to crashes; absorb upload spikes; independent scaling |
| GOP-based video splitting | Parallel encoding of chunks; resumable processing; required for HLS/DASH |
| HLS/DASH adaptive streaming | Auto-switch quality based on network speed without pausing |
| Temporary storage between stages | Resume from checkpoint on worker failure; no full restart |
| Cold storage for long-tail videos | 80% of content is rarely watched; CDN cost not justified |
| GPU task workers | Video encoding is mathematically intensive; GPUs are 50x faster than CPUs for this |

---

## Quick Interview Q&A Cheat Sheet

**Q: How does video upload work?**  
A: User requests upload → API server authenticates and returns a pre-signed S3 URL → user uploads directly to S3 (bypassing API servers) → a message is published to Kafka → transcoding pipeline picks it up asynchronously. In parallel, metadata (title, description) is saved to the DB.

**Q: Why is transcoding needed?**  
A: (1) Raw formats (MKV, MOV) not supported everywhere; (2) Raw 4K too large for 3G users; (3) Need multiple resolutions for adaptive bitrate streaming; (4) Need thumbnails and watermarks. Transcoding converts one raw file into 10+ usable files.

**Q: How does adaptive quality switching work?**  
A: HLS/DASH splits video into 6-second chunks, each available in multiple qualities. A manifest file lists all chunks and URLs. The player monitors network speed, requests the appropriate quality chunk for each segment. User sees smooth playback instead of buffering.

**Q: What is the DAG model in transcoding?**  
A: A Directed Acyclic Graph defines which transcoding tasks depend on each other (sequential) and which can run simultaneously (parallel). Video encoding at 1080p/720p/360p all run in parallel; audio encoding runs in parallel too. The DAG ensures maximum parallelism.

**Q: How do you handle transcoding failures?**  
A: Each video is split into chunks stored in temporary storage. If a worker fails mid-chunk, its message lease expires, another worker picks it up from the queue and re-processes just that chunk. Exponential backoff for retries. After N failures, move to Dead Letter Queue for manual inspection.

**Q: How do you reduce CDN costs at YouTube's scale?**  
A: (1) Only cache popular content on CDN edge; (2) Long-tail videos stored in cold storage, transcoded on demand; (3) Regionalize distribution (Korean content stays in Asian CDN nodes); (4) Build your own CDN and peer directly with ISPs (Google does this).

**Q: How does video seek (jump to 5:30) work?**  
A: Video is split into fixed-duration segments. A manifest file maps timestamps to segment URLs. The player calculates which segment contains the target timestamp, requests that CDN URL directly. Pure client-side operation — no server state needed.
