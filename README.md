# Go Proof of Work Blockchain

A small learning project: an in-memory proof-of-work blockchain in Go, exposed over a two-route HTTP API.

## What it does

- Keeps a chain of blocks in memory, starting from a genesis block created at startup.
- Each block has an index, timestamp, integer `Data`, hash, previous hash, difficulty and nonce.
- `POST /` mines a new block for the `Data` you send, using a simple proof-of-work loop.
- `GET /` returns the whole chain as JSON.
- Checks each new block against the previous one (index, previous hash, recomputed hash) before appending it.

## How it works

- `calculateHash` in `main.go` builds a string from the block's index, timestamp, data, previous hash and nonce, and hashes it with SHA-256 (`crypto/sha256`).
- `generateBlock` increments a counter, uses its hex form as the nonce, and recomputes the hash until `isHashValid` passes.
- `isHashValid` checks that the hash starts with the required number of zeros. `getDifficulty` reads it from the `DIFFICULTY` environment variable and falls back to `1`.
- The mining loop sleeps one second after each failed attempt, so mining is slow on purpose and easy to watch in the logs ("do more work!" / "work done!").
- `isBlockValid` compares the new block's index and `PrevHash` with the last block and recomputes its hash.
- Routing uses `gorilla/mux` (`routes.go`). The port comes from the `PORT` environment variable, loaded from `.env` with `joho/godotenv`.
- A `sync.Mutex` guards block generation.

## Run it

Requires Go 1.22.7 or later (from `go.mod`).

```bash
git clone https://github.com/pri-3x/Go-POW
cd Go-POW
go mod tidy
```

Set the port in `.env` (a `.env` file is included in the repo):

```text
PORT=8080
DIFFICULTY=1
```

Start the server:

```bash
go run .
```

Read the chain:

```bash
curl http://localhost:8080/
```

Add a block. The `Data` field is an integer:

```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{"Data": 75}'
```

JSON field matching in Go is case-insensitive, so `{"data": 75}` also works.

## Project layout

```text
main.go       Block type, genesis block, hashing, mining and validation, HTTP server
handlers.go   GET / and POST / handlers
routes.go     gorilla/mux router
.env          PORT setting
```

## Notes / limitations

- Single process, in-memory only. The chain is lost on restart. There is no networking between nodes and no consensus.
- Difficulty is set once from `DIFFICULTY` (default `1`). It does not adjust to block times.
- Mining sleeps one second per failed nonce, and the server's `WriteTimeout` is 10 seconds. A slow mine can outlast the timeout, in which case the client may not receive the response even though the block was added.
- The mutex covers block generation only. Reads in `GET /` and the append after validation are not locked.
- If `PORT` is not set, the server listens on a random port.
- No tests.
