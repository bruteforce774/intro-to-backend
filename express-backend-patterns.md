# Reusable Express Backend Patterns

Patterns drawn from a Node.js + Express + MongoDB + Mongoose tutorial project. Written for someone who has seen Express before but is new to building a full backend with Mongoose and a structured project layout.

---

## Project Structure

```
backend/src/
├── app.js
├── index.js
├── config/
│   ├── database.js
│   └── constants.js
├── models/
│   ├── user.model.js
│   └── post.model.js
├── controllers/
│   ├── user.controller.js
│   └── post.controller.js
└── routes/
    ├── user.route.js
    └── post.route.js
```

This separation is the backbone of any maintainable Express app. Each folder has a clear responsibility:

- **config/** — anything that sets up the environment: database connection, constants, environment variables
- **models/** — the shape of your data and any logic tied directly to that data (e.g. password hashing)
- **controllers/** — the actual work each endpoint does: read the request, talk to the database, send a response
- **routes/** — just the wiring: which URL maps to which controller function

The benefit is that when something breaks or needs changing, you know exactly which folder to look in.

---

## Mounting Routers with a Prefix

```js
// app.js
app.use("/api/v1/users", userRouter);
app.use("/api/v1/posts", postRouter);
```

In Express, `app.use()` applies a piece of middleware to all requests that match a given path. Here we're using it to attach entire routers — not just single routes — under a common prefix.

This means every route defined in `userRouter` automatically lives under `/api/v1/users`. So a route defined as `/register` in that router becomes `/api/v1/users/register` in the running app. The route file itself never needs to know about the prefix.

The `/v1/` in the path is a common convention for API versioning. If you ever make breaking changes to your API, you can add a `v2` router alongside `v1` without disrupting existing clients.

---

## Defining Routes and Mapping to Controllers

```js
// routes/user.route.js
import { Router } from 'express';
import { loginUser, logoutuser, registerUser } from '../controllers/user.controller.js';

const router = Router();

router.route('/register').post(registerUser);
router.route('/login').post(loginUser);
router.route('/logout').post(logoutuser);

export default router;
```

`Router()` creates a mini Express app that only handles routing. You define paths on it, attach HTTP verbs (`.get()`, `.post()`, `.patch()`, `.delete()`), and point each one at a controller function.

The controller functions are imported from a separate file — the route file's only job is this mapping. It contains no logic of its own. This is important because it means you can read the route file and immediately understand the shape of the API without wading through implementation details.

Note that routes using a URL parameter look like this:

```js
// routes/post.route.js
router.route('/update/:id').patch(updatePost);
router.route('/delete/:id').delete(deletePost);
```

The `:id` is a placeholder. Whatever appears in that position in the actual URL (e.g. `/update/64a7f3c2e1b2a3d4e5f6a7b8`) becomes available in the controller as `req.params.id`.

---

## Basic Controller Shape

Every controller in this project follows the same skeleton. Once you recognize it, every controller becomes easy to read and write.

```js
const createPost = async (req, res) => {
    try {
        // 1. Pull data from the request body
        const { name, description, age } = req.body;

        // 2. Validate — return early with a 400 if something is wrong
        if (!name || !description || !age) {
            return res.status(400).json({
                message: "All fields are required"
            });
        }

        // 3. Do the database operation
        const post = await Post.create({ name, description, age });

        // 4. Send a success response
        res.status(201).json({
            message: "Post created successfully", post
        });

    } catch (error) {
        // 5. Catch anything unexpected and return a 500
        res.status(500).json({
            message: "Internal Server error", error
        });
    }
}
```

A few things worth noting:

- Controllers are `async` functions because database operations return Promises. The `await` keyword pauses execution until the Promise resolves, giving you the result directly rather than chaining `.then()` calls.
- The `return` on the validation response is important. Without it, execution would continue past the if-block and try to run the database operation anyway, likely causing an error.
- HTTP status codes carry meaning: `400` means the client sent bad data, `201` means a resource was successfully created, `500` means something went wrong on the server.
- The `catch` block is a safety net. Even if you think your code is correct, unexpected things happen — network issues, malformed data that slipped past validation, database errors. Always have a catch.

---

## Using the URL Parameter with `:id`

```js
// route definition
router.route('/update/:id').patch(updatePost);

// inside the controller
const post = await Post.findByIdAndUpdate(req.params.id, req.body, { new: true });
```

`req.params.id` is how you access the `:id` placeholder from the URL. If the request came in as `PATCH /api/v1/posts/update/64a7f3c2e1b2a3d4e5f6a7b8`, then `req.params.id` equals `64a7f3c2e1b2a3d4e5f6a7b8`.

`findByIdAndUpdate` is a Mongoose helper that wraps MongoDB's `findOneAndUpdate`. It takes three arguments: the id to look up, the update data, and an options object. The `{ new: true }` option tells Mongoose to return the document *after* the update is applied. Without it, you get back the old version of the document, which is almost never what you want.

---

## Guarding Against an Empty Update Body

```js
if (Object.keys(req.body).length === 0) {
    return res.status(400).json({
        message: "No data provided for update"
    });
}
```

`req.body` is the parsed JSON from the request. If the client sends an empty object `{}`, `Object.keys()` returns an empty array, and `.length === 0` catches it.

This matters for PATCH routes in particular. A PATCH request is meant to partially update a resource, so it's valid to update just one field. But updating zero fields is never meaningful. Without this guard, an empty request would silently succeed — Mongoose would run the update with no changes and return the unmodified document with a 200 OK, which is misleading.

---

## Mongoose Schema with Constraints

A Mongoose schema defines the shape of documents in a MongoDB collection — what fields exist, what types they are, and what rules they must follow.

```js
import mongoose, { Schema } from "mongoose";

const userSchema = new Schema({
    username: {
        type: String,
        required: true,   // field must be present
        unique: true,     // no two documents can share this value
        lowercase: true,  // automatically converts to lowercase before saving
        trim: true,       // automatically strips leading/trailing whitespace
        minLength: 1,
        maxLength: 30
    },
    password: {
        type: String,
        required: true,
        minLength: 6,
        maxLength: 50
    },
    email: {
        type: String,
        required: true,
        unique: true,
        lowercase: true,
        trim: true
    }
}, { timestamps: true })
```

The second argument to `new Schema()` is an options object. `{ timestamps: true }` tells Mongoose to automatically add two fields to every document: `createdAt` and `updatedAt`. Mongoose manages these for you — `createdAt` is set when the document is first saved, and `updatedAt` is updated every time the document changes. You almost always want this.

Note that `unique: true` in a Mongoose schema creates a unique index in MongoDB. It is a database-level constraint, not just a validation rule. This means MongoDB will reject duplicate values even if you bypass Mongoose entirely.

---

## Hashing a Password Before Saving

Passwords must never be stored as plain text. This project uses `bcrypt`, a one-way hashing algorithm designed specifically for passwords.

```js
import bcrypt from "bcrypt";

userSchema.pre("save", async function (next) {
    if (!this.isModified("password")) return next();
    this.password = await bcrypt.hash(this.password, 10);
    next();
});
```

`pre("save")` is a Mongoose **middleware hook** — a function that runs automatically at a specific point in a document's lifecycle, in this case just before it is saved to the database. You attach it to the schema, and Mongoose handles calling it.

Inside the hook, `this` refers to the document being saved. The `isModified("password")` check is critical: it asks whether the password field has changed since the document was last loaded. If it hasn't — say you're updating the user's email — the hook calls `next()` immediately and skips re-hashing. Without this check, the already-hashed password would get hashed again on every save, making it impossible to verify later.

The `10` in `bcrypt.hash()` is the number of salt rounds — how many times the hashing algorithm iterates. Higher numbers are more secure but slower. 10 is the standard default.

---

## Adding a Method to a Schema

Just as you can attach hooks to a schema, you can attach custom methods. These become available on any document instance created from that model.

```js
// model definition
userSchema.methods.comparePassword = async function (password) {
    return await bcrypt.compare(password, this.password)
}
```

```js
// called in the login controller
const user = await User.findOne({ email: email.toLowerCase() });
const isMatch = await user.comparePassword(password);
```

`bcrypt.compare()` takes a plain-text password and a hash, and returns `true` or `false`. Because the hash was created with a salt, you can't simply hash the input again and compare strings — you have to use bcrypt's own compare function, which knows how to handle the salt.

Putting this logic in a schema method rather than the controller keeps the controller clean and makes the behavior reusable. Any controller that needs to verify a password just calls `user.comparePassword()` without needing to know anything about bcrypt.

---

## Duplicate Entry Check Before Creating

```js
const existing = await User.findOne({ email: email.toLowerCase() });
if (existing) {
    return res.status(400).json({ message: "User already exists!" });
}
```

`Model.findOne()` queries the database for a single document matching the given conditions. It returns the document if found, or `null` if not. Here we use it to check for a duplicate email before creating a new user.

The `.toLowerCase()` call normalizes the input so that `Test@Example.com` and `test@example.com` are treated as the same address. The `lowercase: true` option in the schema does the same normalization on save, but you need to normalize before the lookup too — otherwise `findOne` might miss an existing user whose email was stored in lowercase.

Even though the schema has `unique: true` on the email field, doing this check explicitly gives you a clean, predictable error message. Without it, a duplicate would throw a raw MongoDB error that you'd have to parse and translate yourself.

---

## Database Connection with Graceful Failure

```js
// config/constants.js
export const DB_NAME = "intro-to-backend"
```

```js
// config/database.js
import mongoose from "mongoose";
import { DB_NAME } from "./constants.js";

const connectDB = async () => {
    try {
        const connectionInstance = await mongoose.connect(`${process.env.MONGODB_URI}/${DB_NAME}`)
        console.log(`MongoDB connected !!! ${connectionInstance.connection.host}`);
    } catch (error) {
        console.log("MongoDB connection failed", error);
        process.exit(1)
    }
}

export default connectDB;
```

The MongoDB connection URI contains the host, credentials, and optionally the database name. By keeping the base URI in `.env` and appending the database name from `constants.js`, you get a clean separation: the secret (the URI with credentials) lives in the environment and is never committed to source control, while the non-secret (the database name) lives in the code where it belongs.

`process.exit(1)` is Node's way of stopping the process with an error code. The `1` signals failure (as opposed to `0`, which signals a clean exit). This is important — if the database connection fails, the server has no reason to keep running. Without this line, Express would start up and begin accepting requests it can't actually fulfill, which is a confusing failure mode.

`connectionInstance.connection.host` lets you confirm in the logs exactly which database server you connected to — useful when switching between a local development instance and a hosted cloud cluster.

---

## Wiring It All Together in the Entry Point

```js
// index.js
import dotenv from "dotenv";
import connectDB from "./config/database.js";
import app from "./app.js";

dotenv.config({ path: './.env' });

const startServer = async () => {
    await connectDB();
    app.listen(process.env.PORT || 8000, () => {
        console.log(`Server is running at port: ${process.env.PORT}`);
    });
}

startServer();
```

`dotenv.config()` reads your `.env` file and loads its contents into `process.env`. This must happen before anything else that reads environment variables — which is why it's at the top of the entry point.

The database connection is `await`ed before `app.listen()` is called. This ensures the server only starts accepting requests once it has a working database connection. If you called them in parallel or reversed the order, requests could arrive before the database is ready.

The `|| 8000` fallback on the port means the app still runs if `PORT` is missing from `.env`, defaulting to port 8000. This is a small but useful defensive pattern.
