# Whats_App_group_Service — a WebSocket chat you can take apart

A small chat server built with **Python + Flask + WebSockets** to learn how
real-time messaging actually works:

- one-to-one messages
- group messages (fan-out)
- what happens when someone is **offline** (store-and-forward)
- ✓ sent, ✓✓ delivered, blue ✓✓ read receipts
- presence ("online" / "last seen") and typing indicators

There is no Socket.IO and no hidden magic. The connection registry, the
offline queue and the group fan-out are each about 20 lines of plain Python.

---

## 1. Run it

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

python app.py                 # terminal 1: the server, port 8000
python demo.py                # terminal 2: scripted walk-through of every scenario
```

Then open **http://localhost:8000** in 2–4 browser tabs (or windows side by side),
and log in as `alice`, `bob`, `charlie` and `dave`. Each tab has a **wire log**
on the right that shows every raw WebSocket frame it sends (↑) and receives (↓).
The server terminal narrates the same events from the server's side.

> Port 8000 is used because macOS AirPlay Receiver takes port 5000.
> Set `PORT=9000 python app.py` to change it.

---

## 2. What a WebSocket is

Plain HTTP is **request → response, then hang up**. The server can't talk
unless the client asks first. For chat, that would mean polling every second:
"any new messages? … any now? … now?"

A **WebSocket** starts life as an ordinary HTTP request that asks to *upgrade*:

```
Browser                                        Server
   │  GET /ws?user=alice HTTP/1.1                 │
   │  Upgrade: websocket                          │
   │  Connection: Upgrade                         │
   │  Sec-WebSocket-Key: dGhlIHNhbXBsZQ==         │
   │ ───────────────────────────────────────────► │
   │                                              │
   │  HTTP/1.1 101 Switching Protocols            │
   │  Upgrade: websocket                          │
   │  Sec-WebSocket-Accept: s3pPLMBiTxaQ9k...     │
   │ ◄─────────────────────────────────────────── │
   │                                              │
   │  ══════ same TCP connection, now a ══════    │
   │  ══════  two-way pipe of "frames"  ══════    │
   │                                              │
   │  {"type":"send","to":"bob","text":"hi"}  ──► │
   │ ◄── {"type":"message", ...}                  │   server can push
   │ ◄── {"type":"presence","user":"bob",...}     │   any time it wants
   │       ping ◄──►  pong  (keep-alive)          │
```

Key properties:

| | HTTP | WebSocket |
|---|---|---|
| Who can start talking | only the client | **either side, any time** |
| Lifetime | one request | **stays open** (minutes, hours) |
| Overhead per message | full headers | 2–14 byte frame header |
| Server knows you're "there" | no | **yes, while the socket is open** |

That last row is the whole basis of "online": **a user is online exactly
when the server holds an open socket for them.**

In Flask (via `flask-sock`), each connection is one function call that runs
in its own thread for as long as the socket is open:

```python
@sock.route("/ws")
def ws_handler(ws):
    ...register...
    while True:
        frame = ws.receive()   # blocks until the client sends something
        handle(frame)
```

---

## 3. Architecture

```
                 ┌──────────────────────────── Flask process ───────────────────────────┐
                 │                                                                       │
 alice tab ──ws──┤  ws_handler thread ─┐                                                 │
 alice phone─ws──┤  ws_handler thread ─┤    ConnectionManager  (in memory)               │
 bob tab ────ws──┤  ws_handler thread ─┼──►   "alice" → {sock1, sock2}                   │
                 │                     │      "bob"   → {sock3}                          │
                 │                     │      (dave missing → OFFLINE)                   │
                 │                     │                                                 │
                 │                     └──►  store.py  (SQLite, durable)                 │
                 │                             messages  : what was said                 │
 any tab ──HTTP──┤  /api/history  ────────►    receipts  : per recipient, how far it got │
                 │  /api/users                           pending → delivered → read      │
                 └───────────────────────────────────────────────────────────────────────┘
