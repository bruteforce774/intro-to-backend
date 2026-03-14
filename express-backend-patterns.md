# Reusable Express Backend Patterns

Patterns drawn from a Node.js + Express + MongoDB + Mongoose tutorial project.

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

Routes define the URL shape. Controllers hold the logic. Models define the data shape. Config handles environment and connection concerns.

---

## Mounting Routers with a Prefix

```js
// app.js
app.use("/api/v1/users", userRouter);
app.use("/api/v1/posts", postRouter);
```

Add a new resource, add a new line. Route files only need to care about what comes *after* the prefix.

---

## Defining Routes and Mapping to Controllers

```js
// routes/user.route.js
router.route('/register').post(registerUser);
router.route('/login').post(loginUser);
router.route('/logout').post(logoutuser);

// routes/post.route.js
router.route('/create').post(createPost);
router.route('/getPosts').get(getPosts);
router.route('/update/:id').patch(updatePost);
router.route('/delete/:id').delete(deletePost);
```

The HTTP verb and controller function are the only two things that change per route.

---

## Basic Controller Shape

Every controller follows this skeleton: destructure from `req.body`, validate, do the database operation, return JSON.

```js
const createPost = async (req, res) => {
    try {
        const { name, description, age } = req.body;

        if (!name || !description || !age) {
            return res.status(400).json({
                message: "All fields are required"
            });
        }

        const post = await Post.create({ name, description, age });

        res.status(201).json({
            message: "Post created successfully", post
        });

    } catch (error) {
        res.status(500).json({
            message: "Internal Server error", error
        });
    }
}
```

---

## Using the URL Parameter with `:id`

```js
// route definition
router.route('/update/:id').patch(updatePost);

// controller — req.params.id comes from :id in the route
const post = await Post.findByIdAndUpdate(req.params.id, req.body, { new: true });
```

`{ new: true }` returns the updated document rather than the old one — easy to forget but important.

---

## Guarding Against an Empty Update Body

```js
if (Object.keys(req.body).length === 0) {
    return res.status(400).json({
        message: "No data provided for update"
    });
}
```

Reusable guard for any PATCH route.

---

## Mongoose Schema with Constraints

```js
const userSchema = new Schema({
    username: {
        type: String,
        required: true,
        unique: true,
        lowercase: true,
        trim: true,
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

`timestamps: true` automatically adds `createdAt` and `updatedAt` to every document — you almost always want this.

---

## Hashing a Password Before Saving

```js
userSchema.pre("save", async function (next) {
    if (!this.isModified("password")) return next();
    this.password = await bcrypt.hash(this.password, 10);
    next();
});
```

The `pre("save")` hook runs automatically before any save. The `isModified` check is critical — without it the password gets re-hashed every time you save the document for any reason, corrupting it.

---

## Adding a Method to a Schema

```js
// model definition
userSchema.methods.comparePassword = async function (password) {
    return await bcrypt.compare(password, this.password)
}

// called in controller as:
const isMatch = await user.comparePassword(password);
```

Attaches reusable logic directly to a model instance rather than scattering it across controllers.

---

## Duplicate Entry Check Before Creating

```js
const existing = await User.findOne({ email: email.toLowerCase() });
if (existing) {
    return res.status(400).json({ message: "User already exists!" });
}
```

Standard pattern before any `Model.create()` where a field must be unique.

---

## Database Connection with Graceful Failure

```js
// config/constants.js
export const DB_NAME = "intro-to-backend"

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

`process.exit(1)` stops the server from running in a broken state if the database connection fails. Keeping `DB_NAME` in a constants file lets you swap databases without touching the connection logic.

---

## Wiring It All Together in the Entry Point

```js
// index.js
const startServer = async () => {
    await connectDB();
    app.listen(process.env.PORT || 8000, () => {
        console.log(`Server is running at port: ${process.env.PORT}`);
    });
}

startServer();
```

The database connection is awaited before the server starts listening. The `|| 8000` fallback means the app still runs if `PORT` is missing from `.env`.
