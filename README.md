# AI BlogNest API — Documentation

A RESTful backend for a blogging platform with built-in AI content generation. Built with **Node.js, Express, MongoDB (Mongoose), JWT authentication, bcrypt** and **Google Gemini**.

> This documentation was written from the source files in `Code_Files.zip`. The `.env` file was intentionally not read.

---

## 1. Overview

| Item | Value |
|---|---|
| Package name | `ai-blognest-api` v1.0.0 |
| Entry point | `src/server.js` |
| Module system | CommonJS |
| Architecture | MVC (routes → controllers → models) plus a service layer for Gemini |
| Default port | `5000` (override with `PORT`) |
| Auth | JWT Bearer token, 7-day expiry |

**Features**

- User registration and login with hashed passwords
- JWT-protected routes
- Blog CRUD with author-only edit/delete
- AI blog generation (saved directly as a blog post)
- AI summarization of any text

---

## 2. Tech Stack

| Dependency | Purpose |
|---|---|
| `express` | HTTP server and routing |
| `mongoose` | MongoDB ODM |
| `jsonwebtoken` | Create/verify JWTs |
| `bcryptjs` | Password hashing |
| `express-validator` | Request validation |
| `@google/generative-ai` | Gemini client |
| `cors` | Cross-origin requests (open to all origins) |
| `morgan` | HTTP request logging (`dev` format) |
| `dotenv` | Loads environment variables |
| `axios` | Installed but not used in the source |
| `nodemon` (dev) | Auto-restart during development |

---

## 3. Project Structure

```
Code Files/
├── package.json
├── README.md
├── thunder-client-ai-blognest-api.postman_collection.json   # Ready-made API requests
├── thunder-client-env.json                                  # Thunder Client environment
└── src/
    ├── server.js                  # Loads env, connects DB, starts server
    ├── app.js                     # Express app, middleware, route mounting
    ├── config/
    │   └── db.js                  # MongoDB connection
    ├── models/
    │   ├── User.js
    │   └── Blog.js
    ├── routes/
    │   ├── authRoutes.js          # /api/auth
    │   ├── blogRoutes.js          # /api/blogs
    │   └── aiRoutes.js            # /api/ai
    ├── controllers/
    │   ├── authController.js
    │   ├── blogController.js
    │   └── aiController.js
    ├── middleware/
    │   ├── authMiddleware.js      # JWT verification
    │   └── errorMiddleware.js     # Central error handler
    └── services/
        └── geminiService.js       # Gemini API wrapper
```

---

## 4. Getting Started

**Prerequisites:** Node.js, a MongoDB instance (local or Atlas), and a Gemini API key.

```bash
# 1. Install dependencies
npm install

# 2. Create a .env file in the project root (see section 5)

# 3. Run in development (auto-reload)
npm run dev

# ...or run in production mode
npm start
```

| Script | Command |
|---|---|
| `npm start` | `node src/server.js` |
| `npm run dev` | `nodemon src/server.js` |

Health check: `GET /` returns `{ "message": "AI BlogNest API is running" }`.

---

## 5. Environment Variables

Create a `.env` file in the project root. `.env` is already listed in `.gitignore`, so it won't be committed.

| Variable | Required | Description |
|---|---|---|
| `PORT` | No | Server port. Defaults to `5000`. |
| `MONGO_URI` | **Yes** | MongoDB connection string. The app exits on startup if it is missing. |
| `JWT_SECRET` | **Yes** | Secret used to sign and verify tokens. |
| `GEMINI_API_KEY` | **Yes** (for AI routes) | Google Gemini API key. AI routes throw an error if missing. |
| `GEMINI_MODEL` | No | Gemini model name. Falls back to `gemini-1.5-flash`. The value `gemini-1.5` is also mapped to `gemini-1.5-flash`. |

> The README mentions copying a `.env.example`, but no such file is included in the archive.

---

## 6. Data Models

### User

| Field | Type | Rules |
|---|---|---|
| `name` | String | Required, trimmed |
| `email` | String | Required, **unique**, lowercased, trimmed |
| `password` | String | Required, min length 6 (stored as a bcrypt hash) |
| `createdAt` / `updatedAt` | Date | Automatic timestamps |

### Blog

