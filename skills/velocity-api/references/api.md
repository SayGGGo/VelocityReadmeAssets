# Working with Velocity

All JSON responses use `data`; errors use `error.code`, `error.message` and
`request_id`. OAuth token endpoints use the standard OAuth error response.
Amounts ending `_cents` or `Cents` are RUB kopecks: 29000 = 290 ₽.

| Workflow | Endpoint | Scope |
|---|---|---|
| Read owner | GET /me | profile:read |
| Include private profile fields | GET /me | additionally profile:personal |
| Quotas and credential scopes | GET /usage | any valid credential |
| Inference models | GET /models | models:read |
| Stateless generation | POST /responses | ai:generate |
| Catalog | GET /marketplace/models | models:read |
| Publish/update/delete own offer | POST /marketplace/models; PATCH/DELETE /marketplace/models/{id} | models:write |
| Plans and wallet | GET /plans; GET /wallet | billing:read |
| Buy API Pro or account plan | POST /subscriptions | billing:purchase |
| Notifications | GET /notifications | notifications:read |
| Own bots | GET/POST /bots | bots:write |
| Deliver notification | POST /bots/{id}/messages | bots:write + recipient opt-in |
| Propose tab | GET/POST /dashboard/tabs | tabs:write + dashboard approval |

```python
import json, os, urllib.request
token = os.environ['VELOCITY_API_KEY']
req = urllib.request.Request(
    'https://api.isvelocity.ru/public/v1/responses',
    data=json.dumps({'model':'air', 'messages':[{'role':'user','content':'Explain a derivative simply.'}], 'max_output_tokens':512}).encode(),
    headers={'Authorization':'Bearer '+token,'Content-Type':'application/json'})
with urllib.request.urlopen(req, timeout=90) as response:
    print(json.load(response)['data']['output'])
```

Generate without creating a persistent chat. For follow-up requests pass previous
user/assistant messages explicitly; never paste tokens into message content.

## Quotas

API Free: 10000 HTTP requests, 50 generations, 50000 AI tokens per calendar month
UTC; 30 HTTP requests/min, 2 generations/min, 5 keys.
API Pro: 290 ₽/30 days; 100000 HTTP requests, 5000 generations, 5000000 AI tokens
per calendar month UTC; 120 HTTP requests/min, 10 generations/min, 20 keys.
Purchasing Pro does not reset existing usage. `/usage` is authoritative.
Requests count across keys and OAuth clients against the authorizing user.
50 messages, 24000 text bytes, max_output_tokens up to 4096 per generation.
The server reserves a conservative token budget and refunds its unused part.

## OAuth

Register client and exact redirect URI in the dashboard. Generate a 43–128
character verifier, S256 challenge and random state. Open `/oauth/authorize` in
the user's browser. Exchange the returned one-use code at `/oauth/token` using
form encoding and the original verifier. Access tokens last 1h, refresh tokens
30d. Refresh rotates the token; replay of an old refresh revokes the grant.
Never retry a refresh concurrently. Revoke at `/oauth/revoke` or in the dashboard.

## Bots and tabs

Create a named bot, share its consent_url with intended recipients, then send
`{"user_id":123,"text":"Lesson starts in 15 minutes"}` with Idempotency-Key.
A username, public profile or previous notification is not consent. Limits: 10
messages/min per owner, 20/day per bot/recipient. Delivery is an account
notification, not a Telegram message or private chat.

Propose `{"title":"My notes","url":"https://example.com/notes"}` at
`/dashboard/tabs`. Wait for state=approved; external content opens separately.
