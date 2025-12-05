## Downloads

- Backend archive: see the latest **Release** (catvsdoglive_backend.zip)
- Frontend archive: catvsdoglive_frontend.zip (in repo root)
- Model training archive: catvsdoglive_modeltraining.zip (in repo root)

# catvsdog.live – Full Open-Source Archive

catvsdog.live is a real-time crypto token classifier and sentiment tracker that was live from **July 2025 – December 2025**.  
The live service is going to be discontinued December 2025; this repository contains the complete **backend**, **frontend**, and **model training** source code, including the production model interface and SQLite database schemas used in production.

The system was originally deployed on Ubuntu with:

- Node.js 18.x (PM2 supervised)  
- Python 3.12+ (separate virtual environment)  

The codebase is published for learning, experimentation, research, and reuse.

All code was provided by ChatGPT and Claude. Developer had no past experience on coding.

---

## Table of Contents

- [Features](#features)  
  - [Backend](#backend)  
  - [Frontend](#frontend)  
- [Architecture Overview](#architecture-overview)  
- [Repository Structure](#repository-structure)  
- [Getting Started](#getting-started)  
  - [Prerequisites](#prerequisites)  
  - [Backend Setup](#backend-setup)  
  - [Worker Processes](#worker-processes)  
  - [Frontend Setup](#frontend-setup)  
- [Databases & Models](#databases--models)  
- [Configuration](#configuration)  
- [Development Notes](#development-notes)  
- [License](#license)  
- [Credits](#credits)  

---

## Features

### Backend

Tech stack: Node.js 18.x, Python 3.12+

Core capabilities (implemented by the backend Node.js and Python modules):

- Real-time token ingestion
  - Connects to the PumpPortal WebSocket API.
  - Normalizes and de-duplicates token creation events.
  - Supports per-pool metadata through an internal token registry.

- High-throughput processing pipeline
  - index.js, api.js, tokenProcessingService.js, and tokenProcessor.js orchestrate:
    - token metadata handling (direct-image URLs vs JSON metadata)
    - resilient HTTP/IPFS fetching with rate limiting
    - dispatch to worker threads for database writes and image handling
  - fileHandler.js uses axios, sharp, jsdom, svgo, and svg2img to:
    - download images over HTTP and via IPFS gateways or a local node
    - validate content-type, dimensions, and file size
    - sanitize SVGs (no scripts, foreignObject, or remote references) and convert them to PNG
    - write files safely under IMAGE_DIR using fileDurability.js (fsync + atomic rename)

- Image classification via Python + FastAI
  - testmodel.py is a standalone WebSocket client:
    - connects to the backend WebSocket server using the "testmodel-protocol" subprotocol
    - loads a FastAI ensemble of ResNet models from the models directory:
      - resnet50_model.pkl
      - resnet101_model.pkl
      - resnet152_model.pkl
    - performs inference on downloaded images and combines ensemble outputs
    - applies text-aware adjustments using token name, ticker, and description
    - returns JSON results to the backend:
      - token address
      - classification (Cat / Dog / Other)
      - confidence
      - final image path

- Sentiment analysis (VADER + custom lexicon)
  - sentimentmodel.py is a second WebSocket client:
    - connects to the backend WebSocket server using the "vader-protocol" subprotocol
    - uses vaderSentiment.SentimentIntensityAnalyzer
    - loads custom_lexicon.json, which contains:
      - custom sentiment scores for crypto slang and emojis
      - a slang_map used to normalize abbreviations
    - receives name, ticker, and description for each token
    - produces neg, neu, pos, and compound scores
    - writes normalized text into texts.db (SQLite) using an internal queue with disk-backed persistence

- Sentiment aggregation
  - aggregator.js:
    - aggregates entries from the sentiment_minute_partials table in sentiment.db
    - runs every 60 seconds to compute unweighted average compound scores over:
      - last 1 hour
      - last 4 hours
      - last 12 hours
      - last 24 hours
    - reads a lifetime average from sentiment_lifetime_stats
    - uses operationsTracker.js to track aggregation and pruning operations
    - triggers pruneOldSentimentData() periodically to keep sentiment_minute_partials bounded

- Trending words & global counters
  - tokens.db stores:
    - tokens (metadata, classification, status)
    - counters (catCount, dogCount, otherCount)
    - trending_words (top 9 words in token names)
  - websocket.js:
    - maintains in-memory counters and a sliding list of the most recent tokens
    - accumulates word frequencies from tokenNameAndTicker
    - periodically syncs the top 9 words into trending_words and broadcasts them

- Durable SQLite storage
  - database.js configures:
    - DB_PATH for tokens.db
    - SENTIMENT_DB_PATH for sentiment.db
    - IMAGE_DIR for image storage (token_images by default)
    - TEXTS_DB_PATH for texts.db
  - ensures IMAGE_DIR exists on startup
  - sets journal modes and pragmas for SQLite databases

- Resiliency, retries, and maintenance
  - retryFailedTokens.js:
    - finds tokens stuck in pending or failed metadata/image states
    - retries them once, with age and host-based filters
  - operationsTracker.js:
    - assigns operation IDs to long-running operations (DB, file, WebSocket)
    - tracks TTLs and detects potentially stuck operations
    - supports safe shutdown via sweepPendingOperations()
  - maintenance.js and maintenance-worker.js:
    - support a maintenance mode where new tokens are cached
    - run VACUUM and ANALYZE on tokens.db and sentiment.db
    - flush cached tokens after maintenance

- WebSocket hub
  - websocket.js spins up a WebSocket server on TESTMODEL_WS_PORT for multiple subprotocols:
    - frontend-protocol – Node.js frontend proxy
    - testmodel-protocol – Python classifier
    - vader-protocol – Python sentiment worker
    - catvsdog-ai-protocol – optional separate AI stats backend (external, not included)
  - handles:
    - state snapshots (full-state)
    - counters updates (counters-update)
    - classification results (image-prediction)
    - token lists (token-list-update)
    - cat/dog image grids (cat-dog-images-update)
    - trending words (trending-words-update)
    - per-token sentiment (vader-sentiment)
    - aggregated sentiment history (sentiment-history)

The backend is responsible for all ingestion, database management, image I/O, and real-time signalling.

---

### Frontend

Tech stack: Node.js 18.x, Express, ws, static HTML/CSS/JS

Frontend behaviour is implemented by frontend.js, Python analytics scripts, and public assets:

- HTTP + WebSocket proxy
  - frontend.js:
    - Express HTTP server serving static assets from the public directory
    - REST endpoints that expose cached backend state:
      - /get-counters
      - /get-trending-words
      - /get-token-info
      - /get-cat-dog-images
      - /get-current-processing-image
      - /get-top-words
      - /get-generated-names
      - /get-rolling-text
    - WebSocket client to the backend using frontend-protocol and an API key
    - WebSocket server for browser clients
    - maintains in-memory state mirrors of:
      - counters
      - recent tokens
      - cat/dog image grids
      - current "processing" token
      - trending words
      - sentiment history
      - rolling analyzed texts
      - top-word statistics
      - AI-assisted generated names
    - forwards backend events to all connected browsers with reconnection and rate limiting

- Real-time token stream UI
  - public/index.html with public/js/counter.js:
    - shows current token classification (image, class, confidence, address)
    - displays cat and dog counters with 7-segment-style digits
    - renders cat and dog image grids (1, 4, or 9 images)
    - shows the top 9 trending words
    - flashes a highlight around the current image on each new prediction
  - public/stream.html with public/js/stream.js:
    - displays a scrolling list of recent tokens, including:
      - image
      - name and ticker
      - description
      - classification

- Sentiment dashboard
  - public/sentiment.html with public/js/sentiment.js and public/js/sentiment-enhancements.js:
    - shows the current compound sentiment on a gauge
    - displays average sentiment over 1h, 4h, 12h, 24h, and lifetime
    - shows differences relative to lifetime with direction indicators
    - lists top words computed by Python analytics
    - provides a rolling text feed of recently analyzed texts

- Token name generator
  - public/generator.html with public/js/generator-logic.js, public/js/generator-ui.js, and public/js/generator-enhancements.js:
    - local generation mode:
      - reads public/data.json
      - composes meme-token-style names and tickers (AI, seasonal, finance, animals, Inus, protocols, etc.)
      - includes special functions that generate names from curated patterns
    - AI assist mode:
      - fetches names from the backend via /get-generated-names
      - uses names derived from actual analyzed texts (via word_generator.py)
      - maintains a local history list with visual highlighting for special substrings (e.g., "elon")

- Trending word TTS
  - generate_tts.py:
    - reads the latest trending words from a JSON file written by frontend.js
    - produces an audio file (trending-words-latest.wav) using espeak and optionally mbrola + sox
    - writes the result into public/audio
  - public/js/globalAudio.js and public/js/toggleOnOff.js:
    - control global playback of the trending-words audio clip

- Token game
  - public/game-window.html with public/game.json and public/js/script.js:
    - implements a small interactive market-cap "game"
    - uses Chart.js (loaded via script tag) to render a line chart of price over turns

- Content and styling
  - public/content/*.txt store explanatory text for the main page
    - loaded via AJAX, sanitized with DOMPurify (inside counter.js)
  - CSS files in public/css:
    - styles.css
    - windicss.css
    - generator-enhancements.css
    - sentiment-enhancements.css
  - favicon and multiple hero images in public/images

- Security and robustness
  - frontend.js configures:
    - helmet with a strict Content-Security-Policy
    - CORS with an allowed-origin list
    - express-rate-limit on API routes and /login
    - 404 and 503 fallback pages
  - WebSocket "full state" requests are rate-limited per client IP to protect the backend.

The frontend acts as a thin, hardened proxy between the backend and browser clients, and hosts the static UI.

---

## Architecture Overview

End-to-end flow:

1. Ingestion  
   - The backend Node.js code (api.js and index.js) opens a WebSocket connection to PumpPortal.  
   - On each token creation event, tokenProcessor.js and tokenProcessingService.js:
     - normalize the event
     - decide whether the URI is direct image vs metadata JSON
     - create or update a row in tokens.db via dbHandler.js.

2. Enrichment  
   - For tokens with metadata URLs, ipfsDownloader.js and urlUtils.js:
     - download metadata JSON from IPFS gateways or a local node
     - resolve image URLs and descriptions
   - fileHandler.js:
     - downloads the image, validates it, and stores it in IMAGE_DIR
     - sends success or failure messages back to the main thread
   - On a successful image:
     - api.js enqueues a classification request to testmodel.py over WebSocket
     - api.js enqueues a sentiment request to sentimentmodel.py over WebSocket

3. ML and sentiment workers  
   - testmodel.py:
     - loads FastAI ResNet models from the models directory
     - runs inference and returns classification + confidence
   - sentimentmodel.py:
     - loads VADER + custom_lexicon.json
     - constructs and normalizes the full text for the token
     - writes text entries into texts.db (analyzed_texts table)
     - returns sentiment scores

4. Storage  
   - dbHandler.js writes:
     - updated token rows to tokens.db
     - counters (catCount, dogCount, otherCount)
   - aggregator.js and database.js manage:
     - minute-level sentiment rows in sentiment_minute_partials
     - lifetime stats in sentiment_lifetime_stats
   - sentimentmodel.py maintains texts.db with a queue and backup files under database/queue.

5. Delivery  
   - websocket.js:
     - keeps in-memory snapshots of tokens, counters, trending words, and sentiment history
     - broadcasts updates to frontend.js (Node frontend) and other possible consumers
   - frontend.js:
     - mirrors the state in memory
     - serves initial JSON snapshots over REST
     - pushes real-time updates to browser WebSocket clients

6. Presentation  
   - Browser pages (index.html, stream.html, sentiment.html, generator.html) connect to the frontend WebSocket server:
     - request a full-state snapshot on connect
     - subscribe to incremental updates
   - JavaScript in public/js renders counters, images, charts, sentiment indicators, and generator output.

This design keeps the backend core focused on ingestion, persistence, and messaging, while the Python workers and frontend are thin, replaceable components.

---

## Repository Structure

At the GitHub repository root you currently have:

- `README.md` – full documentation and architecture overview  
- `LICENSE` – MIT license  
- `catvsdoglive_frontend.zip` – frontend snapshot (Node.js frontend + static assets)  
- `catvsdoglive_modeltraining.zip` – model training script and class weights  

The **backend** snapshot is stored as a **GitHub Release asset**, not as a tracked file in the repository root:

- `catvsdoglive_backend.zip` – attached to the latest Release under the **Releases** tab on GitHub.

All three archives together contain the full source code and artifacts.

### Backend archive (`catvsdoglive_backend.zip`)

Location:

- Available as an asset in the latest GitHub **Release**.

Contents (inside the archive):

- `backend/`
  - `index.js`
  - `api.js`
  - `websocket.js`
  - `database.js`
  - `aggregator.js`
  - `retryFailedTokens.js`
  - `maintenance.js`
  - `maintenance-worker.js`
  - `operationsTracker.js`
  - `queueManager.js`
  - `fileDurability.js`
  - `workers.js`
  - `dbHandler.js`
  - `fileHandler.js`
  - `tokenProcessingService.js`
  - `tokenProcessor.js`
  - `ipfsDownloader.js`
  - `urlUtils.js`
  - `browserHeaders.js`
  - `tokenRegistry.js`
  - `testmodel.py`
  - `sentimentmodel.py`
  - `custom_lexicon.json`
  - `.env.example_backend`
  - `package.json`
  - `package-lock.json`
  - `database/`
    - `tokens.db`
    - `sentiment.db`
    - `texts.db`
    - `queue/`
      - backup and emergency queue files (if present)
  - `models/`
    - `resnet50_model.pkl`
    - `resnet101_model.pkl`
    - `resnet152_model.pkl`
  - `token_images/` (directory expected at runtime as `IMAGE_DIR`; image files themselves are not included to keep the archive size reasonable)

Notes:

- The `token_images` directory is created and filled at runtime; it is included as a directory but not populated with image files.
- SQLite WAL/SHM files are intentionally not included; SQLite recreates them as needed.
  
## Related project – GENESIS_MACHINE

[GENESIS_MACHINE](https://github.com/catvsdoglive/genesis-machine) is an autonomous meme token
generator (Mistral + Stable Diffusion) that uses live trending words from this project via the
`/get-trending-words` endpoint.

It can run standalone or alongside catvsdog-live to generate synthetic tokens and images based on
the classifier’s live trending data.

### Frontend archive (`catvsdoglive_frontend.zip`)

Location:

- Stored in the repository root.

Contents (inside the archive):

- `frontend/`
  - `frontend.js`
  - `word_counter.py`
  - `word_generator.py`
  - `generate_tts.py`
  - `.env.example_frontend`
  - `package.json`
  - `package-lock.json`
  - `public/`
    - `index.html`
    - `stream.html`
    - `sentiment.html`
    - `generator.html`
    - `disclaimer.html`
    - `404.html`
    - `503.html`
    - `503_backup.html`
    - `comingsoon.html`
    - `robots/robots.txt`
    - `css/`
      - `styles.css`
      - `windicss.css`
      - `generator-enhancements.css`
      - `sentiment-enhancements.css`
    - `js/`
      - `script.js`
      - `counter.js`
      - `stream.js`
      - `sentiment.js`
      - `sentiment-enhancements.js`
      - `generator-logic.js`
      - `generator-ui.js`
      - `generator-enhancements.js`
      - `globalAudio.js`
      - `toggleOnOff.js`
      - `popup.js`
      - `button.js`
    - `libs/`
      - `pixi.mjs`
    - `images/`
      - `logo.png`
      - `catvsdog.png`
      - `catvsdog-ai.png`
      - `catvsdogai.png`
      - `404.png`
      - `503.png`
      - `default-placeholder.png`
      - hero and meme images (`eloncatdog.png`, `elonmeme1.png`, `elonmeme2.png`, `elonmeme3.png`, `elonpump.png`, etc.)
    - `content/`
      - `about.txt`
      - `section1.txt`
      - `section2.txt`
      - `rectangle.txt`
      - `marquee.txt`
    - `data.json`
    - `game.json`
    - `game-window.html`
    - `audio/`
      - directory used for generated audio files; `trending-words-latest.wav`, `tts_debug.log`, and test files are generated at runtime and are not included in the archive.

### Model training archive (`catvsdoglive_modeltraining.zip`)

Location:

- Stored in the repository root.

Contents:

- `train_all.py`  
- `classweights.txt`  

`train_all.py`:

- Sets up a FastAI training loop for the Cat / Dog / Other classifier.
- Expects `classweights.txt` to provide per-class weights (for example, based on dataset class frequencies).
- Is intended for offline use with your own dataset mounted in the paths referenced in the script.

This training archive is not required to run the live system but allows you to retrain or fine-tune models using the same pipeline.

## Getting Started

### Prerequisites

System:

- Recent Linux or macOS
- SQLite tools (optional, for manual inspection)

Node.js:

- Node.js 18.x
- npm (or another package manager of your choice)

Python:

- Python 3.12+ (3.10+ should also work)
- A dedicated virtual environment is strongly recommended.

Python packages (installed via pip):

- torch
- fastai
- pillow
- pillow-heif
- pillow-avif-plugin
- vaderSentiment
- emoji
- websockets
- python-dotenv

Optional system binaries for TTS:

- espeak
- mbrola
- sox

If these binaries are not available, generate_tts.py will either fall back to a simpler mode or you can disable TTS.

---

### Backend Setup

1. Unpack the backend archive:

    unzip catvsdoglive_backend.zip

   This creates a directory named backend.

2. Change into the backend directory and install Node dependencies:

    cd backend  
    npm install

3. Create a backend .env file:

   If .env.example_backend is present:

    cp .env.example_backend .env

   Then edit .env and set:

   - DB_PATH, SENTIMENT_DB_PATH, TEXTS_DB_PATH  
   - IMAGE_DIR  
   - TESTMODEL_WS_PORT  
   - PUMPPORTAL_WS_URL  
   - VADER_WS_URL  
   - TESTMODEL_MODEL_DIR  
   - TESTMODEL_DEVICE (cpu or cuda)  
   - API_KEY_PROD  

4. Verify models and databases:

   - The models directory contains:
     - resnet50_model.pkl
     - resnet101_model.pkl
     - resnet152_model.pkl
   - The database directory contains:
     - tokens.db
     - sentiment.db
     - texts.db

   If you prefer to start from empty databases, you can delete the .db files before first run; the Node backend and sentiment worker will recreate the schema.

5. Run the Node backend:

   Development:

    node index.js

   Production with PM2:

    pm2 start index.js --name catvsdog-backend

On startup, index.js ensures IMAGE_DIR exists (creating the token_images directory if needed), starts workers from workers.js, connects to PumpPortal if configured, and starts the backend WebSocket server.

---

### Worker Processes

The image classifier and sentiment analyzer are separate Python processes.

From the backend directory:

1. Activate your Python virtual environment (if any), then start the classifier:

    python testmodel.py

   testmodel.py reads configuration from the same .env file as the backend and connects to the backend WebSocket server using the testmodel-protocol subprotocol.

2. Start the sentiment worker:

    python sentimentmodel.py

   sentimentmodel.py also reads the .env file, connects using the vader-protocol subprotocol, and manages texts.db and the sentiment queue.

Both workers must be running for the full system (classification + sentiment + analytics) to function.

---

### Frontend Setup

1. Unpack the frontend archive:

    unzip catvsdoglive_frontend.zip

   This creates a directory named frontend.

2. Change into the frontend directory and install Node dependencies:

    cd frontend  
    npm install

3. Create a frontend .env file:

   If .env.example_frontend is present:

    cp .env.example_frontend .env

   Then edit .env and set:

   - FRONTEND_PORT  
   - FRONTEND_WS_PORT  
   - BACKEND_WS_URL (e.g. ws://127.0.0.1:9000)  
   - API_KEY_PROD (must match the backend)  
   - PUBLIC_DIR (usually ./public)  
   - IMAGE_DIR (path where backend stores images)  
   - TEXTS_DB_PATH (points to backend/database/texts.db)  
   - CORS_ORIGIN  
   - API_RATE_LIMIT  

4. Run the frontend server:

   Development:

    node frontend.js

   Production with PM2:

    pm2 start frontend.js --name catvsdog-frontend

The frontend server will:

- serve static files from PUBLIC_DIR  
- expose REST endpoints under /  
- open a WebSocket connection to the backend  
- host a WebSocket server for browser clients  

5. Open the UI in your browser:

- Main counter: http://localhost:FRONTEND_PORT/  
- Stream: http://localhost:FRONTEND_PORT/stream.html  
- Sentiment: http://localhost:FRONTEND_PORT/sentiment.html  
- Generator: http://localhost:FRONTEND_PORT/generator.html  

Replace FRONTEND_PORT with the actual port configured in your .env.

---

## Databases & Models

### Runtime databases

Three SQLite databases are used:

1. tokens.db  
   - Managed by the Node backend (database.js and dbHandler.js).  
   - Tables:
     - tokens – token metadata, classification, and state
     - counters – global counts (catCount, dogCount, otherCount)
     - trending_words – top 9 words with counts
   - Automatically pruned to keep the number of tokens bounded.

2. sentiment.db  
   - Managed by database.js and aggregator.js.  
   - Tables:
     - sentiment_minute_partials – per-minute aggregates of neg, neu, pos, compound, token_count
     - sentiment_lifetime_stats – lifetime_avg, total_count, updated_at
   - Old minute rows are regularly pruned.

3. texts.db  
   - Managed primarily by sentimentmodel.py.  
   - Tables:
     - analyzed_texts – stores normalized text strings and timestamps
   - Uses WAL mode and backup files under database/queue to avoid data loss.

You can inspect these directly:

    sqlite3 database/tokens.db  
    sqlite3 database/sentiment.db  
    sqlite3 database/texts.db

### Runtime models

The classifier uses an ensemble of FastAI models located in the models directory:

- resnet50_model.pkl  
- resnet101_model.pkl  
- resnet152_model.pkl  

These models are loaded by testmodel.py and are sufficient for inference.

### Training pipeline (catvsdoglive_modeltraining.zip)

The model training support files live in a separate archive:

- catvsdoglive_modeltraining.zip
  - train_all.py
  - classweights.txt

train_all.py:

- sets up a FastAI training loop for the cat/dog/other classifier
- expects classweights.txt to provide per-class weights (for example based on dataset class frequencies)
- is intended for offline use with your own dataset mounted in the paths referenced in the script

This training archive is not required for running the live system but allows you to retrain or fine-tune models following the same pipeline.

---

## Configuration

Both backend and frontend load environment variables from .env files using dotenv.

### Backend .env

Common variables:

- DB_PATH – path to tokens.db (relative to backend)  
- SENTIMENT_DB_PATH – path to sentiment.db  
- TEXTS_DB_PATH – path to texts.db  
- IMAGE_DIR – directory for token images (e.g. token_images)  
- TESTMODEL_WS_PORT – port for the backend WebSocket server  
- PUMPPORTAL_WS_URL – PumpPortal WebSocket endpoint  
- VADER_WS_URL – WebSocket URL that sentimentmodel.py connects to  
- TESTMODEL_MODEL_DIR – directory containing ResNet model files  
- TESTMODEL_DEVICE – device for inference (cpu or cuda)  
- API_KEY_PROD – API key for securing frontend-protocol connections  

If DB_PATH, SENTIMENT_DB_PATH, or IMAGE_DIR are not set, database.js defaults them to:

- database/tokens.db  
- database/sentiment.db  
- token_images  

### Frontend .env

Common variables:

- FRONTEND_PORT – HTTP port for the frontend Express server  
- FRONTEND_WS_PORT – WebSocket port for browser clients  
- BACKEND_WS_URL – WebSocket URL of the backend service  
- API_KEY_PROD – API key used when connecting to the backend WebSocket  
- PUBLIC_DIR – path to static assets (usually ./public)  
- IMAGE_DIR – path where token images are served from  
- TEXTS_DB_PATH – path to texts.db for analytics scripts  
- CORS_ORIGIN – allowed origins for CORS  
- API_RATE_LIMIT – requests per window for rate-limited routes  

Ensure API_KEY_PROD matches between backend and frontend so that frontend-protocol connections are accepted.

---

## Development Notes

### Security

Before exposing the system to the public internet:

- Use a reverse proxy (for example Nginx or Caddy) in front of backend and frontend:
  - enforce HTTPS and WSS
  - configure rate limiting and request size limits
- Keep .env files out of version control and restrict access to them.
- Consider additional authentication/authorization on REST and WebSocket endpoints if you expose internal data.
- Audit IPFS and HTTP fetching behaviour in fileHandler.js, ipfsDownloader.js, and urlUtils.js for your own threat model.
- Review CSP and CORS configuration in frontend.js and adjust to your deployment domains.

### Performance

- QueueManager in the backend dynamically manages concurrency and backpressure for image downloads.
- retryFailedTokens.js prevents repeated hammering of problematic endpoints.
- sentiment.db is pruned over time to keep the number of minute partials bounded.
- tokens.db is pruned by dbHandler.js to cap the total number of stored tokens.
- For higher throughput or stricter durability guarantees, you may port the SQLite access layer in database.js to PostgreSQL or another RDBMS.

### Extensibility

Some straightforward extension points:

- Add new image models:
  - extend testmodel.py to load additional .pkl models
  - adjust ensemble logic to combine more outputs
- Add more text analytics:
  - implement a new Python script that connects via its own WebSocket subprotocol
  - mirror the pattern used by sentimentmodel.py
- Build alternative frontends:
  - reuse the WebSocket protocol and REST endpoints
  - connect a React/Vue/Svelte SPA instead of (or in addition to) the static pages
- Swap or extend data sources:
  - modify api.js to connect to different token feeds, keeping the internal token representation consistent.

### Limitations

- The live catvsdog.live service is no longer running; any connection to PumpPortal depends on their current API and terms.
- IPFS availability depends on external gateways or a local node.
- SQLite was sufficient at original traffic levels; very heavy loads might require a different database backend.
- Some training details (dataset paths, augmentations) live only in train_all.py and may require adaptation to your own dataset.

---

## License

MIT License.

---

## Credits

Built and operated by a single maintainer, with help from:

- ChatGPT and Claude during design and implementation  

The repository is shared as-is, without warranty, so others can learn from, dissect, and repurpose the code and ideas.