| Field | Type | Rules |
|---|---|---|
| `title` | String | Required, trimmed |
| `content` | String | Required |
| `category` | String | Required, trimmed |
| `author` | ObjectId → `User` | Required |
| `authorName` | String | Required, trimmed (denormalized copy of the author's name) |
| `createdAt` / `updatedAt` | Date | Automatic timestamps |

---

## 7. Authentication

Protected routes require this header:

```
Authorization: Bearer <token>
```

The middleware (`authMiddleware.js`):

1. Checks for a `Bearer` header, otherwise returns `401 Authorization token missing`.
2. Verifies the token with `JWT_SECRET`.
3. Loads the user (without the password) and attaches it to `req.user`.
4. Returns `401` if the user no longer exists or the token is invalid/expired.

Tokens are signed with payload `{ id: <userId> }` and expire after **7 days**.

---

## 8. API Reference

Base URL: `http://localhost:5000` (or your `PORT`)

### Summary

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/auth/register` | No | Create an account |
| POST | `/api/auth/login` | No | Get a JWT |
| GET | `/api/auth/profile` | Yes | Current user's profile |
| POST | `/api/blogs` | Yes | Create a blog |
| GET | `/api/blogs` | No | List all blogs |
| GET | `/api/blogs/:id` | No | Get one blog |
| PUT | `/api/blogs/:id` | Yes (author) | Update a blog |
| DELETE | `/api/blogs/:id` | Yes (author) | Delete a blog |
| POST | `/api/ai/generate-blog` | Yes | Generate and save an AI blog |
| POST | `/api/ai/summarize` | **No** | Summarize text |

---

### 8.1 Auth

#### `POST /api/auth/register`

**Body**

```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "password": "123456"
}
```

Validation: `name` not empty, `email` valid, `password` at least 6 characters.

**Responses**

- `201 Created`
  ```json
  {
    "message": "User registered successfully",
    "user": { "id": "…", "name": "John Doe", "email": "john@example.com" }
  }
  ```
- `400` — `{ "message": "Email already registered" }`
- `422` — `{ "errors": [ … ] }` (validation errors)

#### `POST /api/auth/login`

**Body**

```json
{ "email": "john@example.com", "password": "123456" }
```

**Responses**

- `200 OK` — `{ "token": "<jwt>" }`
- `401` — `{ "message": "Invalid credentials" }` (same message for unknown email or wrong password)
- `422` — validation errors

#### `GET /api/auth/profile`  🔒

**Response `200`**

```json
{ "id": "…", "name": "John Doe", "email": "john@example.com" }
```

---

### 8.2 Blogs

#### `POST /api/blogs`  🔒

**Body**

```json
{
  "title": "My First AI Blog",
  "content": "This is a sample AI-generated blog post content.",
  "category": "Technology"
}
```

All three fields are required. `author` and `authorName` are set from the logged-in user.

**Responses:** `201` with the created blog, `422` on validation errors.

#### `GET /api/blogs`

Returns all blogs, newest first. `author` is populated with `name` and `email`.

#### `GET /api/blogs/:id`

Returns one blog (with populated author). `404 { "message": "Blog not found" }` if missing.

#### `PUT /api/blogs/:id`  🔒

**Body** (all fields optional, but must not be empty if sent)

```json
{
  "title": "Updated AI Blog Title",
  "content": "Updated content for the blog post.",
  "category": "Education"
}
```

**Responses**

- `200` — updated blog
- `403` — `{ "message": "You are not allowed to edit this blog" }` (not the author)
- `404` — blog not found
- `422` — validation errors

#### `DELETE /api/blogs/:id`  🔒

**Responses**

- `200` — `{ "message": "Blog deleted successfully" }`
- `403` — `{ "message": "You are not allowed to delete this blog" }`
- `404` — blog not found

---

### 8.3 AI

#### `POST /api/ai/generate-blog`  🔒

Generates a short blog post with Gemini and **saves it** to the database.

**Body**

```json
{ "topic": "Artificial Intelligence", "category": "Technology" }
```

`topic` is required; `category` is optional.

**How it works**

1. Sends Gemini a prompt asking for a blog post (introduction, key points, conclusion) in **100–150 words** with no markdown formatting.
2. Cleans the response (see *Text cleaning* below).
3. Sets `title` to `"Guide to <topic>"`.
4. Uses the provided `category`, or infers one from the topic (see *Category inference*).
5. Saves the blog with the current user as author and returns it.

**Response `201`** — the created blog object.

#### `POST /api/ai/summarize`

**Body**

```json
{ "content": "This is a long blog post content that needs a short summary." }
```

**Response `200`**

```json
{ "summary": "…" }
```

This endpoint does not save anything.

---

### 8.4 Error responses

| Status | Meaning | Shape |
|---|---|---|
| `401` | Missing/invalid/expired token | `{ "message": "…" }` |
| `403` | Not the blog's author | `{ "message": "…" }` |
| `404` | Blog not found | `{ "message": "Blog not found" }` |
| `422` | Validation failed | `{ "errors": [ { "msg": "…", "path": "…" } ] }` |
| `500` | Unhandled error | `{ "message": "…" }` |

Unhandled errors go to `errorMiddleware.js`, which logs the stack trace and responds with `err.status` (default `500`) and `err.message`.

---

## 9. Internal Logic Worth Knowing

### Text cleaning (`cleanGeneratedText`)

Applied to all Gemini output. It removes markdown headings (`#`), bullet markers (`* - +`), bold/italic asterisks, normalizes line endings, collapses 3+ blank lines to 2, and trims whitespace.