```

| File | Role |
|---|---|
| [app.py](app.py) | Flask app, the `/ws` handler, and one handler per frame type |
| [connections.py](connections.py) | **Live** state: who is connected, on which sockets |
| [store.py](store.py) | **Durable** state: messages + receipts in SQLite |
| [static/app.js](static/app.js) | Browser client: ACKs, read receipts, outbox, reconnect |
| [demo.py](demo.py) | Scripted Python clients that run every scenario |
| [log.py](log.py) | Coloured server narration |

The most important idea in this project is that **there are two kinds of state**:

1. **Live** (`ConnectionManager`): who is reachable *right now*. It is lost on restart, and that's fine.
2. **Durable** (`store.py`): what must eventually be delivered. It survives restarts and disconnects.

Every message goes to **durable first**, and then to **live if possible**.

---

## 4. The protocol (JSON frames)

Every frame is a JSON object with a `type`.

**Client → server**

| type | example | meaning |
|---|---|---|
| `send` | `{"type":"send","to":"bob","text":"hi","client_id":"c1"}` | 1:1 message |
| `send` | `{"type":"send","group":"friends","text":"hi","client_id":"c2"}` | group message |
| `ack` | `{"type":"ack","message_id":"…"}` | "my device received it" |
| `read` | `{"type":"read","message_id":"…"}` | "I've seen it on screen" |
| `typing` | `{"type":"typing","to":"bob"}` | ephemeral, not stored |
| `create_group` | `{"type":"create_group","name":"Trip","members":["bob"]}` | |

**Server → client**

| type | meaning |
|---|---|
| `hello` | first frame after connecting: your users and groups |
| `message` | a message for you (`offline: true` if it waited in the queue) |
| `sent` | your message is stored on the server (✓); carries `client_id` so you can match it |
| `status` | a receipt changed: `by`, `status`, and the `aggregate` tick you should show |
| `presence` | someone came online / went offline (+ `last_seen`) |
| `typing` | someone is typing |
| `group_added` | you were added to a group |

---

## 5. One-to-one, both online

```
 alice                       server                                  bob
   │ send{to:bob,"hi",c1}      │                                      │
   │ ────────────────────────► │ 1. INSERT message                    │
   │                           │    INSERT receipt(bob, pending)      │
   │ ◄──── sent{c1, id=42}     │ 2. tell alice it's safe        ✓     │
   │                           │ 3. registry["bob"] → {sock}          │
   │                           │ ── message{id=42} ─────────────────► │
   │                           │ ◄──────────────────── ack{42} ────── │ 4. bob's device confirms
   │                           │    UPDATE receipt → delivered        │
   │ ◄── status{42,delivered}  │                               ✓✓     │
   │                           │ ◄──────────────────── read{42} ───── │ 5. bob opens the chat
   │ ◄── status{42,read}       │    UPDATE receipt → read   blue ✓✓   │
```

Code: `on_send` → `store.save_message` → `manager.send_to_user` in [app.py](app.py).

**Why an explicit `ack`, instead of "the write succeeded"?**
`ws.send()` returning only means the bytes went into a kernel buffer. Bob's
phone may have entered a tunnel a second ago, and the OS hasn't noticed yet.
Only bob's *app* can say "I have it". So the server stays at ✓ until it hears
`ack`, and if the ack never comes, the message is still `pending` and gets
re-sent on bob's next connect.

---

## 6. One-to-one, recipient OFFLINE (store-and-forward)

```
 alice                       server                         bob (no socket)
   │ send{to:bob,"you there?"} │
   │ ────────────────────────► │ INSERT message, receipt(bob, pending)
   │ ◄──── sent  ✓             │ registry["bob"] → nothing
   │                           │ → do nothing else. It's in the DB.
   │   … minutes / days …      │
   │                           │ ◄═══ bob connects (GET /ws?user=bob) ═══
   │                           │ deliver_pending(bob):
   │                           │   SELECT … WHERE receipt.user=bob AND status='pending'
   │                           │ ── message{offline:true} ───────────────► │
   │                           │ ◄─────────────────────────── ack ──────── │
   │ ◄── status{delivered} ✓✓  │
```

Code: `deliver_pending()` in [app.py](app.py) and `pending_for()` in [store.py](store.py).

What if **alice** is offline when bob's ack arrives? The receipt is still
written to the DB. Alice simply gets no live `status` frame. When she reopens
the chat, `/api/history` returns the up-to-date ticks.

What if the **server crashes** mid-flush? The receipts are still `pending`, so
everything is re-sent on the next connect. The client de-duplicates by
`message.id`. This is **at-least-once delivery + idempotent receiver**, the
standard recipe.

### When the *sender* is offline: the client outbox

The browser has its own mirror image of the server queue. If alice's socket is
down when she hits Send, the frame goes into `state.outbox` and shows 🕓. When
the socket reconnects (`ws.onopen`), the outbox is flushed. See `sendMessage`
and `flushOutbox` in [static/app.js](static/app.js).

```
   🕓 queued (in browser outbox)  →  ✓ sent (in server DB)  →  ✓✓ delivered  →  blue ✓✓ read
