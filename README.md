# crunchtime-tally

A vote-tallying server for live performance. Listens for OSC messages from [crunchtime-reed-switch](../crunchtime-reed-switch) sensors, tracks 12 voters each with 3 choices (A/B/C), and fires QLab cues in real-time as votes change.

## How it works

**Inputs — OSC (UDP, configurable port)**

Each physical reed switch sensor maps to a voter seat and a choice. There are 36 sensors total (12 seats × 3 choices each). Sensors are numbered 1–36:

- Sensors 1, 2, 3 → Seat 1 (choices A, B, C)
- Sensors 4, 5, 6 → Seat 2 (choices A, B, C)
- … and so on up to Seat 12

| Message | Meaning |
|---|---|
| `/cue/{N}on/start` | Sensor N activated — voter in seat ⌊(N-1)/3⌋ chose A/B/C |
| `/cue/{N}off/start` | Sensor N deactivated — seat reverts to no vote |
| `/reset` | Clear all votes |

The server also sends `/ack` back to the originating sensor address and relays the raw message to QLab on port 53000.

**State**

A 12-element array tracks each seat's current vote (`N`=none, `A`, `B`, `C`). A seat's vote is overwritten when a new sensor fires, and cleared when any sensor for that seat deactivates.

**Outputs — OSC to QLab (localhost:53000)**

After every vote change the server fires four cues:

| Cue | Example | Meaning |
|---|---|---|
| `/cue/aTally{n}/start` | `/cue/aTally3/start` | Current count for choice A |
| `/cue/bTally{n}/start` | `/cue/bTally5/start` | Current count for choice B |
| `/cue/cTally{n}/start` | `/cue/cTally2/start` | Current count for choice C |
| `/cue/total{n}/start` | `/cue/total10/start` | Total votes cast (A+B+C) |

Plus one winner/tie cue:

| Cue | Condition |
|---|---|
| `/cue/aWin/start` | A has more votes than B and C |
| `/cue/bWin/start` | B has more votes than A and C |
| `/cue/cWin/start` | C has more votes than A and B |
| `/cue/abTie/start` | A and B tied, C different |
| `/cue/acTie/start` | A and C tied, B different |
| `/cue/cbTie/start` | C and B tied, A different |
| `/cue/abcTie/start` | All three equal |

## Configuration

Address is set via `.env` file or environment variable, with a `--addr` flag override:

```
TALLY_ADDR=10.0.0.73:8765
```

```sh
# Use .env
go run .

# Or pass directly
go run . --addr 10.0.0.73:8765
```

## Build & run

```sh
go run .          # run directly
go build -o tally # build binary
./tally
```

Requires Go 1.20+. Dependencies: `go-osc` (OSC protocol), `godotenv` (`.env` loading).

## Sensor numbering reference

```
Seat:    1        2        3        4   ...  12
         ↑        ↑        ↑        ↑        ↑
Sensors: 1,2,3   4,5,6   7,8,9  10,11,12  34,35,36
Choice:  A,B,C   A,B,C   A,B,C   A,B,C    A,B,C
```

Sensor → seat: `seat = floor((sensor - 1) / 3)`  
Sensor → choice: `sensor % 3` → `1=A, 2=B, 0=C`

## License

MIT — see source header for full text.
