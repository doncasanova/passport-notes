
## Adding Passport to your project:
The benefits of going through this somewhat painful process:
- All requests to your server have the very useful `req.user` property.
- All requests to your server can also be checked with `req.isAuthenticated()` to lock out rogue users.
- Passport can use cookies to ensure that a user stays logged in (for a reasonable period of time).
- It becomes easy to check from the global `App.js` whether a user is logged in, and thus what to display in the Nav bar.
- We'll use `bcrypt` to handle encryption.

### Install:
Run `npm install passport passport-local express-session brcrypt-nodejs --save`.

### Additions to `server.js`
```// This is the file that will handle all security stuff:
const passport = require('./strategies/user.strategy');
// (Keep in mind we must change this file for production version):
// This is a more generic file that handles cookies:
const sessionConfig = require('./modules/session-middleware');

// Passport Session Configuration
app.use(sessionConfig);
// Start up passport sessions
app.use(passport.initialize());
app.use(passport.session());
```

### Add `modules` and `strategies` directories
- If you do not have these directories in your root project directory, add them.
- You'll also need a `models` directory, but you should already have one.
- We'll be adding four files in these three directories.

### Step 1: Add `encryption.js` to your `modules` directory (it will be used by our `Strategy`)
In this file, write:
```var bcrypt = require('bcrypt-nodejs');
var SALT_WORK_FACTOR = 10;

var publicAPI = {
  encryptPassword: function(password) {
      var salt = bcrypt.genSaltSync(SALT_WORK_FACTOR);
      return bcrypt.hashSync(password, salt);
  },
  comparePassword: function(candidatePassword, storedPassword) {
      console.log('comparing passwords');
      console.log(candidatePassword, storedPassword);
      //ndidatePassword, this.password
      return bcrypt.compareSync(candidatePassword, storedPassword);
  }
};

module.exports = publicAPI;
```

### Step 2: Add `session.config.js` to your `modules` directory
In this file, write:
```var session = require('express-session');

module.exports = session({
   secret: 'secret',
   key: 'user', // this is the name of the req.variable. 'user' is convention, but not required
   resave: 'true',
   saveUninitialized: false,
   cookie: { maxage: 60000, secure: false }
});
```

### Step 3: Add a `Person.js` file to your `models` directory
Nothing fancy here, probably you'll just integrate this with your current User model:
```const mongoose = require('mongoose');

const Schema = mongoose.Schema;

// Mongoose Schema
const PersonSchema = new Schema({
  username: { type: String, required: true, index: { unique: true } },
  password: { type: String, required: true },
});

module.exports = mongoose.model('Person', PersonSchema);
```

### Add `user.strategy.js` to your `strategies` directory:
This is the big kahuna. In this file, write:
```const passport = require('passport');
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
    if( verbose ) console.log( 'trying to log in:', username, password );
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