```

---

## 7. Group messages (fan-out)

A group is just a list of members. Sending to a group means **one message row
plus one receipt row per member** (everyone except the sender). This is called
*fan-out on write*:

```
 alice ── send{group:friends} ──► server
                                    │ members = [alice, bob, charlie, dave]
                                    │ recipients = [bob, charlie, dave]
                                    │ INSERT message
                                    │ INSERT receipt(bob, pending)
                                    │ INSERT receipt(charlie, pending)
                                    │ INSERT receipt(dave, pending)
                                    │
                                    ├──► bob      online  → push → ack → delivered
                                    ├──► charlie  online  → push → ack → delivered
                                    └──╳ dave     OFFLINE → stays pending
                                                 … later, dave connects → flushed → ack
```

**Group ticks** show the *minimum* over all receipts, like WhatsApp:

| receipts | alice sees |
|---|---|
| bob=delivered, charlie=delivered, dave=pending | ✓ |
| all delivered | ✓✓ |
| bob=read, charlie=read, dave=delivered | ✓✓ |
| all read | blue ✓✓ |

See `aggregate_status()` in [store.py](store.py). Hover over a message you
sent in the browser to see the per-member breakdown.

So **1:1 is just a group of two.** The code path is the same, and only the
recipient list differs.

---

## 8. Presence, last seen, heartbeats

- **Online** = `ConnectionManager` has ≥1 socket for you. When your *first*
  socket opens, the server broadcasts `presence{online:true}`. When your *last*
  socket closes, it stores `last_seen` and broadcasts `presence{online:false}`.
- **Multiple devices**: `alice → {laptop, phone}`. A message to alice is written
  to *every* socket. Your own sent messages are mirrored to your other devices.
  Open two tabs as `alice` to see this.
- **Dead connections**: a phone that loses signal never sends a "close" frame.
  The server sends a WebSocket **ping every 25s** (`SOCK_SERVER_OPTIONS` in
  `app.py`). No pong means the socket is closed, and the user goes offline.
- **Reconnect**: the browser retries with exponential backoff (1s, 2s, 4s … max 10s).

**Durable vs ephemeral events:**

| Stored and re-delivered | Fire-and-forget (dropped if offline) |
|---|---|
| messages, receipts, group membership | typing, presence changes |

Nobody wants "bob is typing…" from three hours ago.

---

## 9. Experiments to try

1. **1:1 live.** Tabs as alice and bob. Send. Watch the ticks go ✓ → ✓✓ → blue ✓✓, and the `ack`/`read` frames in bob's wire log.
2. **Recipient offline.** In bob's tab click **Go offline**. Send bob three messages from alice (they stay ✓). Look at the server terminal: `kept as 'pending' in DB`. Click **Go online** in bob's tab. The messages arrive tagged *arrived after reconnect*, and alice flips to ✓✓.
3. **Sender offline.** In alice's tab click **Go offline**, then send. You get 🕓 and "put in the OUTBOX" in the wire log. Go online, and the outbox flushes.
4. **Group with someone offline.** Close dave's tab. Send to *Friends*. The tick stays ✓ even after bob and charlie get it, until dave logs in.
5. **Server restart.** With bob offline, send him messages, then stop the server (Ctrl-C) and start it again. Log in as bob: the messages are still delivered, because they were in SQLite, not in memory.
6. **Multi-device.** Two tabs as alice. Send from one, and it appears in the other.
7. **Look at the DB:** `sqlite3 chat.db "select username,status,count(*) from receipts group by 1,2"`.

---

## 10. From here to production

This project is deliberately small. A real messenger adds:

| Problem | This demo | Real world |
|---|---|---|
| **Identity** | `?user=alice` (anyone can be anyone!) | auth token / session checked during the handshake |
| **More than one server** | registry is in one process's memory | Redis pub/sub or a message broker: "deliver to bob" is published, and whichever server holds bob's socket pushes it. Plus a shared presence store. |
| **App fully closed (phone)** | message waits until next connect | **push notification** (APNs / FCM) to wake the app, which then connects and flushes |
| **Duplicate sends on flaky network** | client re-sends from outbox | server de-dupes on `(sender, client_id)` |
| **Huge groups** | one receipt row per member | fan-out on *read* for big channels, cursors instead of per-message receipts |
| **Ordering** | sorted by timestamp | per-conversation sequence numbers |
| **Encryption** | plaintext | end-to-end encryption (Signal protocol): the server only relays ciphertext |
| **Server** | Werkzeug dev server, a thread per socket | gunicorn + gevent/eventlet, or an async server (Quart, FastAPI, aiohttp) to hold 10k+ sockets |
# Whats_App_group_Service
