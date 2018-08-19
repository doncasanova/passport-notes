
# Adding Passport to your Mongo Project:
The benefits of going through this somewhat painful process:
- All requests to your server have the very useful `req.user` property.
- All requests to your server can also be checked with `req.isAuthenticated()` to lock out rogue users.
- Passport can use cookies to ensure that a user stays logged in (for a reasonable period of time).
- It becomes easy to check from the global `App.js` whether a user is logged in, and thus what to display in the Nav bar.
- We'll use `bcrypt` to handle encryption.

## Basic Setup:
### Install:
Run `npm install passport passport-local express-session cookie-session brcrypt --save`.

### Additions to `server.js`
```
// This will handle all security stuff:
const passport = require('./strategies/user-strategy.js');
// This is a more generic file that handles cookies:
const sessionConfig = require('./modules/session-middleware.js');

// Passport session config and init:
app.use(sessionConfig);
app.use(passport.initialize());
app.use(passport.session());
```

## Adding Files:
### Add `modules` and `strategies` directories
- If you do not have these directories in your root project directory, add them.
- You'll also need a `models` directory, but you should already have one.
- We'll be adding four files in these three directories.

### Step 1: Add `encryption.js` to your `modules` directory (it will be used by our `Strategy`)
In this file, write:
```
var bcrypt = require('bcrypt');
var SALT_WORK_FACTOR = 10;

var publicAPI = {
  encryptPassword: function(password) {
      var salt = bcrypt.genSaltSync(SALT_WORK_FACTOR);
      return bcrypt.hashSync(password, salt);
  },
  comparePassword: function(candidatePassword, storedPassword) {
      return bcrypt.compareSync(candidatePassword, storedPassword);
  }
};

module.exports = publicAPI;
```

### Step 2: Add `session-middleware.js` to your `modules` directory
In this file, write:

```
const cookieSession = require('cookie-session');

const serverSessionSecret = () => {
  if (!process.env.SERVER_SESSION_SECRET ||
      process.env.SERVER_SESSION_SECRET.length < 8 ||
      process.env.SERVER_SESSION_SECRET === warnings.exampleBadSecret) {
    // Warning if user doesn't have a good secret:
    console.log('Yikes!');
  }
  return process.env.SERVER_SESSION_SECRET;
};

module.exports = cookieSession({
   secret: serverSessionSecret() || 'secret',
   key: 'user', // this is the name of the req.variable. 'user' is convention, but not required
   resave: 'true',
   saveUninitialized: false,
   cookie: { maxage: 60000, secure: false }
});
```

### Step 3: Add a `Person.js` file to your `models` directory
Nothing fancy here, probably you'll just integrate this with your current User model:
```
const mongoose = require('mongoose');
const Schema = mongoose.Schema;

const PersonSchema = new Schema({
  username: { type: String, required: true, index: { unique: true } },
  password: { type: String, required: true },
});

module.exports = mongoose.model('Person', PersonSchema);
```

### Step 4: Add `user-strategy.js` to your `strategies` directory:
This is the big kahuna. In this file, write:
```
const passport = require('passport');
const LocalStrategy = require('passport-local').Strategy;
const encryptLib = require('../modules/encryption');
const Person = require('../models/Person');

passport.serializeUser((user, done) => {
  done(null, user.id);
});

passport.deserializeUser((id, done) => {
  Person.findById(id).then((result) => {
    // Handle Errors
    const user = result;

    if (!user) {
      // user not found
      done(null, false, { message: 'Incorrect credentials.' });
    } else {
      // user found
      done(null, user);
    }
  }).catch((err) => {
    console.log('query err ', err);
    done(err);
  });
});

// Does actual work of logging in
passport.use('local', new LocalStrategy({
  passReqToCallback: true,
  usernameField: 'username',
}, ((req, username, password, done) => {
    Person.find({ username })
      .then((result) => {
        const user = result && result[0];
        if (user && encryptLib.comparePassword(password, user.password)) {
          // all good! Passwords match!
          done(null, user);
        } else if (user) {
          // not good! Passwords don't match!
          done(null, false, { message: 'Incorrect credentials.' });
        } else {
          // not good! No user with that name
          done(null, false);
        }
      }).catch((err) => {
        console.log('error', err);
        done(null, {});
      });
  })));

module.exports = passport;
```

### Phew!
Now, if there is a god, we should be able to ping our authentication route like this:
```
router.get('/', (req, res) => {
  // check if logged in
  if (req.isAuthenticated()) {
    res.send(req.user);
  } else {
    res.sendStatus(403);
  }
});
```

### I Lied, There's More
There are a few more routes we need -- for registering, logging in, and logging out:
```
// REGISTER:
router.post('/register', (req, res, next) => {
  const username = req.body.username;
  const password = encryptLib.encryptPassword(req.body.password);
  const newPerson = new Person({ username, password });
  newPerson.save()
    .then(() => {
      res.sendStatus(201);
    })
    .catch((err) => { next(err); });
});

// LOGIN:
router.post('/login', userStrategy.authenticate('local'), (req, res) => {
  res.sendStatus(200);
});

// LOGOUT (clear session data):
router.get('/logout', (req, res) => {
  // Use passport's built-in method to log out the user
  req.logout();
  res.sendStatus(200);
});
```

Oh, and before we can use these routes, we'll need to `require` our dependencies:
```
const encryptLib = require('../modules/encryption.js'); // The '.js' is optional, but I like it
const Person = require('../models/Person.js');
const userStrategy = require('../strategies/user-strategy.js');
```

### Conclusion
If that didn't work, I probably messed up, reach out to me and we'll get to the bottom of it.
