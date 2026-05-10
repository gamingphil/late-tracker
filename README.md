# Late Tracker

A daily betting app my friend group. Every morning everyone predicts whether a specific friend will arrive late. The admin records the actual result, scores are tallied, and a leaderboard tracks who predicts best over time.

Built with plain HTML/JS on the frontend and a Cloudflare Worker as the backend, with JSONBin as the data store.

## Using the app

Open the website, enter your name on first visit, and place your bet. Other people's bets are hidden until you've placed your own. The admin records the result once the outcome is known.

On iOS, the website can be added as an app on your home screen by opening the site in Safari, tapping the Share button and selecting "Add to Home Screen".
I don't know if a similar thing is possible on non-iOS devices.

### Identity
 
You are identified solely by the name you enter. There are no accounts or passwords for regular users. Use your name consistently across devices. If you bet as "Phil" on your phone and "philip" on your laptop, the app treats those as two different people and your scores will be split.

---

## API

The Cloudflare Worker is a public HTTP API. All POST requests send and receive JSON.

**Base URL:** `https://late-tracker.joshua-crawford.workers.dev`

### `GET /data`

Returns the full current state of the bin.

```
GET /data
```

**Response**
```json
{
  "todayBets": {
    "2026-05-10": {
      "friendA": "late",
      "friendB": "ontime"
    }
  },
  "history": [
    {
      "date": "2026-05-09",
      "result": "late",
      "bets": { "friendA": "late", "friendB": "ontime" },
      "winners": ["friendA"],
      "losers": ["friendB"]
    }
  ],
  "scores": {
    "friendA":  { "correct": 4, "total": 4 },
    "friendB": { "correct": 3, "total": 4 }
  }
}
```

---

### `POST /bet`

Place a bet for today.

**Rules enforced server-side:**
- A name cannot bet twice on the same day
- Betting is blocked once the result for that day has been recorded

**Body**
```json
{
  "name":   "Phil",
  "choice": "late",
  "date":   "2026-05-10"
}
```

`choice` must be `"late"` or `"ontime"`.  
`date` must be an ISO date string in `YYYY-MM-DD` format.

**Response (success)**
```json
{ "success": true }
```

**Response (error)**
```json
{ "error": "You have already placed a bet today" }
```

---

### `POST /result`

Record or correct the result for a given day. Admin-only (requires the admin password).

**Body**
```json
{
  "password": "admin-password",
  "result":   "late",
  "date":     "2026-05-10",
  "edit":     false
}
```

`result` must be `"late"` or `"ontime"`.  
Set `edit` to `true` to correct an already-recorded result. Scores are recalculated automatically.

**Response (success)**
```json
{ "success": true, "winnersCount": 2 }
```

---

### `POST /clear`

Wipes all history, bets, and scores. Admin-only.

**Body**
```json
{ "password": "admin-password" }
```

---

## Example code in Python

```python
import requests
from datetime import date

BASE = 'https://late-tracker.joshua-crawford.workers.dev'
TODAY = date.today().isoformat()  # e.g. "2026-05-10"

# Read current data
data = requests.get(f'{BASE}/data').json()
print(data['scores'])

# Place a bet
response = requests.post(f'{BASE}/bet', json={
    'name':   'Phil',
    'choice': 'late',
    'date':   TODAY
})
print(response.json())  # {"success": true} or {"error": "..."}
```

### Error handling

Non-2xx responses always include an `"error"` field explaining what went wrong. A simple pattern:

```python
def api_post(path, body):
    r = requests.post(f'{BASE}{path}', json=body)
    result = r.json()
    if not r.ok:
        raise Exception(result.get('error', 'Unknown error'))
    return result
```

---

## Data format notes

- Dates are always `YYYY-MM-DD` strings in local time (each device uses its own clock).
- `todayBets` accumulates entries for every day indefinitely; days with no recorded result are simply never scored.
- `scores` is a running tally updated incrementally on each `/result` call.

---

## Fair play

The Worker enforces the rules: you cannot bet twice, bet after the result is in, or record a result without the admin password, regardless of whether you use the website or call the API directly. Beyond that, automated or model-based bets are fair game. Good luck.