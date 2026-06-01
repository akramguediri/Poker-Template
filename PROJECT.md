# Texas Hold'em Poker — Full Project Documentation

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Repository Structure](#2-repository-structure)
3. [The Original CLI Game — `Poker_game/`](#3-the-original-cli-game--poker_game)
   - [cards.py](#cardspy)
   - [evaluator.py](#evaluatorpy)
   - [player.py](#playerpy)
   - [table.py](#tablepy)
   - [ui.py](#uipy)
   - [game.py](#gamepy)
   - [main.py (CLI)](#mainpy-cli)
4. [The Web Application — `backend/`](#4-the-web-application--backend)
   - [Why a rewrite was needed](#why-a-rewrite-was-needed)
   - [game_session.py — The State Machine](#game_sessionpy--the-state-machine)
   - [main.py — The FastAPI App](#mainpy--the-fastapi-app)
   - [Frontend — static/](#frontend--static)
5. [Docker Setup](#5-docker-setup)
6. [How to Run](#6-how-to-run)
7. [API Reference](#7-api-reference)
8. [Data Flow — A Full Hand](#8-data-flow--a-full-hand)
9. [Known Limitations](#9-known-limitations)

---

## 1. Project Overview

This project is a Texas Hold'em poker game that started as a Python command-line application and was extended into a full web application. The web version consists of:

- A **FastAPI backend** that manages game state and exposes a REST API
- A **vanilla JS/HTML/CSS frontend** that renders the poker table in the browser
- A **Docker setup** to package and run everything with a single command

The game supports one human player ("You") against two AI opponents ("Ada Bot" and "Grace Bot"). Players start with 1000 chips each, with blinds of 5 (small) and 10 (big).

---

## 2. Repository Structure

```
poker/
├── docker-compose.yml          # Runs the whole application
│
├── Poker_game/                 # Original CLI game (still runnable)
│   ├── main.py                 # CLI entry point
│   ├── game.py                 # Main game loop (uses blocking input())
│   ├── cards.py                # Card, Rank, Suit, Deck
│   ├── evaluator.py            # Hand ranking logic
│   ├── player.py               # Player data model
│   ├── table.py                # Shared table state
│   ├── ui.py                   # Console I/O helpers
│   └── workstations/           # Student exercise worksheets
│
└── backend/                    # Web application (FastAPI + frontend)
    ├── Dockerfile              # Container image definition
    ├── .dockerignore           # Excludes __pycache__ etc.
    ├── requirements.txt        # Python dependencies
    ├── main.py                 # FastAPI app — API routes + static mount
    ├── game_session.py         # Game state machine (replaces game.py)
    ├── cards.py                # Copied from Poker_game/ (canonical copy)
    ├── evaluator.py            # Copied from Poker_game/
    ├── player.py               # Copied from Poker_game/
    ├── table.py                # Copied from Poker_game/
    └── static/                 # Frontend (served directly by FastAPI)
        ├── index.html          # Single-page app shell
        ├── style.css           # Poker table styling
        └── app.js              # Game UI logic, fetch calls
```

The game engine files (`cards.py`, `evaluator.py`, `player.py`, `table.py`) are copied into `backend/` to keep it self-contained. The originals in `Poker_game/` remain the CLI versions.

---

## 3. The Original CLI Game — `Poker_game/`

### `cards.py`

Defines the fundamental card types used throughout the game.

**`Suit` (IntEnum):** Four suits — `CLUBS=0`, `DIAMONDS=1`, `HEARTS=2`, `SPADES=3`. Each has a `.symbol` property returning a single letter: `C`, `D`, `H`, `S`.

**`Rank` (IntEnum):** Values 2–14 (Ace=14). Each has a `.label` property: digits `2`–`9`, then `T`, `J`, `Q`, `K`, `A`.

**`Card` (frozen dataclass):** A `(rank, suit)` pair. `str(card)` returns a two-character code like `"AH"` (Ace of Hearts) or `"TD"` (Ten of Diamonds). Frozen means it is immutable and hashable.

**`Deck`:** Holds all 52 cards in a list, shuffled on construction. `deal(n)` pops `n` cards from the end of the list. Raises `ValueError` if the deck runs out.

```python
deck = Deck()
hand = deck.deal(2)  # e.g. [Card(Rank.ACE, Suit.HEARTS), Card(Rank.TEN, Suit.DIAMONDS)]
print(hand[0])       # "AH"
```

---

### `evaluator.py`

Ranks any 5-card poker hand and finds the best possible hand from a set of 7 cards (2 hole + 5 community).

**`HandCategory` (IntEnum):** Ordered from worst to best:

| Value | Category |
|-------|----------|
| 0 | HIGH_CARD |
| 1 | ONE_PAIR |
| 2 | TWO_PAIR |
| 3 | THREE_OF_A_KIND |
| 4 | STRAIGHT |
| 5 | FLUSH |
| 6 | FULL_HOUSE |
| 7 | FOUR_OF_A_KIND |
| 8 | STRAIGHT_FLUSH |

**`HandRank` (frozen dataclass, ordered):** Stores a `(category, tiebreaker_tuple)`. The `order=True` on the dataclass means Python's `<`, `>` operators work directly, comparing `category` first, then the tiebreaker ranks. This is what makes `max(...)` over all 5-card combinations work correctly.

**`HandEvaluator.best_rank(cards)`:** Takes a list of 5–7 cards. Uses `itertools.combinations(cards, 5)` to generate every possible 5-card hand, ranks each one with `_rank_five`, and returns the best via `max`.

**`_rank_five(cards)`:** Builds the hand classification:
- Counts rank frequencies with `Counter`
- Checks for flush (all same suit)
- Checks for straight via `_straight_high` (handles the A-2-3-4-5 wheel case)
- Returns a `HandRank` with an appropriate tiebreaker tuple

**`_straight_high(ranks)`:** Returns the highest card of the straight if one exists, or `None`. Handles the special ace-low case by checking for the set `{14, 2, 3, 4, 5}`.

---

### `player.py`

A `dataclass` representing a player at the table.

| Field | Type | Description |
|-------|------|-------------|
| `name` | `str` | Display name |
| `chips` | `int` | Current chip count |
| `is_human` | `bool` | Whether this is the human player |
| `hole_cards` | `list[Card]` | Private cards dealt to this player |
| `current_bet` | `int` | Chips bet in the current hand |
| `folded` | `bool` | Whether the player has folded |

**`active` property:** Returns `True` if the player has not folded AND has chips remaining (`not self.folded and self.chips > 0`). Used to determine who is still in a hand or betting round.

**`reset_for_hand()`:** Clears `hole_cards`, resets `current_bet` to 0, sets `folded` to False. Called at the start of every new hand.

**`receive(cards)`:** Appends cards to `hole_cards`. Called during the deal.

**`bet(amount)`:** Deducts up to `amount` chips (capped by available chips), adds to `current_bet`, returns the actual amount wagered. Used by the original CLI game; the web version manages chip deduction directly.

---

### `table.py`

Holds state shared by all players during a hand.

| Field | Description |
|-------|-------------|
| `community_cards` | List of up to 5 cards visible to everyone |
| `pot` | Total chips currently in the pot |

**`reset()`:** Clears community cards and sets pot to 0. Called at the start of each hand.

**`add_to_pot(amount)`:** Increments the pot. The web version also accesses `table.pot` directly.

---

### `ui.py`

Console I/O helpers used only by the CLI game. The web app does not use this file.

**`show_table(community_cards, pot)`:** Prints community cards and pot size.

**`show_player(player)`:** Prints a player's name, hole cards, and chip count.

**`ask_action(player, call_amount)`:** Prints the player's situation and reads a line from stdin. Returns the raw string the user typed. This is the blocking `input()` call that made the CLI game unsuitable for direct use in a web server.

**`ask_raise_amount(minimum, maximum)`:** Loops until the user enters a valid integer in range.

**`show_message(message)`:** Prints a single string.

**`format_cards(cards)`:** Joins card strings with commas, e.g. `"AH, KD"`.

---

### `game.py`

The main game loop for the CLI version.

**`TexasHoldemGame.__init__`:** Stores players, blinds, creates a `Table`, `HandEvaluator`, and `ConsoleUI`. Raises `ValueError` if fewer than 2 players are provided.

**`play_hand()`:** Orchestrates a complete hand:
1. Shuffle a new deck, reset table and players
2. Deal 2 hole cards to each player
3. Post blinds
4. Show the human's cards
5. Run betting rounds for Pre-flop, Flop, Turn, River (stopping early if only one player remains)
6. Run showdown

**`_post_blinds()`:** Deducts small blind from `players[0]` and big blind from `players[1]`, adds both to the pot.

**`_betting_round(street)`:** The most complex method. Tracks `player_bets` (how much each player has put in this round) and `current_max_bet`. Loops over active players in order. For humans, calls `ui.ask_action`; for bots, calls `_bot_action`. Handles raises by updating `current_max_bet` and giving other players a chance to re-act. The loop ends when all active players have matched the current max bet and have acted.

**`_bot_action(player, call_amount)`:** Simple rule-based strategy:
- If `call_amount == 0` → check
- If `call_amount > player.chips * 0.5` → fold (too expensive)
- Otherwise → call

**`_showdown()`:** If one player remains, they win the pot. Otherwise, evaluates the best 5-card hand from each active player's hole cards plus community cards, and awards the pot to the best hand.

**`_only_one_player_left()`:** Returns `True` if exactly one player is still active.

---

### `main.py` (CLI)

```python
players = [
    Player("You", chips=1_000, is_human=True),
    Player("Ada Bot", chips=1_000),
    Player("Grace Bot", chips=1_000),
]
game = TexasHoldemGame(players)
game.play_hand()
```

Creates three players and plays one hand. Run with:
```bash
cd Poker_game
python main.py
```

---

## 4. The Web Application — `backend/`

### Why a rewrite was needed

The CLI game uses `input()` in `ui.py` and `game.py`. This is a **blocking call** — it freezes the process until the user types something. A web server cannot do this; it must handle many requests concurrently and cannot block while waiting for a browser to send a form.

The solution is to convert the game into a **state machine**: instead of a loop that pauses mid-execution waiting for input, the game stores all its state in memory and exposes two transitions:

- **`get_state()`** — snapshot the current situation
- **`submit_action(action)`** — advance the game by one human decision

Between those two calls, the server immediately runs all bot actions without blocking.

---

### `game_session.py` — The State Machine

`GameSession` is the core of the web application. One instance lives in memory per active game.

#### Construction

```python
session = GameSession(game_id)
```

Creates three players (`You`, `Ada Bot`, `Grace Bot`) each with 1000 chips, a fresh `Table`, a `HandEvaluator`, and initializes all betting state to empty/zero.

#### Betting state fields

| Field | Purpose |
|-------|---------|
| `player_bets` | `dict[player_index → chips_committed_this_round]` |
| `current_max_bet` | The current highest bet in this round |
| `acted_players` | Set of player indices who have acted this round |
| `action_queue` | Ordered list of player indices yet to act |
| `awaiting_action` | `True` when the human must act before anything can proceed |
| `current_actor_idx` | Which player is currently being prompted |
| `current_call_amount` | How many chips the human needs to match the current max bet |

#### Game phases

```
idle → pre_flop → flop → turn → river → hand_complete
                                       ↘ game_over (if human out of chips)
```

Phases only advance forward. `_advance_phase()` reads the current phase and transitions to the next one, dealing community cards as needed.

#### `start_hand()`

Called at the beginning of each hand:
1. Checks if the human has chips; if not, sets `phase = "game_over"` and returns
2. Creates a new `Deck`, resets the table and all players
3. Deals 2 cards to each player (one at a time, in turn order)
4. Posts blinds (player index 0 = small blind, index 1 = big blind)
5. Sets phase to `"pre_flop"` and calls `_start_betting_round(is_preflop=True)`

#### `_start_betting_round(is_preflop)`

Initialises the betting state for one round (Pre-flop, Flop, Turn, or River):

- Clears `player_bets`, `acted_players`, resets `awaiting_action`
- For pre-flop: seeds `player_bets[0] = small_blind`, `player_bets[1] = big_blind`, sets `current_max_bet = big_blind`, and starts action at player index 2 (UTG, the one after the big blind)
- For all other streets: starts action at player index 0 with `current_max_bet = 0`
- Builds `action_queue` by iterating from `start_idx` around the table, including only active players
- Immediately calls `_process_queue()` to run any bot actions

#### `_process_queue()` — The core loop

This is the engine of the state machine. It processes the `action_queue` until either:

- The human's turn comes (sets `awaiting_action = True` and returns, leaving the queue intact)
- The queue is exhausted (all players have matched the bet → advance to next phase)
- Only one active player remains (→ `_end_hand()`)

For each player at the front of the queue:

1. Skip if not active (folded or out of chips)
2. Skip if already acted at the current max bet (settled)
3. Compute `call_amount = current_max_bet - player_bets[actor]`
4. If human → record `awaiting_action = True`, `current_call_amount`, return
5. If bot → call `_bot_action()`, pop from queue, call `_apply_action()`, continue loop

#### `_apply_action(idx, action, call_amount)`

Applies one player's decision and updates state:

**Fold:**
- Sets `player.folded = True`
- Adds to `acted_players`

**Raise:**
- Computes `raise_to = current_max_bet + big_blind` (fixed raise size = 1 big blind)
- Computes `additional = raise_to - player_bets[idx]`
- If player has enough chips: deducts `additional`, adds to pot, updates `player_bets[idx]` and `current_max_bet`
- **Rebuilds `action_queue`:** after a raise, every other active player must re-act. The new queue contains all active players in order starting from the seat after the raiser
- If not enough chips to raise: silently falls through to "call"

**Call / Check:**
- If `call_amount == 0`: logs "checks"
- Otherwise: deducts `min(call_amount, player.chips)` from player, adds to pot

#### `submit_action(action)`

Called by the API when the human submits a decision:

1. Validates that `awaiting_action` is True
2. Records the stored actor index and call amount
3. Sets `awaiting_action = False`
4. Pops the human from the front of `action_queue`
5. Calls `_apply_action` with the submitted action
6. Calls `_process_queue()` to run subsequent bot actions and potentially advance the phase

#### `_advance_phase()`

Transitions the game to the next street:

| From | To | Deals |
|------|-----|-------|
| `pre_flop` | `flop` | 3 community cards |
| `flop` | `turn` | 1 community card |
| `turn` | `river` | 1 community card |
| `river` | showdown | nothing |

Each transition logs a separator message (e.g. `"-- Flop: 6C 4D AD --"`) and calls `_start_betting_round()` for the new street.

#### `_showdown()`

Called after river betting:
- If 1 active player: they win the pot uncontested
- If multiple active: evaluates each player's best 5-card hand from their 2 hole cards + 5 community cards using `HandEvaluator.best_rank()`, awards the pot to the winner, appends the hand category label to messages (e.g. `"Two Pair"`)
- Sets `phase = "hand_complete"`

#### `_end_hand()`

Called when all but one player folds mid-street:
- Awards the pot to the last remaining active player
- Sets `phase = "hand_complete"`

#### `get_state()`

Returns a JSON-serializable snapshot of the full game state. Card visibility rules:

- **Human player:** always shows real hole cards
- **Bot players during play:** shows `["??", "??"]` (hidden card backs)
- **Bot players at `hand_complete` or `game_over`:** shows real cards (the reveal)
- **Folded players:** shows `[]` (no cards displayed)

Full response shape:

```json
{
  "game_id": "f5d6efe3-...",
  "phase": "pre_flop",
  "awaiting_action": true,
  "call_amount": 5,
  "community_cards": [],
  "pot": 25,
  "players": [
    {
      "index": 0,
      "name": "You",
      "chips": 985,
      "is_human": true,
      "hole_cards": ["9D", "TD"],
      "folded": false,
      "active": true,
      "is_current_actor": true
    },
    {
      "index": 1,
      "name": "Ada Bot",
      "chips": 990,
      "is_human": false,
      "hole_cards": ["??", "??"],
      "folded": false,
      "active": true,
      "is_current_actor": false
    }
  ],
  "messages": [
    "You posts small blind 5",
    "Ada Bot posts big blind 10",
    "Grace Bot calls 10"
  ]
}
```

---

### `main.py` — The FastAPI App

**Session store:** A plain `dict[str, GameSession]` held in process memory. This is fine for a single-process demo; a production version would use Redis or a database.

**Route registration order matters:** FastAPI routes are matched in the order they are registered. The `StaticFiles` mount at `/` is registered last, so it acts as a catch-all for everything not matched by an API route. If the static mount were registered first, it would shadow the API routes.

#### Routes

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/game/new` | Creates a session, starts first hand, returns initial state |
| `GET` | `/api/game/{game_id}` | Returns current state snapshot |
| `POST` | `/api/game/{game_id}/action` | Submits human action, returns new state |
| `POST` | `/api/game/{game_id}/next-hand` | Starts the next hand, returns new state |
| `GET` | `/*` | Serves static files (`index.html`, `style.css`, `app.js`) |

**Error responses:**
- `404` if `game_id` is not found
- `400` if `action` is not one of `fold`, `call`, `raise`
- `409` if `/action` is called when `awaiting_action` is False, or `/next-hand` when the hand is not complete

**Static file path:** Resolved with `os.path.dirname(__file__)`, so it always points to `backend/static/` regardless of where `uvicorn` is launched from. The `html=True` flag on `StaticFiles` makes requests to `/` automatically serve `index.html`.

---

### Frontend — `static/`

The frontend is a single-page application with no framework dependencies — plain HTML, CSS, and JavaScript.

#### `index.html`

The page skeleton. Contains five main regions:

1. **`.site-header`** — Title and "New Game" button
2. **`#opponents`** — Dynamically filled with bot player seats
3. **`.community-section`** — Phase label, community cards, pot display
4. **`.human-section`** — The human player's hole cards and chip count
5. **`.action-bar`** — Fold / Call / Raise / Next Hand buttons
6. **`.log-panel`** — Scrollable game event log (rendered to the right)

All content inside those regions is injected by JavaScript — the HTML itself has almost no hardcoded game data.

#### `style.css`

Uses CSS custom properties (`--felt`, `--gold`, etc.) to define the color palette in one place.

**Layout:** `.main-layout` is a flexbox row — the poker table takes all available space (`flex: 1`), and the log panel has a fixed width of 260px. On screens narrower than 700px, the layout switches to a column via a `@media` query.

**The felt:** The `.felt` element uses a `radial-gradient` from bright green in the centre to darker green at the edges, with a thick dark border and `border-radius: 140px` to create an oval shape. An inner `box-shadow` adds depth.

**Cards:** `.card` is a 58×88px white div with `font-family: Georgia` (serif, for a classic playing card feel). It has three children:
- `.card-top` — rank and suit in the top-left
- `.card-center` — large suit symbol centred with `position: absolute`
- `.card-bottom` — rank and suit in the bottom-right, rotated 180° to mirror the top (standard playing card layout)

Red suits (Hearts ♥, Diamonds ♦) get `color: var(--card-red)`. Black suits get `color: var(--card-black)`.

**Card backs:** `.card-back` overrides the background with a diagonal stripe pattern using `repeating-linear-gradient` over a dark blue base.

**Buttons:** All buttons share the `.btn` base class with shared padding, border-radius, and transition. Hover lifts the button 2px with `transform: translateY(-2px)`. Disabled buttons drop to 35% opacity. Each action type has its own colour class (`btn-fold` = red, `btn-call` = green, `btn-raise` = amber, `btn-next` = gold).

#### `app.js`

**Global state:** `gameId` holds the UUID of the current game session. It is `null` until the first API call succeeds.

**`renderCard(code)`:** Parses a 2-character card code:
- `"??"` → returns a `.card-back` div
- Any other code (e.g. `"AH"`) → maps `T` to `10`, maps suit letter to Unicode symbol (`♥ ♦ ♣ ♠`), determines color, and builds the full card HTML

**`renderCards(codes)`:** Maps `renderCard` over an array, or shows a dash if the array is empty.

**`render(state)`:** The main render function. Called after every API response. It:
1. Updates the phase label
2. Replaces community card HTML
3. Updates pot display
4. Clears and rebuilds the opponents div (skipping the human player)
5. Updates the human's cards and chip label
6. Enables/disables action buttons based on `awaiting_action`
7. Updates the Call button label to show the call amount (`"Call 10"`) or `"Check"` if `call_amount == 0`
8. Shows/hides the "Next Hand" button based on phase; when phase is `game_over`, changes it to "New Game" and rewires its `onclick`
9. Re-renders the log, applying the `.separator` CSS class to lines starting with `"--"` (street announcements)

**`escHtml(str)`:** Sanitises any user-visible string before inserting it into the DOM via `innerHTML`, preventing XSS from unexpected message content.

**`apiPost(url, body)`:** Thin wrapper around `fetch`. Sends `Content-Type: application/json` when a body is provided. Throws on non-2xx responses, extracting the FastAPI `detail` field from the JSON error.

**Boot:** `window.addEventListener("DOMContentLoaded", newGame)` — a new game starts automatically when the page loads.

---

## 5. Docker Setup

### `backend/Dockerfile`

```dockerfile
FROM python:3.12-slim      # Minimal Python image (~170 MB vs ~900 MB full)

WORKDIR /app               # All subsequent paths are relative to /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt  # Install deps before copying
                                                     # source (layer caching)

COPY *.py ./               # All Python source files
COPY static/ ./static/     # Frontend assets

EXPOSE 8000                # Documents the port (does not publish it)

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

The `COPY *.py` / `COPY static/` split means editing source files does not invalidate the pip install layer — Docker only re-runs `pip install` if `requirements.txt` changes.

`--host 0.0.0.0` is required inside Docker: the default `127.0.0.1` would only accept connections from inside the container, making it unreachable from the host.

### `backend/.dockerignore`

```
__pycache__
*.pyc
*.pyo
.git
```

Prevents compiled bytecode and version-control data from being copied into the image.

### `docker-compose.yml`

```yaml
services:
  poker:
    build: ./backend       # Build the image from backend/Dockerfile
    ports:
      - "8000:8000"        # host_port:container_port
    restart: unless-stopped
```

`restart: unless-stopped` means the container automatically restarts after crashes or system reboots, but not if you manually stop it with `docker compose stop`.

### `requirements.txt`

```
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.9.2
```

- `fastapi` — the web framework
- `uvicorn[standard]` — the ASGI server; `[standard]` adds `httptools` and `uvloop` for better performance
- `pydantic` — used by FastAPI for request body validation (`ActionRequest` model)

All versions are pinned to exact releases to ensure reproducible builds.

---

## 6. How to Run

### Option A — Docker (recommended)

```bash
# From the poker/ directory
docker compose up
```

Open `http://localhost:8000` in your browser. The game starts automatically.

To stop:
```bash
docker compose down
```

To rebuild after code changes:
```bash
docker compose up --build
```

### Option B — Local Python

```bash
cd backend
pip install -r requirements.txt
uvicorn main:app --host 127.0.0.1 --port 8000 --reload
```

Open `http://localhost:8000`.

`--reload` enables hot-reload: the server restarts automatically when you save a Python file. Do not use `--reload` in production.

### Option C — CLI only (original game)

```bash
cd Poker_game
python main.py
```

Plays one hand in the terminal. Does not use FastAPI or Docker.

---

## 7. API Reference

All endpoints return the same **game state object** (see the `get_state()` section above for the full schema).

### `POST /api/game/new`

Creates a fresh session and starts the first hand.

**Request body:** none

**Response:** Initial game state. `awaiting_action` is usually `True` immediately because bots act instantly for the pre-flop positions before the human.

---

### `GET /api/game/{game_id}`

Returns the current state of an existing session without modifying anything. Useful for polling or debugging.

**Response:** Game state object

**Errors:** `404` if `game_id` is unknown

---

### `POST /api/game/{game_id}/action`

Submits the human player's action for the current turn.

**Request body:**
```json
{ "action": "fold" }
```
Valid values: `"fold"`, `"call"`, `"raise"`

- `"fold"` — surrender the hand; the player's `folded` flag is set to `true`
- `"call"` — match the current bet; if `call_amount` is 0, this is a check
- `"raise"` — increase the bet by one big blind (10 chips); if insufficient chips, falls back to call

After processing the human's action, the server immediately runs all subsequent bot actions and may advance through multiple betting phases (e.g., if all bots fold instantly, the hand ends in one call).

**Errors:**
- `400` — invalid action string
- `404` — game not found
- `409` — `awaiting_action` is False (it's not the human's turn)

---

### `POST /api/game/{game_id}/next-hand`

Resets the message log and starts a new hand. Chip counts carry over from the previous hand.

**Request body:** none

**Errors:**
- `404` — game not found
- `409` — current hand is not complete (`phase` is not `hand_complete` or `game_over`)

**Special case:** If the human has 0 chips, `start_hand()` sets `phase = "game_over"` instead of dealing cards. The frontend then shows a "New Game" button (which calls `POST /api/game/new`) rather than "Next Hand".

---

## 8. Data Flow — A Full Hand

```
Browser                       FastAPI (main.py)        GameSession
   |                               |                       |
   |-- POST /api/game/new -------->|                       |
   |                               |-- new GameSession() ->|
   |                               |-- session.start_hand()|
   |                               |     deal cards        |
   |                               |     post blinds       |
   |                               |     bots act (UTG)    |
   |                               |     → awaiting human  |
   |<-- 200 state (awaiting=true) -|                       |
   |                               |                       |
   |  [User clicks "Call 5"]       |                       |
   |                               |                       |
   |-- POST /action {"action":"call"} -->|                 |
   |                               |-- submit_action("call")|
   |                               |     apply call        |
   |                               |     BB checks         |
   |                               |     → advance to flop |
   |                               |     deal 3 cards      |
   |                               |     bots check        |
   |                               |     → awaiting human  |
   |<-- 200 state (phase="flop") --|                       |
   |                               |                       |
   |  [User clicks "Raise"]        |                       |
   |                               |                       |
   |-- POST /action {"action":"raise"} ->|                 |
   |                               |-- submit_action("raise")|
   |                               |     apply raise       |
   |                               |     bots call         |
   |                               |     → advance to turn |
   |                               |     deal 1 card       |
   |                               |     bots check        |
   |                               |     → awaiting human  |
   |<-- 200 state (phase="turn") --|                       |
   |                               |                       |
   |  [User clicks "Check"/"Call"] |                       |
   |         ... (river) ...       |                       |
   |                               |                       |
   |-- POST /action {"action":"call"} -->|                 |
   |                               |     river bets settle |
   |                               |     → _showdown()     |
   |                               |     cards revealed    |
   |                               |     winner announced  |
   |<-- 200 state (phase="hand_complete") |               |
   |                               |                       |
   |  [User clicks "Next Hand ▶"]  |                       |
   |                               |                       |
   |-- POST /next-hand ----------->|                       |
   |                               |-- session.start_hand()|
   |                               |     (with carried chips)|
   |<-- 200 state (phase="pre_flop") |                    |
```

---

## 9. Known Limitations

**All-in not supported.** The `Player.active` property returns `False` when `chips == 0`. A player who has committed all their chips (gone all-in) would be treated as if they folded in subsequent betting rounds. This is a deliberate simplification — with starting stacks of 1000 and blinds of 5/10, it takes many hands to approach this situation.

**Single winner per hand.** The showdown logic picks the single best hand. Split pots (when two players have the same best hand) are not handled — one player wins arbitrarily.

**Fixed raise size.** Raises are always exactly one big blind (10 chips). There is no custom raise amount input.

**No blind rotation.** The small blind is always `players[0]` ("You") and the big blind is always `players[1]` ("Ada Bot"). In real poker, the dealer button rotates each hand.

**In-memory sessions only.** Restarting the server loses all game sessions. There is no persistence.

**Single game at a time per browser.** The `gameId` is stored in a JavaScript variable, so refreshing the page or opening a second tab starts a new game. Old sessions remain in server memory until the process restarts.

**No multiplayer.** The human player is hardcoded as player index 0. Multiple real players on the same server are not supported.
