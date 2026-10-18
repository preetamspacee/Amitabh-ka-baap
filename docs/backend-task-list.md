# DSA Buddy Backend Task List

Last updated Oct 18, 2026. Editable copy: https://claude.ai/code/artifact/9a77ff26-f073-48e9-a272-e86c564f7238

## Where it stands

Nothing is broken. The backend is simply missing. Today three jobs happen in the browser, and each one has to move to a server so that data is shared and survives across devices.

| Job | Today (in the browser) | After (on the backend) |
| --- | --- | --- |
| Problem list | Hard-coded in `data/problems.js` | Stored in a `problems` table, served by `GET /api/problems` |
| Solved progress and streak | `localStorage`, one browser only | `user_problem_status` table, per user, on any device |
| Daily challenge | Picked from today's date in `app.js` | Chosen by the server and saved in `daily_challenges` |
| Chat commands | Answered by `app.js` | Answered by `POST /api/chat` (and later by WhatsApp) |

The plan builds a normal web API first. WhatsApp comes last, because it adds a third-party account, approvals and costs on top of a backend that must already work.

## Tech choices

The stack below follows the README's own plan (Node.js, Express, PostgreSQL) and adds Prisma to make the database easier for a beginner.

| Piece | Choice | Why |
| --- | --- | --- |
| Language and runtime | Node.js with JavaScript | The frontend is already JavaScript, so there's one language to learn |
| Web framework | Express | Small, widely taught, and named in the README |
| Database | PostgreSQL | Named in the README; free hosted options exist |
| Database tool | Prisma | Describes tables in one file, creates them for you, and gives safe queries |
| Login | Email + password with bcrypt hashing, then a JWT token | Simple enough to build yourself and learn how auth works |
| Tests | Vitest + Supertest | Call your API in tests without a browser |
| Hosting | Render (server) + Neon or Supabase (database) | Free tiers, deploy straight from GitHub |

## Database design

Four tables are enough for the first version. The README also suggests `message_events` for WhatsApp, which waits until phase 7.

| Table | Columns | Notes |
| --- | --- | --- |
| `users` | id, email (unique), password\_hash, name, timezone, created\_at | Never store the plain password |
| `problems` | id, slug (unique), title, topic, difficulty, summary, goal, hint, source\_url, video\_url, tags, is\_active | Seeded from today's 8 problems in `data/problems.js` |
| `user_problem_status` | user\_id, problem\_id, status, solved\_at | One row per user per problem; the pair is unique |
| `daily_challenges` | date (unique), problem\_id | One problem per day, the same for everyone |

The streak is not stored. The server works it out from `solved_at` dates when asked, exactly as `app.js` does now.

## Task list

Work through the phases in order. Each phase ends with something you can run and check before moving on.

### Phase 1: Set up the server

- [ ] Create a `server/` folder with `npm init`, then install `express`, `cors` and `dotenv`
- [ ] Write `server/index.js` with one route, `GET /api/health`, that returns `{ "ok": true }`
- [ ] Add a `.env` file for settings such as `PORT` and `DATABASE_URL` (already ignored by `.gitignore`)
- [ ] Add `npm run dev` with `nodemon` so the server restarts when you save
- [ ] Check it: open `http://localhost:3000/api/health` in the browser

### Phase 2: Add the database

- [ ] Create a free PostgreSQL database on Neon or Supabase, or run one locally with Docker
- [ ] Install Prisma and write the four tables in `prisma/schema.prisma`
- [ ] Run `npx prisma migrate dev` to create the tables
- [ ] Write a seed script that copies the 8 problems from `data/problems.js` into `problems`
- [ ] Check it: open `npx prisma studio` and see the 8 rows

### Phase 3: Problem and daily endpoints

- [ ] `GET /api/problems` with optional `?topic=`, `?difficulty=` and `?search=` filters
- [ ] `GET /api/problems/:id` for one problem
- [ ] `GET /api/problems/:id/hint` for its hint
- [ ] `GET /api/daily` that picks today's problem and saves it in `daily_challenges` if it isn't there yet
- [ ] Return clear errors: 404 for an unknown problem, 400 for a bad filter

### Phase 4: Users and progress

- [ ] `POST /api/auth/register` and `POST /api/auth/login`, hashing passwords with bcrypt and returning a JWT
- [ ] Middleware that reads the token and rejects requests without a valid one
- [ ] `POST /api/progress/:problemId` to mark solved and `DELETE` to unmark
- [ ] `GET /api/progress` returning solved count, streak, topic coverage and recent activity
- [ ] Move the streak calculation from `app.js` into the server

### Phase 5: Chat command endpoint

- [ ] `POST /api/chat` taking `{ "text": "solve 1" }` and returning the bot's reply
- [ ] Support the same commands as today: `daily`, `random`, `topics`, `search`, `solve`, `progress`, `help`
- [ ] Keep command handling in its own file so WhatsApp can reuse it later

### Phase 6: Quality and deploy

- [ ] Validate request bodies (for example with `zod`) and add a rate limit to login
- [ ] Write tests for each endpoint with Vitest + Supertest
- [ ] Deploy the server to Render with `DATABASE_URL` and `JWT_SECRET` set as environment variables
- [ ] Add a GitHub Actions workflow that runs the tests on every push

### Phase 7: WhatsApp (later)

- [ ] Set up a WhatsApp Cloud API test number in Meta's developer console
- [ ] Add `POST /webhook/whatsapp`, verify Meta's signature, and pass the text to the phase 5 command handler
- [ ] Add a scheduler for daily messages, plus `STOP` handling and the `message_events` table

## API endpoints

The first version is 12 endpoints, plus the WhatsApp webhook later. The ones marked "yes" need a logged-in user.

| Method | Path | Login | Does |
| --- | --- | --- | --- |
| GET | `/api/health` | no | Says the server is up |
| POST | `/api/auth/register` | no | Creates a user |
| POST | `/api/auth/login` | no | Returns a token |
| GET | `/api/me` | yes | Returns the current user |
| GET | `/api/problems` | no | Lists problems, with filters |
| GET | `/api/problems/:id` | no | One problem |
| GET | `/api/problems/:id/hint` | no | Its hint |
| GET | `/api/daily` | no | Today's challenge |
| GET | `/api/progress` | yes | Solved count, streak, topics, recent activity |
| POST | `/api/progress/:problemId` | yes | Marks a problem solved |
| DELETE | `/api/progress/:problemId` | yes | Unmarks it |
| POST | `/api/chat` | optional | Answers a chat command |
| POST | `/webhook/whatsapp` | Meta signature | Phase 7 only |

## Frontend changes

Once the API works, the existing site needs a few edits so it reads from the server instead of the browser.

- [ ] Replace `window.DSA_PROBLEMS` in `app.js` with a `fetch('/api/problems')` call, and delete `data/problems.js`
- [ ] Add a simple login and sign-up form, and keep the token after login
- [ ] Send solve and unsolve clicks to `/api/progress` instead of `localStorage`
- [ ] Send typed chat messages to `/api/chat` and show the reply it returns
- [ ] Show a loading state and a friendly error when the server can't be reached
- [ ] Serve the site from Express, or host it on GitHub Pages and allow it in CORS
