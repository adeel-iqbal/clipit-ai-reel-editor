<div align="center">

# Clipit

**Searchable clip intelligence for a podcast library. Find the best moments, turn them into finished 9:16 reels, post them.**

[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-Uvicorn-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)](https://react.dev/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pgvector-4169E1?logo=postgresql&logoColor=white)](https://github.com/pgvector/pgvector)
[![Claude](https://img.shields.io/badge/Claude-Haiku%20%2B%20Sonnet-D97757?logo=anthropic&logoColor=white)](https://www.anthropic.com/)
[![FFmpeg](https://img.shields.io/badge/FFmpeg-video-007808?logo=ffmpeg&logoColor=white)](https://ffmpeg.org/)
[![Docker](https://img.shields.io/badge/Docker-single%20image-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)

</div>

<img src="Clipit%20Screenshots/1%20Login.png" alt="Clipit sign-in screen" width="100%"/>

> [!NOTE]
> **README-only showcase.** Clipit was built at [East River Digital](https://eastriverdigital.com) (ERD) and runs in production there, so the source code is private. This repo documents what the system does, how it is built and the numbers it has produced. A live walkthrough is available on request. Current version: v3.2 (Nov 2026).

## 🧭 Overview

Editors were spending hours scrubbing long episodes to find one good 45-second moment, then hours more editing it. Clipit reads every second of a video library and makes it searchable by topic, trend or plain-language chat. It ranks the best moments and edits them into captioned, hooked, branded 9:16 shorts that post straight to YouTube and Instagram.

Type "petrol levy", get ranked clips in seconds, click **Make Short**, get a finished reel.

| Aspect | Detail |
|---|---|
| **Users** | Content teams. The first workspace is DigiTales, ERD's podcast. Multi-client since v3.0 (Oct 2026). |
| **Input** | YouTube episodes, playlists and channels, uploaded files up to 5 GB, and other creators' public channels (download only, credited, never published from Clipit) |
| **Output** | Scored clip cards, finished 9:16 shorts (captions, hook, B-roll, name tag, branding), per-person thumbnails, scheduled posts |
| **Languages** | Urdu-English code-switching, Roman Urdu and English |

## 📊 By the numbers

| Metric | Value |
|---|---|
| DigiTales library | 194 episodes, 16,000+ searchable clips |
| Clip unit | ~45 s, sentence-aligned, no gaps |
| First external client library | 30 videos → 1,720 clips |
| Search | 48 candidates scored once, 8 cards per page, score floor 5.0 / 10 |
| Cost per finished short | ~$1, mostly the reframe step |
| Full short render | ~29 min → ~12 min |
| B-roll overlay pass | 8.4 min → 108 s |
| Instagram insights load | 45 s → ~6 s |
| Max upload | 5 GB, sent in 8 MB resumable pieces |

## 🏗️ Architecture

The browser talks only to FastAPI. Fast work (search, chat) runs inside the request. Slow work (imports, renders, posts) becomes a job row that a background worker claims.

```mermaid
flowchart LR
    U["Browser<br/>React 19 + Vite"] -->|"request"| A["FastAPI<br/>auth sets workspace"]
    A -->|"JSON or job id"| U
    A -->|"SQL"| D[("Supabase<br/>Postgres + pgvector<br/>Storage, Auth, job tables")]
    A -->|"prompts"| L["AI services<br/>Claude, OpenAI<br/>ElevenLabs, Perplexity"]
    W["Workers<br/>ingest, shorts,<br/>schedule, thumbnails"] -->|"claim with SKIP LOCKED"| D
    W -.-> L
    W --> M["Media and posting<br/>Klap, ZapCap, Pexels<br/>YouTube, Meta"]
```

**Key idea:** the UI never waits on a long job. The API writes a `queued` row, a worker claims it with `FOR UPDATE SKIP LOCKED` and writes progress to the row, and the browser polls every 3 seconds. Leaving the page breaks nothing.

## ⚙️ How it works

### 1. Ingestion: video to searchable clips

```mermaid
flowchart TB
    subgraph IN["1 · Read the episode"]
        direction LR
        A["YouTube link,<br/>playlist or CSV"] --> B["Transcribe<br/>ElevenLabs Scribe"]
        B --> C["Chunk in code<br/>~45 s, sentence-aligned"]
        C --> D["Enrich<br/>Haiku: English text, tags"]
    end
    subgraph OUT["2 · Load and verify"]
        direction LR
        E["Embed and load<br/>text-embedding-3-small"] --> F{"Verify<br/>coverage"}
        F -->|"pass"| G["Delete local audio,<br/>write episode digest"]
        F -->|"fail"| H["Clean up<br/>partial rows"]
    end
    IN --> OUT
```

- ✂️ **Chunking happens in code, not in the LLM.** When the model chose boundaries, clips looked complete but hid gaps. Back-to-back sentence-aligned windows make a gap impossible by construction. The LLM only translates and tags.
- 🌐 **Translate, then embed.** Guests mix Urdu and English, and Roman Urdu spelling varies ("mehngai", "mehangai"). Embedding the English translation lets a search for "inflation" find a clip where someone said "mehngai".
- ✅ **Verification gate.** Full transcript stored, no speech gap over 15 s, no overlaps, and transcript length at least 90% of YouTube's own duration (this catches half-downloaded files). Local audio is deleted only after every check passes.
- 🔁 Every step skips work already done, so a run can stop and resume safely.

### 2. Retrieval: how clip cards are found

Retrieval finds candidates, the LLM judges and writes, and code computes every timestamp and link.

```mermaid
flowchart TB
    subgraph R1["1 · Retrieve"]
        direction LR
        Q["Topic<br/>'petrol levy'"] --> X["Query expansion<br/>Haiku"]
        X --> H["Hybrid search<br/>pgvector + full-text<br/>16 hits per term"]
        H --> P["48 candidates<br/>merged, deduped"]
    end
    subgraph R2["2 · Rank and write"]
        direction LR
        S["Score once<br/>Sonnet: relevance,<br/>virality, quality"] --> F["Frozen ranking<br/>30 min cache"]
        F --> C["8 cards per page<br/>cuts computed in code"]
    end
    R1 --> R2
```

| Stage | Detail |
|---|---|
| **Query expansion** | Haiku adds related terms: synonyms, other angles, Urdu phrasing |
| **Hybrid search** | pgvector cosine search (ivfflat, 1536 dims) finds meaning. Postgres full-text search (GIN index, `ts_rank`) finds exact names like "IMF". Guest, category, season and workspace are SQL `WHERE` filters. |
| **Score once** | One Claude Sonnet call scores the whole pool from 0 to 10: `composite = 0.40 × relevance + 0.35 × virality + 0.25 × quality`. Anything under 5.0 is never shown. |
| **Frozen ranking** | The sorted pool is cached for 30 minutes per topic, filter set and workspace. **Show more** walks down the same list, with no re-scoring and no repeats. |
| **Write cards** | Sonnet gets the top 8 clips with their neighbouring chunks and the whole-episode digest, plus the team's taste profile and the channel's real YouTube performance. It returns a title, the moment type, why it works and a hook. |
| **Cuts in code** | Haiku proposes a reel cut within ±22 s (15 to 55 s long). Code snaps it to real word timings, extends it through connectors ("and", "but", "so", "aur", "to") and never ends on a dangling word. |

- 🧠 **Personalisation without thumbs.** Searches, saves and shorts feed a taste profile. Home recommends the nearest unused clips to the average embedding of what the team saved and shorted (0.65 closeness + 0.35 clip strength, one per episode).
- 🔎 **Evaluation.** Real team queries are checked by hand against what an editor would pick, alongside usage (what gets saved and turned into shorts).

### 3. Make Short: clip to finished reel

Make Short runs in two stages. The user fixes captions and B-roll before the final render, so the paid reframe never runs twice.

```mermaid
flowchart TB
    subgraph S1["Stage 1 · Prepare"]
        direction LR
        A["Wizard<br/>captions, hook, B-roll,<br/>speaker and brand"] --> B["Range download<br/>Klap reframe<br/>word timings, B-roll picks"]
        B --> C{"You review<br/>captions and B-roll"}
    end
    subgraph S2["Stage 2 · Render and ship"]
        direction LR
        D["Render<br/>captions, hook, name tag,<br/>logo, intro and outro"] --> E["Ship<br/>thumbnails, YouTube,<br/>Instagram, schedule"]
    end
    S1 --> S2
```

| Stage | What happens |
|---|---|
| **Prepare** | Downloads only the clip's range (~15 to 30 MB instead of a ~180 MB video), reframes 16:9 to 9:16 with Klap, runs one ElevenLabs pass for word timings, and proposes B-roll from Pexels that Sonnet vision must score 6+ |
| **Review** | Fix caption words (timings stay exact), swap, prompt, upload or remove B-roll, or just download the reframe |
| **Render** | Captions (ZapCap templates or an in-house libass mode), hook card, name tag, logo, intro and outro, filler cuts, colour grade, browser-safe `yuv420p` |

- 🪝 **Hooks** are 3 to 6 words written by Haiku and checked by a second model for faithfulness, because hooks kept inflating claims. They are drawn with Pillow and placed away from the face (OpenCV).
- 🤫 **Filler cuts** remove ums, repeats and long pauses at word boundaries, but only where ffmpeg `silencedetect` confirms the audio is quiet. Captions and B-roll are remapped to the new timeline.
- 🧩 **Merge** joins up to 8 clips from anywhere: each part is trimmed to word-accurate bounds, loudness-normalised and crossfaded in one ffmpeg pass.
- 🎬 **AI reel editor (v3.2)** is modelled on the team's top 200 Instagram Reels (median 44 s, edited rather than straight cuts). Sonnet picks one continuous stretch, drops weak lines and can add an opening hook line. Code enforces a strong opener, no ending on a question, 30 to 75 s and at most 6 pieces. A failure goes back to the model with the reason, up to 3 times, before a safe continuous cut is used.
- 🖼️ **Thumbnails** use real frames: YuNet finds faces, SFace groups them by person, and Sonnet vision picks the best frame and writes a 3 to 7 word headline from what that person says.

### 4. Publish, schedule, measure

| Feature | How |
|---|---|
| **YouTube** | OAuth and resumable `videos.insert`. A 9:16 video under 3 minutes posts as a Short. Default visibility is Private. |
| **Instagram** | Graph API: create a REELS container, poll until `FINISHED`, then `media_publish`. Collaborators and tagged people are supported. |
| **Calendar** | A worker checks every 60 s and claims due posts with `SKIP LOCKED`. Posting to both platforms creates two rows, so one can fail without the other. |
| **Insights** | YouTube watch time, retention, traffic sources and "Clip this next"; Instagram reach, follower growth and best time to post. Connected stats feed back into ranking. |

### 5. Workspaces and roles

v3.0 turned the internal tool into a multi-client product: one codebase, one database, one deploy.

| Concern | Implementation |
|---|---|
| **Isolation** | Every data table has a `workspace` column. Auth middleware sets the workspace in a Python `contextvar` and every query filters on it. Background threads start through a binder so they keep it. |
| **No id leaks** | Ids in URLs and request bodies are checked against the workspace. A wrong workspace returns 404, not 403. |
| **Sessions** | Supabase Auth checks passwords. Clipit issues an opaque httpOnly session token stored in the database (30 days), so it can be revoked at once. |
| **Access** | Invite-only, with one-time join links valid for 14 days. Plans: `trial`, `active`, `internal`, with seat limits. |
| **Shared data** | Only public data is shared: trend keywords and transcripts of public channel videos. |
| **Migrations** | Additive only (new columns with defaults), so the running build never breaks during a schema change. |

| Permission | Owner | Admin | Member |
|---|:---:|:---:|:---:|
| Use every feature, change own password | ✅ | ✅ | ✅ |
| Invite people, remove members and admins | ✅ | ✅ | ❌ |
| Add or remove owners, change anyone's role | ✅ | ❌ | ❌ |

## ⚖️ Engineering decisions

| Decision | Chose | Over | Why |
|---|---|---|---|
| Vector store | pgvector in Supabase Postgres | Pinecone, Weaviate | Vector search, filters and joins in one SQL query, with no second store to sync |
| Search | Hybrid vector + full-text | Vector only | Embeddings blur exact names like "IMF" |
| Ranking | LLM judge on relevance, virality, quality | Cohere Rerank | A reranker scores relevance only; a content team needs "will this work as a reel" |
| Chunking | Fixed ~45 s windows in code | LLM or topic-based chunks | LLM boundaries left hidden gaps; fixed windows are predictable and gapless |
| Multilingual search | Translate to English, then embed | Multilingual embeddings on raw text | Roman Urdu spelling is inconsistent; translation normalises meaning first |
| Transcription | ElevenLabs Scribe, Whisper as gap-fill | Whisper, Deepgram, AssemblyAI | Best tested on Urdu-English code-switching, with exact word timings |
| Job queue | Postgres table + `SKIP LOCKED` | Celery + Redis | A few jobs at a time, and job state already lives where the UI polls |
| Reframe | Klap API, about $1 a clip | In-house speaker tracking | Weeks of work to match; the time went into edit quality instead |
| Frameworks | Direct SDK calls | LangChain, LlamaIndex | Retrieval is custom SQL with strict JSON schemas, and a framework hides what needs tuning |
| Live updates | 3 s polling | WebSockets | Jobs take minutes, and polling survives refreshes and proxies |
| Frontend | React + Vite build served by FastAPI | Next.js | The app sits behind a login, so server rendering adds nothing, and one container serves both |

## 🐛 Hard problems solved

| Problem | Root cause | Fix |
|---|---|---|
| Clips had hidden gaps inside episodes | The LLM chose cut points and skipped stretches | Chunking moved into code, and a gap check runs before audio is deleted |
| ffmpeg downloads from the server failed with "End of file" | YouTube media URLs are locked to the requesting IP, and ffmpeg can't tunnel HTTPS through an HTTP proxy | yt-dlp resolves the URL through the proxy, proxychains keeps ffmpeg on the same IP, and only the needed range downloads instead of ~180 MB and 11 min per clip |
| Worked locally, failed in the container | yt-dlp's challenge-solver script was cached on the dev machine | Fetched via `--remote-components`, with yt-dlp and deno versions pinned |
| Review video froze on one frame | Klap exports full-range `yuvj420p`, which some browsers freeze on | Always convert to limited-range `yuv420p` before hosting |
| Sentence logic silently failed for a client | Transcripts came back in Hindi or Urdu script, and the checks assumed `[a-z]` | Accept `।` `۔` `؟` as sentence ends and transliterate native script to Roman Urdu |
| Renders failed with "max clients reached" | psycopg2's `with conn` commits but doesn't close | A context manager that waits for a free pool slot and always closes |
| Best moments returned 0 cards, with no error | The JSON schema lacked `additionalProperties: false` at every level, and the 400 was swallowed | Fixed the schema and stopped silent exception handling on the main path |
| Gap-cutting removed real speech (49 s became 33 s) | Word timings missed some words | Cut only where `silencedetect` confirms silence |

## 🛠️ Tech stack

| Layer | Technology |
|---|---|
| **Frontend** | React 19, Vite, Tailwind CSS 4, React Router 7 |
| **Backend** | Python 3.12, FastAPI, Uvicorn, psycopg2 with raw SQL and a connection pool |
| **Data** | Supabase: PostgreSQL, pgvector (ivfflat, cosine), full-text search (GIN), Storage with resumable TUS uploads, Auth |
| **LLMs** | Claude Haiku 4.5 (expansion, translation, cut points), Claude Sonnet 5 and 4.6 (scoring, cards, edits, vision) |
| **Embeddings** | OpenAI text-embedding-3-small, 1536 dims |
| **Speech** | ElevenLabs Scribe, Whisper as fallback |
| **Trends** | Perplexity sonar, Google Trends (pytrends) |
| **Video** | ffmpeg, libass, Pillow, OpenCV (YuNet, SFace), yt-dlp |
| **Media APIs** | Klap (reframe), ZapCap (captions), Pexels (B-roll), gpt-image-2 (thumbnail restyle) |
| **Platforms** | YouTube Data API v3, YouTube Analytics API v2, Meta Graph API |
| **Deploy** | One multi-stage Docker image: Node 22 builds React, Python 3.12 serves API and UI on port 8000 |

## 🖼️ Screenshots

Grouped in the order a clip moves through Clipit. Click any **More from this step** to expand it.

### 1. Find the moment

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/6%20Trends%20-%20trending%20keywords%20and%20clip%20cards.png" alt="Trends: live keywords, each one opens ranked clip cards" width="100%"/>
      <br/><sub>Trends: live keywords, each one opens ranked clip cards</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/13%20Chat%20-%20a%20conversation%20with%20clip%20cards.png" alt="Chat: ask in plain words, get scored clips back" width="100%"/>
      <br/><sub>Chat: ask in plain words, get scored clips back</sub>
    </td>
  </tr>
</table>

<details>
<summary>More from this step (5 screens)</summary>
<br/>

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/2%20Home.png" alt="Home: four ways in, plus Recommended for you" width="100%"/>
      <br/><sub>Home: four ways in, plus Recommended for you</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/4%20Trends%20-%20pick%20range%20and%20topics.png" alt="Trends: pick a range and topics" width="100%"/>
      <br/><sub>Trends: pick a range and topics</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/5%20Trends%20-%20fetching.png" alt="Trends: fetching from Perplexity and Google Trends" width="100%"/>
      <br/><sub>Trends: fetching from Perplexity and Google Trends</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/14%20Chat%20-%20clip%20cards%20in%20the%20answer.png" alt="Chat: clip cards inside the answer" width="100%"/>
      <br/><sub>Chat: clip cards inside the answer</sub>
    </td>
  </tr>
  <tr>
    <td colspan="2" align="center" valign="top">
      <img src="Clipit%20Screenshots/15%20More%20like%20this%20-%20similar%20clips%20from%20other%20episodes.png" alt="More like this: similar clips from other episodes" width="50%"/>
      <br/><sub>More like this: similar clips from other episodes</sub>
    </td>
  </tr>
</table>

</details>

### 2. Pick and edit a clip

<table>
  <tr>
    <td width="26%" align="center" valign="top">
      <img src="Clipit%20Screenshots/8%20Clip%20card%20-%20score%2C%20reel%20script%2C%20actions.png" alt="Clip card: score, reel script, actions" width="100%"/>
      <br/><sub>Clip card: score, reel script, actions</sub>
    </td>
    <td width="74%" align="center" valign="top">
      <img src="Clipit%20Screenshots/9%20Edit%20lines%20-%20trim%20start%20and%20end%20on%20the%20script.png" alt="Edit lines: trim start and end on the script" width="100%"/>
      <br/><sub>Edit lines: trim start and end on the script</sub>
    </td>
  </tr>
</table>

<details>
<summary>More from this step (3 screens)</summary>
<br/>

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/10%20Edit%20lines%20-%20end%20dragged%20earlier%2C%20length%20updates.png" alt="Edit lines: end dragged earlier, length updates live" width="100%"/>
      <br/><sub>Edit lines: end dragged earlier, length updates live</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/11%20Merge%20-%20clips%20selected%20%28bottom%20bar%29.png" alt="Merge: clips selected from the bottom bar" width="100%"/>
      <br/><sub>Merge: clips selected from the bottom bar</sub>
    </td>
  </tr>
  <tr>
    <td colspan="2" align="center" valign="top">
      <img src="Clipit%20Screenshots/12%20Merge%20-%20arrange%20and%20trim%20parts.png" alt="Merge: arrange and trim each part" width="50%"/>
      <br/><sub>Merge: arrange and trim each part</sub>
    </td>
  </tr>
</table>

</details>

### 3. Make the short

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/22%20Make%20Short%20-%20review%20the%20reframed%20clip.png" alt="Review: fix caption words and B-roll before the final render" width="100%"/>
      <br/><sub>Review: fix caption words and B-roll before the final render</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/24%20Make%20Short%20-%20your%20short%20is%20ready.png" alt="Done: the finished 9:16 short" width="100%"/>
      <br/><sub>Done: the finished 9:16 short</sub>
    </td>
  </tr>
</table>

<details>
<summary>More from this step (8 screens)</summary>
<br/>

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/16%20Make%20Short%20step%201%20-%20Captions.png" alt="Step 1: captions, styled or simple" width="100%"/>
      <br/><sub>Step 1: captions, styled or simple</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/17%20Make%20Short%20step%202%20-%20Hook.png" alt="Step 2: hook line, placement and font" width="100%"/>
      <br/><sub>Step 2: hook line, placement and font</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/18%20Make%20Short%20step%203%20-%20B-roll.png" alt="Step 3: B-roll, none, auto or select" width="100%"/>
      <br/><sub>Step 3: B-roll, none, auto or select</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/19%20Make%20Short%20step%203%20-%20B-roll%20pick%20words%20from%20the%20script.png" alt="Step 3: pick B-roll words from the script" width="100%"/>
      <br/><sub>Step 3: pick B-roll words from the script</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/20%20Make%20Short%20step%204%20-%20Speaker%20and%20Brand.png" alt="Step 4: speaker name tag and branding" width="100%"/>
      <br/><sub>Step 4: speaker name tag and branding</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/21%20Make%20Short%20step%205%20-%20Review.png" alt="Step 5: review settings, then generate" width="100%"/>
      <br/><sub>Step 5: review settings, then generate</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/25%20Make%20Short%20-%20thumbnails%20box%20on%20the%20done%20screen.png" alt="Thumbnails box on the done screen" width="100%"/>
      <br/><sub>Thumbnails box on the done screen</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/26%20Thumbnails%20-%20one%20per%20person.png" alt="Thumbnails: one per person in the short" width="100%"/>
      <br/><sub>Thumbnails: one per person in the short</sub>
    </td>
  </tr>
</table>

</details>

### 4. Ship it

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/27%20Publish%20to%20YouTube.png" alt="Publish to YouTube as a Short" width="100%"/>
      <br/><sub>Publish to YouTube as a Short</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/28%20Post%20to%20Instagram.png" alt="Post to Instagram as a Reel" width="100%"/>
      <br/><sub>Post to Instagram as a Reel</sub>
    </td>
  </tr>
</table>

<details>
<summary>More from this step (6 screens)</summary>
<br/>

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/29%20Schedule%20a%20post.png" alt="Schedule a post" width="100%"/>
      <br/><sub>Schedule a post</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/30%20Calendar%20-%20month%20view.png" alt="Calendar: month view" width="100%"/>
      <br/><sub>Calendar: month view</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/32%20Saved%20-%20Clips.png" alt="Saved: clips from Trends" width="100%"/>
      <br/><sub>Saved: clips from Trends</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/33%20Saved%20-%20From%20Chat.png" alt="Saved: clips from Chat" width="100%"/>
      <br/><sub>Saved: clips from Chat</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/34%20Saved%20-%20Shorts.png" alt="Saved: finished shorts and their versions" width="100%"/>
      <br/><sub>Saved: finished shorts and their versions</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/35%20Saved%20-%20Channel%20Clips.png" alt="Saved: shorts made from Channel Clips" width="100%"/>
      <br/><sub>Saved: shorts made from Channel Clips</sub>
    </td>
  </tr>
</table>

</details>

### 5. Library, uploads and other channels

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/36%20Library%20-%20every%20episode.png" alt="Library: every episode" width="100%"/>
      <br/><sub>Library: every episode</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/43%20Channel%20Clips%20-%20keyword%20search%20across%20all%20channels.png" alt="Channel Clips: keyword search across all channels" width="100%"/>
      <br/><sub>Channel Clips: keyword search across all channels</sub>
    </td>
  </tr>
</table>

<details>
<summary>More from this step (8 screens)</summary>
<br/>

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/37%20Library%20-%20an%20episode%2C%20best%20moments.png" alt="Library: best moments of one episode" width="100%"/>
      <br/><sub>Library: best moments of one episode</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/38%20Library%20-%20trim%20your%20own%20clip.png" alt="Library: trim your own clip" width="100%"/>
      <br/><sub>Library: trim your own clip</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/39%20Library%20-%20import%20from%20YouTube.png" alt="Library: import from YouTube" width="100%"/>
      <br/><sub>Library: import from YouTube</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/40%20Upload%20a%20Video.png" alt="Upload a video, up to 5 GB" width="100%"/>
      <br/><sub>Upload a video, up to 5 GB</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/41%20Upload%20a%20Video%20-%20best%20moments%20of%20your%20file.png" alt="Upload: best moments of your file" width="100%"/>
      <br/><sub>Upload: best moments of your file</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/42%20Channel%20Clips%20-%20channels%20and%20videos.png" alt="Channel Clips: channels and videos" width="100%"/>
      <br/><sub>Channel Clips: channels and videos</sub>
    </td>
  </tr>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/44%20Channel%20Clips%20-%20a%20video%2C%20best%20moments%20or%20trim.png" alt="Channel Clips: best moments or trim on one video" width="100%"/>
      <br/><sub>Channel Clips: best moments or trim on one video</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/45%20Channel%20Clips%20-%20trim%20your%20own.png" alt="Channel Clips: trim your own" width="100%"/>
      <br/><sub>Channel Clips: trim your own</sub>
    </td>
  </tr>
</table>

</details>

### 6. Insights, Brand Kit and account

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/46%20Insights%20-%20YouTube.png" alt="Insights: YouTube" width="100%"/>
      <br/><sub>Insights: YouTube</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/48%20Insights%20-%20Instagram.png" alt="Insights: Instagram" width="100%"/>
      <br/><sub>Insights: Instagram</sub>
    </td>
  </tr>
</table>

<details>
<summary>More from this step (3 screens)</summary>
<br/>

<table>
  <tr>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/50%20Brand%20Kit.png" alt="Brand Kit: logo, fonts, colours, live preview" width="100%"/>
      <br/><sub>Brand Kit: logo, fonts, colours, live preview</sub>
    </td>
    <td width="50%" align="center" valign="top">
      <img src="Clipit%20Screenshots/54%20Notifications%20bell.png" alt="Notifications bell" width="100%"/>
      <br/><sub>Notifications bell</sub>
    </td>
  </tr>
  <tr>
    <td colspan="2" align="center" valign="top">
      <img src="Clipit%20Screenshots/56%20Change%20password.png" alt="Change password" width="50%"/>
      <br/><sub>Change password</sub>
    </td>
  </tr>
</table>

</details>

### Full-page captures


<details>
<summary>Long pages captured top to bottom (6 screens)</summary>
<br/>

<table>
  <tr>
    <td width="41%" align="center" valign="top">
      <img src="Clipit%20Screenshots/3%20Home%20-%20Recommended%20for%20you.png" alt="Home with Recommended for you" width="100%"/>
      <br/><sub>Home with Recommended for you</sub>
    </td>
    <td width="59%" align="center" valign="top">
      <img src="Clipit%20Screenshots/31%20Calendar%20-%20all%20posts%20%28full%20page%29.png" alt="Calendar: all posts" width="100%"/>
      <br/><sub>Calendar: all posts</sub>
    </td>
  </tr>
  <tr>
    <td width="45%" align="center" valign="top">
      <img src="Clipit%20Screenshots/7%20Trends%20-%20all%20clip%20cards%20%28full%20page%29.png" alt="Trends: all clip cards" width="100%"/>
      <br/><sub>Trends: all clip cards</sub>
    </td>
    <td width="55%" align="center" valign="top">
      <img src="Clipit%20Screenshots/47%20Insights%20-%20YouTube%20%28full%20dashboard%29.png" alt="Insights: full YouTube dashboard" width="100%"/>
      <br/><sub>Insights: full YouTube dashboard</sub>
    </td>
  </tr>
  <tr>
    <td width="47%" align="center" valign="top">
      <img src="Clipit%20Screenshots/49%20Insights%20-%20Instagram%20%28full%20dashboard%29.png" alt="Insights: full Instagram dashboard" width="100%"/>
      <br/><sub>Insights: full Instagram dashboard</sub>
    </td>
    <td width="53%" align="center" valign="top">
      <img src="Clipit%20Screenshots/51%20Brand%20Kit%20%28full%20page%29.png" alt="Brand Kit: full page" width="100%"/>
      <br/><sub>Brand Kit: full page</sub>
    </td>
  </tr>
</table>

</details>

## ⚠️ Limitations

| Limit | Detail |
|---|---|
| **Urdu word accuracy** | About 95%. Timing is exact, and spelling can be fixed on the review screen. |
| **YouTube quota** | The default API quota allows about 6 uploads a day. Volume needs a quota increase. |
| **App verification** | Google and Meta app verification is pending, so only listed test users can connect accounts. |
| **Thumbnails** | Need a visible face in the short. |
| **Uploads** | An upload in progress is lost if the server restarts mid-upload. Finished uploads are safe. |

## 🗺️ Roadmap

- [ ] Approval mode for scheduled posts, with email notices
- [ ] Split-screen reels
- [ ] TikTok as a trend source
- [ ] Viral-taste tuning from reference reels
- [ ] HNSW index and separate worker machines as libraries grow

## 📬 Contact

**Adeel Iqbal**

- 📧 Email: [adeelmemon096@yahoo.com](mailto:adeelmemon096@yahoo.com)
- 💼 LinkedIn: [linkedin.com/in/adeeliqbalmemon](https://linkedin.com/in/adeeliqbalmemon)
- 🐙 GitHub: [@adeel-iqbal](https://github.com/adeel-iqbal)
