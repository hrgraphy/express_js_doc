# 🚀 Role-Based Full CRUD API with Two Tables (Foreign Key Concept)
## Express.js + MongoDB + JWT + MVC

---

# 📌 Project Overview

This project includes:

- User Authentication (JWT)
- Role-Based Authorization (Admin & User)
- Two Collections (User & Product)
- Foreign Key Concept using ObjectId Reference
- One-to-Many Relationship (One User → Many Products)
- Populate() for Join-like Query
- Full CRUD Operations
- MVC Architecture

---

# 📦 1️⃣ Project Setup

## Step 1: Initialize Project

```bash
mkdir role-based-foreignkey-api
cd role-based-foreignkey-api
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

Run:

```bash
npm run dev
```

---

# 📁 2️⃣ Folder Structure

```
role-based-foreignkey-api/
│
├── models/
│   ├── User.js
│   └── Product.js
│
├── controllers/
│   ├── userController.js
│   └── productController.js
│
├── routes/
│   ├── userRoutes.js
│   └── productRoutes.js
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
MONGO_URL=mongodb://127.0.0.1:27017/foreignKeyDB
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
const productRoutes = require("./routes/productRoutes");

const app = express();

app.use(cors());
app.use(express.json());

app.use("/api/users", userRoutes);
app.use("/api/products", productRoutes);

mongoose.connect(process.env.MONGO_URL)
  .then(() => console.log("MongoDB Connected"))
  .catch(err => console.log(err));

app.listen(process.env.PORT, () => {
  console.log(`Server running on port ${process.env.PORT}`);
});
```

---

# 🗄 5️⃣ User Model (Table 1)

📁 models/User.js

```js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true, select: false },
  role: {
    type: String,
    enum: ["admin", "user"],
    default: "user"
  }
}, { timestamps: true });

module.exports = mongoose.model("User", userSchema);
```

---

# 🗄 6️⃣ Product Model (Table 2 - Foreign Key Concept)

📁 models/Product.js

```js
const mongoose = require("mongoose");

const productSchema = new mongoose.Schema({
  name: { type: String, required: true },
  price: { type: Number, required: true },
  description: String,

  // FOREIGN KEY (Reference to User)
  createdBy: {
    type: mongoose.Schema.Types.ObjectId,
    ref: "User",
    required: true
  }

}, { timestamps: true });

module.exports = mongoose.model("Product", productSchema);
```

📌 Explanation:
- `createdBy` is Foreign Key
- It stores User `_id`
- `ref: "User"` links Product to User

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

# 👤 9️⃣ User Controller (Auth)

📁 controllers/userController.js

```js
const User = require("../models/User");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");

exports.register = async (req, res) => {
  const hashedPassword = await bcrypt.hash(req.body.password, 10);

  const user = await User.create({
    name: req.body.name,
    email: req.body.email,
    password: hashedPassword,
    role: req.body.role
  });

  res.status(201).json({ message: "User Registered" });
};

exports.login = async (req, res) => {
  const user = await User.findOne({ email: req.body.email }).select("+password");
  if (!user) return res.status(404).json({ message: "User not found" });

  const isMatch = await bcrypt.compare(req.body.password, user.password);
  if (!isMatch) return res.status(400).json({ message: "Invalid password" });

  const token = jwt.sign(
    { id: user._id, role: user.role },
    process.env.JWT_SECRET,
    { expiresIn: "1h" }
  );

  res.json({ token });
};
```

---

# 📦 1️⃣0️⃣ Product Controller (Full CRUD + Foreign Key)

📁 controllers/productController.js

```js
const Product = require("../models/Product");

// CREATE PRODUCT (User creates)
exports.createProduct = async (req, res) => {
  const product = await Product.create({
    name: req.body.name,
    price: req.body.price,
    description: req.body.description,
    createdBy: req.user.id
  });

  res.status(201).json(product);
};

// GET ALL PRODUCTS (Populate Foreign Key)
exports.getAllProducts = async (req, res) => {
  const products = await Product.find()
    .populate("createdBy", "name email role");

  res.json(products);
};

// GET SINGLE PRODUCT
exports.getProductById = async (req, res) => {
  const product = await Product.findById(req.params.id)
    .populate("createdBy", "name email");

  res.json(product);
};

// UPDATE PRODUCT (Owner or Admin)
exports.updateProduct = async (req, res) => {
  const product = await Product.findById(req.params.id);

  if (
    req.user.role !== "admin" &&
    product.createdBy.toString() !== req.user.id
  ) {
    return res.status(403).json({ message: "Access Denied" });
  }

  const updatedProduct = await Product.findByIdAndUpdate(
    req.params.id,
    req.body,
    { new: true }
  );

  res.json(updatedProduct);
};

// DELETE PRODUCT (Admin Only)
exports.deleteProduct = async (req, res) => {
  await Product.findByIdAndDelete(req.params.id);
  res.json({ message: "Product Deleted" });
};
```

---

# 🛣 1️⃣1️⃣ Routes

📁 routes/productRoutes.js

```js
const express = require("express");
const router = express.Router();

const productController = require("../controllers/productController");
const { authenticate } = require("../middleware/authMiddleware");
const { authorize } = require("../middleware/roleMiddleware");

router.post("/", authenticate, productController.createProduct);
router.get("/", authenticate, productController.getAllProducts);
router.get("/:id", authenticate, productController.getProductById);
router.put("/:id", authenticate, productController.updateProduct);
router.delete("/:id", authenticate, authorize("admin"), productController.deleteProduct);

module.exports = router;
```

---

# 📌 Foreign Key Concept Explanation (Exam Answer)

In MongoDB, Foreign Key is implemented using:

```
type: mongoose.Schema.Types.ObjectId
ref: "ModelName"
```

It creates a relationship between two collections.

One User → Many Products  
This is One-to-Many relationship.

Populate() works like SQL JOIN.

---

# 📊 Status Codes

| Code | Meaning |
|------|----------|
| 200 | Success |
| 201 | Created |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 500 | Server Error |

---

🔥 Now you are 100% ready for backend exam.
