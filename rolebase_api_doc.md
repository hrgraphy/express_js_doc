# 🚀 Role-Based Full CRUD API
## Express.js + MongoDB + JWT + MVC Architecture

---

# 📌 Project Overview

This project implements:

- JWT Authentication
- Role-Based Authorization (Admin & User)
- Full CRUD Operations
- Owner-Based Update Restriction
- Password Hashing
- Environment Variables
- Proper Status Codes
- MVC Architecture
- Validation & Error Handling

---

# 📦 1️⃣ Project Initialization

## Step 1: Create Project

```bash
mkdir role-based-crud-api
cd role-based-crud-api
npm init -y
```

---

## Step 2: Install Dependencies

```bash
npm install express mongoose bcryptjs jsonwebtoken cors dotenv
npm install nodemon --save-dev
```

---

## Step 3: Update package.json

```json
"scripts": {
  "start": "node server.js",
  "dev": "nodemon server.js"
}
```

Run server:

```bash
npm run dev
```

---

# 📁 2️⃣ Folder Structure

```
role-based-crud-api/
│
├── models/
│   └── User.js
│
├── controllers/
│   └── userController.js
│
├── routes/
│   └── userRoutes.js
│
├── middleware/
│   ├── authMiddleware.js
│   └── roleMiddleware.js
│
├── .env
├── server.js
└── package.json
```

---

# ⚙️ 3️⃣ Environment Variables

Create `.env`

```
MONGO_URL=mongodb://127.0.0.1:27017/roleCrudDB
JWT_SECRET=supersecretkey
PORT=3000
```

---

# 🖥 4️⃣ server.js

```js
const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
require("dotenv").config();

const userRoutes = require("./routes/userRoutes");

const app = express();

app.use(cors());
app.use(express.json());

app.use("/api/users", userRoutes);

mongoose.connect(process.env.MONGO_URL)
  .then(() => console.log("MongoDB Connected"))
  .catch(err => console.log(err));

app.listen(process.env.PORT, () => {
  console.log(`Server running on port ${process.env.PORT}`);
});
```

---

# 🗄 5️⃣ User Model

📁 models/User.js

```js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, "Name is required"],
    trim: true
  },
  email: {
    type: String,
    required: [true, "Email is required"],
    unique: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6,
    select: false
  },
  role: {
    type: String,
    enum: ["admin", "user"],
    default: "user"
  }
}, { timestamps: true });

module.exports = mongoose.model("User", userSchema);
```

---

# 🎮 6️⃣ Controller (FULL CRUD + ROLE LOGIC)

📁 controllers/userController.js

```js
const User = require("../models/User");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");

// REGISTER
exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    const existingUser = await User.findOne({ email });
    if (existingUser)
      return res.status(400).json({ message: "Email already exists" });

    const hashedPassword = await bcrypt.hash(password, 10);

    const user = await User.create({
      name,
      email,
      password: hashedPassword,
      role
    });

    res.status(201).json({
      message: "User Registered Successfully",
      userId: user._id
    });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// LOGIN
exports.login = async (req, res) => {
  try {
    const user = await User.findOne({ email: req.body.email }).select("+password");
    if (!user)
      return res.status(404).json({ message: "User not found" });

    const isMatch = await bcrypt.compare(req.body.password, user.password);
    if (!isMatch)
      return res.status(400).json({ message: "Invalid password" });

    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: "1h" }
    );

    res.status(200).json({ token });

  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// CREATE USER (Admin Only)
exports.createUser = async (req, res) => {
  const user = await User.create(req.body);
  res.status(201).json(user);
};

// GET ALL USERS (Admin Only)
exports.getAllUsers = async (req, res) => {
  const users = await User.find().select("-password");
  res.status(200).json(users);
};

// GET SINGLE USER
exports.getUserById = async (req, res) => {
  const user = await User.findById(req.params.id).select("-password");
  if (!user)
    return res.status(404).json({ message: "User not found" });

  res.status(200).json(user);
};

// UPDATE USER (Admin or Owner)
exports.updateUser = async (req, res) => {
  if (req.user.role !== "admin" && req.user.id !== req.params.id) {
    return res.status(403).json({ message: "Access Denied" });
  }

  const updatedUser = await User.findByIdAndUpdate(
    req.params.id,
    req.body,
    { new: true }
  );

  res.status(200).json(updatedUser);
};

// DELETE USER (Admin Only)
exports.deleteUser = async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  res.status(200).json({ message: "User Deleted Successfully" });
};
```

---

# 🔐 7️⃣ Authentication Middleware

📁 middleware/authMiddleware.js

```js
const jwt = require("jsonwebtoken");

exports.authenticate = (req, res, next) => {
  const token = req.headers.authorization;

  if (!token)
    return res.status(401).json({ message: "Unauthorized" });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch {
    res.status(400).json({ message: "Invalid Token" });
  }
};
```

---

# 🛡 8️⃣ Role Middleware

📁 middleware/roleMiddleware.js

```js
exports.authorize = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: "Forbidden" });
    }
    next();
  };
};
```

---

# 🛣 9️⃣ Routes

📁 routes/userRoutes.js

```js
const express = require("express");
const router = express.Router();

const userController = require("../controllers/userController");
const { authenticate } = require("../middleware/authMiddleware");
const { authorize } = require("../middleware/roleMiddleware");

router.post("/register", userController.register);
router.post("/login", userController.login);

router.post("/", authenticate, authorize("admin"), userController.createUser);
router.get("/", authenticate, authorize("admin"), userController.getAllUsers);

router.get("/:id", authenticate, userController.getUserById);
router.put("/:id", authenticate, userController.updateUser);
router.delete("/:id", authenticate, authorize("admin"), userController.deleteUser);

module.exports = router;
```

---

# 📊 Status Codes Used

| Code | Meaning |
|------|----------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 500 | Server Error |

---

# 🔍 API Testing Flow

1. Register Admin
2. Login → Copy Token
3. Add Token in Header:
   Authorization: YOUR_TOKEN
4. Test Admin Routes
5. Test User Routes

---
