# Social-Media-Platform
Social Media Platform
// Require the necessary modules
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

// Connect to the database
mongoose.connect('mongodb://localhost/social-media-platform', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
  useCreateIndex: true,
});

// Define the user schema
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, unique: true, required: true },
  password: { type: String, required: true },
});

// Define the post schema
const postSchema = new mongoose.Schema({
  title: { type: String, required: true },
  content: { type: String, required: true },
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  createdAt: { type: Date, default: Date.now },
  comments: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Comment' }],
});

// Define the comment schema
const commentSchema = new mongoose.Schema({
  content: { type: String, required: true },
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  post: { type: mongoose.Schema.Types.ObjectId, ref: 'Post', required: true },
  createdAt: { type: Date, default: Date.now },
});

// Define the user model
const User = mongoose.model('User', userSchema);

// Define the post model
const Post = mongoose.model('Post', postSchema);

// Define the comment model
const Comment = mongoose.model('Comment', commentSchema);

// Serve the web app
app.use(express.static('public'));
app.use(bodyParser.json());

// Register a new user
app.post('/api/register', async (req, res) => {
  try {
    const hashedPassword = await bcrypt.hash(req.body.password, 10);
    const user = new User({
      name: req.body.name,
      email: req.body.email,
      password: hashedPassword,
    });
    await user.save();
    res.status(201).json({ message: 'User created' });
  } catch {
    res.status(500).json({ message: 'Error creating user' });
  }
});

// Login an existing user
app.post('/api/login', async (req, res) => {
  const user = await User.findOne({ email: req.body.email });
  if (!user) {
    return res.status(400).json({ message: 'User not found' });
  }
  try {
    if (await bcrypt.compare(req.body.password, user.password)) {
      const token = jwt.sign({ userId: user._id }, 'secretkey');
      return res.status(200).json({ token });
    } else {
      return res.status(400).json({ message: 'Invalid password' });
    }
  } catch {
    res.status(500).json({ message: 'Error logging in' });
  }
});

// Create a new post
app.post('/api/posts', async (req, res) => {
  const token = req.headers.authorization.split(' ')[1];
  const decoded = jwt.verify(token, 'secretkey');
  const user = await User.findById(decoded.userId);
  if (!user) {
    return res.status(400).json({ message: 'User not found' });
  }
  const post = new Post({
    title: req.body.title,
    content: req.body.content
