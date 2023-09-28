# Express.js
This code sets up an Express.js server with user authentication using Passport.js, a MongoDB database for storing user and product data, and various routes for creating users, authenticating users, logging out, creating products, and fetching product data. It's a simplified example of an e-commerce website backend.
// Import necessary modules
const express = require('express');
const bodyParser = require('body-parser');
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const session = require('express-session');
const mongoose = require('mongoose');

// Create an Express.js web server
const app = express();

// Connect to a MongoDB database
mongoose.connect('mongodb://localhost/ecommerce', { useNewUrlParser: true });
mongoose.connection.on('error', (err) => {
  console.error('MongoDB connection error:', err);
  process.exit(-1);
});

// Define a Mongoose schema and model for users and products
const userSchema = new mongoose.Schema({
  username: String,
  password: String,
  email: String,
});

const productSchema = new mongoose.Schema({
  name: String,
  price: Number,
  description: String,
});

const User = mongoose.model('User', userSchema);
const Product = mongoose.model('Product', productSchema);

// Configure passport for user authentication
passport.use(new LocalStrategy(
  (username, password, done) => {
    User.findOne({ username }, (err, user) => {
      if (err) { return done(err); }
      if (!user) { return done(null, false, { message: 'Incorrect username.' }); }
      if (user.password !== password) { return done(null, false, { message: 'Incorrect password.' }); }
      return done(null, user);
    });
  }
));

passport.serializeUser((user, done) => {
  done(null, user.id);
});

passport.deserializeUser((id, done) => {
  User.findById(id, (err, user) => {
    done(err, user);
  });
});

// Middleware for parsing JSON requests and session management
app.use(bodyParser.json());
app.use(session({ secret: 'secret-key', resave: false, saveUninitialized: false }));
app.use(passport.initialize());
app.use(passport.session());

// Create a new user
app.post('/users', async (req, res) => {
  try {
    const user = new User(req.body);
    await user.save();
    res.status(201).json(user);
  } catch (err) {
    res.status(400).json({ error: 'Invalid data' });
  }
});

// Authenticate a user
app.post('/login', passport.authenticate('local'), (req, res) => {
  res.json(req.user);
});

// Logout a user
app.get('/logout', (req, res) => {
  req.logout();
  res.redirect('/');
});

// Create a new product
app.post('/products', async (req, res) => {
  try {
    const product = new Product(req.body);
    await product.save();
    res.status(201).json(product);
  } catch (err) {
    res.status(400).json({ error: 'Invalid data' });
  }
});

// Get a list of all products
app.get('/products', async (req, res) => {
  const products = await Product.find();
  res.json(products);
});

// Start the server
const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Server is listening on port ${port}`);
});
