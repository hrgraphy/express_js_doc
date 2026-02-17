# 🚀 Role-Based Authentication API (Express.js + MongoDB + JWT)

This project demonstrates:

- ✅ Express.js Server Setup
- ✅ MongoDB Connection
- ✅ MVC Architecture
- ✅ JWT Authentication
- ✅ Role-Based Authorization (Admin & User)
- ✅ Protected Routes

---

# 📦 1️⃣ Project Setup

## Step 1: Create Project

```bash
mkdir role-based-api
cd role-based-api
npm init -y
```

---

## Step 2: Install Dependencies

```bash
npm install express mongoose jsonwebtoken bcryptjs cors dotenv
npm install nodemon --save-dev
```

### 📚 Package Explanation

| Package | Purpose |
|----------|----------|
| express | Web framework |
| mongoose | MongoDB ODM |
| jsonwebtoken | JWT Authentication |
| bcryptjs | Password hashing |
| cors | Enable cross-origin |
| dotenv | Environment variables |
| nodemon | Auto restart server |

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

# 📁 2️⃣ Folder Structure (MVC)

```
role-based-api/
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

Create `.env` file:

```
MONGO_URL=mongodb://127.0.0.1:27017/roleBasedDB
JWT_SECRET=mysecretkey
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

app.listen(3000, () => {
  console.log("Server running on port 3000");
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
    required: true
  },
  email: {
    type: String,
    required: true,
    unique: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ["admin", "user"],
    default: "user"
  }
});

module.exports = mongoose.model("User", userSchema);
```

---

# 🎮 6️⃣ Controller

📁 controllers/userController.js

```js
const User = require("../models/User");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");

exports.register = async (req, res) => {
  try {
    const { name, email, password, role } = req.body;

    const hashedPassword = await bcrypt.hash(password, 10);

    const user = new User({
      name,
      email,
      password: hashedPassword,
      role
    });

    await user.save();
    res.status(201).json({ message: "User Registered" });

  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.login = async (req, res) => {
  try {
    const { email, password } = req.body;

    const user = await User.findOne({ email });
    if (!user) return res.status(404).json({ message: "User not found" });

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) return res.status(400).json({ message: "Invalid password" });

    const token = jwt.sign(
      { id: user._id, role: user.role },
      process.env.JWT_SECRET,
      { expiresIn: "1h" }
    );

    res.json({ token });

  } catch (err) {
    res.status(500).json({ error: err.message });
  }
};

exports.getAllUsers = async (req, res) => {
  const users = await User.find();
  res.json(users);
};
```

---

# 🔐 7️⃣ Authentication Middleware

📁 middleware/authMiddleware.js

```js
const jwt = require("jsonwebtoken");

exports.authenticate = (req, res, next) => {
  const token = req.headers.authorization;

  if (!token) return res.status(401).json({ message: "Unauthorized" });

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
      return res.status(403).json({ message: "Forbidden: Access Denied" });
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

// Logged-in users
router.get("/profile", authenticate, (req, res) => {
  res.json({ message: "User Profile", user: req.user });
});

// Admin only
router.get(
  "/all",
  authenticate,
  authorize("admin"),
  userController.getAllUsers
);

module.exports = router;
```

---

# 🧪 🔍 API Testing (Postman)

## Register Admin

POST  
`http://localhost:3000/api/users/register`

```json
{
  "name": "Admin",
  "email": "admin@gmail.com",
  "password": "123456",
  "role": "admin"
}
```

---

## Register Normal User

```json
{
  "name": "Harmik",
  "email": "harmik@gmail.com",
  "password": "123456"
}
```

(Default role = user)

---

## Login

POST  
`http://localhost:3000/api/users/login`

Copy the token.

---

## Access Admin Route

GET  
`http://localhost:3000/api/users/all`

Header:

```
Authorization: YOUR_TOKEN
```

---

# 📊 Status Codes Used

| Code | Meaning |
|------|----------|
| 200 | Success |
| 201 | Created |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 500 | Server Error |