### Category inference (`inferCategory`)

When `category` is not provided, the topic is lowercased and checked for keywords. The first matching group wins:

| Category | Keywords |
|---|---|
| Technology | ai, technology, software, programming, cloud |
| Education | learning, school, education, teaching, study |
| Health | health, fitness, medicine, wellness, mental |
| Sports | sports, football, basketball, soccer, athlete |
| Lifestyle | lifestyle, fashion, travel, home, beauty |
| Business | business, startup, finance, economy, marketing |
| Travel | travel, tourism, destination, adventure, holiday |
| Finance | money, investment, stocks, banking, crypto |
| Entertainment | entertainment, movies, music, celebrity, gaming |
| *(no match)* | `Unidentified` |

### Gemini service (`callGemini`)

Reads `GEMINI_API_KEY` and `GEMINI_MODEL`, creates a `GoogleGenerativeAI` client, calls `generateContent(prompt)`, and returns the text. It throws if the key is missing or the response is empty.

### Startup flow

`server.js` → `dotenv.config()` → `connectDB()` (exits the process if `MONGO_URI` is missing or the connection fails) → `app.listen(PORT)`.

---

## 10. Testing with Thunder Client / Postman

The archive includes `thunder-client-ai-blognest-api.postman_collection.json` (collection name **AI BlogNest API**) and `thunder-client-env.json`. The collection uses these variables:

- `{{base_url}}` — e.g. `http://localhost:5000`
- `{{blogId}}` — an existing blog's `_id`

**Suggested test order**

1. Register User
2. Login User → copy the token and set it as a Bearer token on protected requests
3. Get Profile
4. Create Blog → copy the returned `_id` into `blogId`
5. Get All Blogs / Get Blog By ID
6. Update Blog / Delete Blog
7. Generate AI Blog / Summarize Blog Content

---

## 11. Observations and Suggested Improvements

These came up while reading the code. They're not part of the documented behavior, but worth knowing about.

1. **`blog.remove()` will fail on Mongoose 7.** `Document.prototype.remove()` was removed in Mongoose 7 (the project uses `^7.5.0`), so `DELETE /api/blogs/:id` will likely throw. Use `await blog.deleteOne()` instead.
2. **`/api/ai/summarize` is unauthenticated.** Unlike `generate-blog`, it has no `authMiddleware`, so anyone can spend your Gemini quota. Add auth and/or rate limiting if that's not intended.
3. **Open CORS.** `app.use(cors())` allows every origin. Restrict it for production.
4. **No rate limiting or security headers.** Consider `express-rate-limit` (especially on login and AI routes) and `helmet`.
5. **Deprecated Mongoose options.** `useNewUrlParser` and `useUnifiedTopology` in `db.js` have no effect in current drivers and can be removed.
6. **Keyword matching is substring-based.** For example, `ai` matches inside many unrelated words, so `inferCategory` can misclassify topics. Whole-word matching would be more accurate. `travel` also appears under both Lifestyle and Travel, so Lifestyle always wins.
7. **Model default.** The fallback model is `gemini-1.5-flash`. Check that it's still available in your Gemini account and set `GEMINI_MODEL` explicitly if not.
8. **Unused dependency.** `axios` is installed but not imported anywhere.
9. **Missing `.env.example`.** The README refers to one; adding it would help new contributors.
10. **`authorName` is a copy.** If a user's name ever changes, existing blogs keep the old name. The populated `author` field stays current.
11. **Handler order in `app.js`.** The error middleware is registered before the `GET /` route. It works, but conventionally the error handler goes last.
