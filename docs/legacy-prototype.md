# Legacy prototype map (TwoBeWed)

Reference only. Do not extend this stack for the new platform.

## Runtime

| Piece | Path |
|---|---|
| Server entry | `server.js` |
| Express app | `config/app-config.js` |
| DB config | `config/db-config.js` (MongoDB, port 4000) |
| API mount | `/api/v1/` |

## APIs

| Area | Model | Routes |
|---|---|---|
| User | `app/apis/user/model/user.model.js` | signup, login, auth |
| Client | `app/apis/client/model/client.model.js` | client CRUD-style |
| Vendor | `app/apis/vendor/model/vendor.model.js` | vendor CRUD-style |
| Router aggregate | `app/apis/router/index.js` | mounts all three |

Auth: bcrypt password hashes + JWT (`user.controllers.js`). Email validation historically required `@twobewed.com`.

## Frontend

AngularJS app `TwoBeWed` in `app/client(frontend)/` with signup/login controllers; `index.html` hosts minimal forms. Bower for front-end libs; npm for server deps (`package.json` name: `two-be-wed`).

## Relevance to the new platform

Useful as a **domain reminder** (planners, clients-with-date, vendors, notes) and as a contrast for why schedule/event generalization and local-first sync are greenfield work.
