#Setting up an authenticated express app

> Written by Chanse Campbell, September 2016

##Part 1 - Setting up the basic express app
Firstly run `npm init` to initialize your express app. You can click through the options, but change the entry point to be `app.js`. Once done your directory should now have a `package.json` file.

We need to `touch app.js` to create the file we'll be working out of. Now let's download the essential apps that we need in order to setup the basic express app:

```
npm install express mongoose body-parser cors morgan --save
```

*Please note that the `--save` is important. Without it any user trying to run your code won't have the blueprints for the required packages needed to run your app!*

Let's now require all of our neccessary packages within the `app.js` file and get the express app to the point where it's running on `port 3000`. At the top of your page you first need to require and invoke express. While we're here, let's also require the other packages that we downloaded so that we can use them later:

```
var express = require("express");
var app = express();

var mongoose = require('mongoose');
var bodyParser = require('body-parser');
var cors = require('cors');
```

Now let's not get ahead of ourselves. Our first task is to make sure that Express is actually running by configuring it up to run on a port. Let's add the following code to do just that:

```
var environment = app.get('env');
var port = process.env.PORT || 3000;

app.listen(port, function(){
  console.log("Express is alive and kicking on port " + port);
});
```

Now it's time to run our express app for the first time with `nodemon`. If it's running fine then we should see our `console.log` in terminal along with no red errors!

Directly above your `app.listen` line now, let's use `morgan`, `cors` and `bodyParser` for the first time by adding:

```
if('test' !== environment) {
  app.use(require('morgan')('dev'));
}

app.use(cors());

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({ extended: true }));

```
As a reminder - `morgan` is middleware only used in a dev environment to log HTTP requests, `cors` stands for 'Cross-origin resource sharing' and helps us make requests for resources that are often restricted as they come from other origins. And `bodyParser` allows us to parse incoming requests from the front-end, available to use in the back-end under the `req.body` property.

Let's finally export the app by adding `module.exports = app;` at the bottom of the page.

##Part 2 - Setting up the database
Now that we have a basic express app running on `port 3000` along with our most valuable middleware, let's turn our attention to setting up the database and creating a schema model for our first resource - users.

Firstly let's head back to terminal and create a `config` and `models` folder. Let's also touch the following files:

```
touch config/db.js
touch models/user.js
```

Now inside of our new `db.js` file let's create an object called `dbURIs` and establish the different environments for our databases:

```
dbURIs = {
  test: "mongodb://localhost/auth-express-test",
  development: "mongodb://localhost/auth-express-app",
  production: process.env.MONGOLAB_URI || "mongodb://localhost/auth-express-app"
}

module.exports = function(env) {
  return dbURIs[env];
}
```

Here we've established a test, development and production environment and exported the `dbURIs` so that we can use the variable elsewhere with whatever the environment happens to be at that time.

We also need to now utilise this exported database variable within our `app.js` file. Let's head back over and add the following just above where we require `morgan`:

```
var databaseUri = require('./config/db')(environment);

mongoose.connect(databaseUri);

```

##Part 3 - Setting up the user model with Bcrypt
Now we've required the database and connected it to mongoose, let's head over to our `user.js` file and create the Mongoose schema that will allow us to easily save a user to the database.

Inside `user.js` add the following schema:

```
var mongoose = require('mongoose');

var userSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true },
  email: { type: String, required: true, unique: true },
  passwordHash: { type: String, required: true }
});

module.exports = mongoose.model("User", userSchema);

```

Notice that we need to require `mongoose` at the top in order to use a schema to sculpt our database structure. We've also exported the model at the bottom so that we can use it within our other files.

Next up, it's time to require another npm package, this time we're going to need bcrypt - `npm install bcrypt --save`. We're also going to now need to require the package at the top of our `user.js` model with `var bcrypt = require('bcrypt');` in order to use it.

With our new bcrypt super powers we now need to add some important things to this schema. Firstly, let's set up a rule that removes certain information from our User Schema when it's displayed in JSON format (for security/usability reasons). We want to essentially blacklist some information from being viewable to the public eye. To do this, below your schema add:

```
userSchema.set('toJSON', {
  transform: function(document, json) {
    delete json.passwordHash;
    delete json.__v;
    return json;
  }
});
```
Next up, we need to setup a virtual `password` and `passwordConfirmation` for our user. Neither of these will be saved in the database, but Bcrypt has the ability to take these fields from the user and create an encrypted `passwordHash` with them to store a password more securely. To do this add:

```
userSchema.virtual('password')
  .set(function(password) {
    this._password = password;

    this.passwordHash = bcrypt.hashSync(this._password, bcrypt.genSaltSync(8));
  });

userSchema.virtual('passwordConfirmation')
  .get(function() {
    return this._passwordConfirmation;
  })
  .set(function(passwordConfirmation) {
    this._passwordConfirmation = passwordConfirmation;
  });
```

Finally, we need to actually add some code to validate that our `password` and `passwordConfirmation` are infact the same before we utilise our nifty `passwordHash`. Let's add:

```
userSchema.path('passwordHash')
  .validate(function(passwordHash) {
    if(this.isNew) {
      if(!this._password) {
        return this.invalidate('password', 'A password is required');
      }

      if(this._password !== this._passwordConfirmation) {
        return this.invalidate('passwordConfirmation', 'Passwords do not match');
      }
    }
  });
  
userSchema.methods.validatePassword = function(password) {
	return bcrypt.compareSync(password, this.passwordHash);
}
```

Our User schema should now be fully setup and ready to be used!

##Part 4 - Setting up our authentication controller
Now that we have a User schema and it's utilising Bcrypt, it's time to set up an `authentications controller` that is going to handle our login and register requests. We're going to keep these in a file seperate to any kind of user controller as they aren't particularly restful. While we're here, let's also add a `routes.js` and `tokens.js` file to our `config` folder too.

```
mkdir controllers
touch controllers/authentications.js
touch config/routes.js
touch config/tokens.js

```

Let's start by heading over to the `tokens.js` file and setup a secret that we are going to need in our `authentications controller`. Add the following:

```
module.exports = {
  secret: "Shh it's a secret"
}
```

While this isn't particulary secret, it will do for the purposes of this walkthrough. Now let's go to our `authentications.js` controller. To set this up we're going to need a little help from our final npm package that we're going to add today, `jwt`.

```
npm install jsonwebtoken --save
```

Jwt stands for `JSON Web Token` and is used as a kind of unique encoded web signature that can be recieved and decoded between two parties. Require it at the top of the controller, along with the `user.js` and `tokens.js` files.

```
var User = require('../models/user');
var jwt = require('jsonwebtoken');
var secret = require('../config/tokens').secret;
```

Now let's first add the `register` function. We want to be able to create a new User utilising the user object contained within `req.body` as provided to us by `bodyParser`. If there's an error we want one to be returned, otherwise a jwt token should be created. Let's write the function:

```
function register(req, res) {
  User.create(req.body, function(err, user) {
    if(err) return res.status(400).json(err);

    var payload = { _id: user._id, username: user.username };
    var token = jwt.sign(payload, secret, { expiresIn: 60*60*24 });

    return res.status(200).json({
      message: "Success",
      token: token
    });
  });
}
```

Notice that we've now also used our secret with our jwt token! Jwt requires both a `payload` and a `secret` as part of it's setup. 

In a similar manner let's now add a second function, this time for users trying to `login`. This function needs to use the mongoose method `findOne` to look up a user using the information recieved through the `req.body`. Let's handle our errors and distribute a jwt token in the same way as we did above:

```
function login(req, res) {
  User.findOne({ email: req.body.email }, function(err, user) {
    if(err) res.send(500).json(err);
    if(!user || !user.validatePassword(req.body.password)) {
      return res.status(401).json({ message: "Invalid credentials" });
    }

    var payload = { _id: user._id, username: user.username };
    var token = jwt.sign(payload, secret, { expiresIn: 60*60*24 });

    return res.status(200).json({
      message: "Success",
      token: token
    });
  });
}
```

Our final job here is to make sure we export the controller so that we can use our functions within our `routes.js` file.

```
module.exports = {
  register: register,
  login: login
}
```

##Part 5 - Setting up our user controller
Now that we've setup our authentications controller, let's repeat the process again but for our user. To start, lets create our users controller.

```
touch controllers/users.js
``` 

Make sure that you then require the `user` model again at the top of the page `var User = require('../models/user');` and then we should be ready to write our controller functions.

We need to create functions utilising `mongoose` methods to access the user's that will exist within our database once they've registered. Firstly, let's `find` them all with an index function:

```
function usersIndex(req, res) {
  User.find(function(err, users) {
    if(err) return res.status(500).json(err);
    return res.status(200).json(users);
  });
}
```

With our first route up and running, the rest should be easier to follow. Pay attention to the mongoose methods used on `user.` and the different parameters that need to be passed into them:

```
function usersCreate(req, res) {
  User.create(req.body, function(err, user) {
    if(err) return res.status(400).json(err);
    return res.status(201).json(user);
  });
}

function usersShow(req, res) {
  User.findById(req.params.id, function(err, user) {
    if(err) return res.status(500).json(err);
    if(!user) return res.status(404).json({ message: "Could not find a user with that id" });
    return res.status(200).json(user);
  });
}

function usersUpdate(req, res) {
  User.findByIdAndUpdate(req.params.id, req.body, { new: true, runValidators: true }, function(err, user) {
    if(err) return res.status(400).json(err);
    return res.status(200).json(user);
  });
}

function usersDelete(req, res) {
  User.findByIdAndRemove(req.params.id, function(err) {
    if(err) return res.status(500).json(err);
    return res.status(204).send();
  });
}
```

Finally, let's export the controller just like we did previously:

```
module.exports = {
  index: usersIndex,
  create: usersCreate,
  show: usersShow,
  update: usersUpdate,
  delete: usersDelete
}
```

##Part 6 - Setting up the route handling
We're nearly there now! Our final task before being able to test the authenticated express app is to set up the route handling to login, register or access our users.

Let's head over to our `routes.js` file and first require the following at the top of the page:

```
var router = require('express').Router();
var jwt = require('jsonwebtoken');
var secret = require('../config/tokens').secret;
var usersController = require('../controllers/users');
var authController = require('../controllers/authentications');
```

Notice that we've required jwt and our secret token here too. This is mainly because we need to setup some middleware to check for a secure route with a valid jwt token before the route can be used. To do this, let's now add the middleware:

```
function secureRoute(req, res, next) {
  if(!req.headers.authorization) return res.status(401).json({ message: "Unauthorized" });

  var token = req.headers.authorization.replace('Bearer ', '');

  jwt.verify(token, secret, function(err, payload) {
    if(err || !payload) return res.status(401).json({ message: "Unauthorized" });

    req.user = payload;
    next();
  });
}
```

Finally, let's link our required controllers up to the correct paths and methods and export the router out:

```
router.route('/users')
  .all(secureRoute)
  .get(usersController.index)
  .post(usersController.create);

router.route('/users/:id')
  .all(secureRoute)
  .get(usersController.show)
  .put(usersController.update)
  .patch(usersController.update)
  .delete(usersController.delete);

router.post('/register', authController.register);
router.post('/login', authController.login);

module.exports = router;
```

We're so close! We need to require our newly created router within our `app.js` file and use it. Let's head back over and add this above where we require the `databaseURI`:

```
var routes = require('./config/routes');
```

Now let's finally use it by adding it in **below** where we require `bodyParser`:

```
app.use('/api', routes);
```

It's time to test our new API out in insomnia. Head over and try submitting a `POST` request to `http://localhost:3000/api/register` to create a user using the same structure created in your schema. 

```
{
  "username": "chansec",
  "email": "chanse@chanse.com",
  "password": "password",
  "passwordConfirmation": "password"
}
```

If it's working you should recieve back a `jwt token` back from the API. You can now add the token into to the Insomnia header and test your `/users` routes. Please note that you will *only* be able to perform CRUD actions on your users when the jwt token is present in the header, otherwise you'll get an `unauthorized message`.

Congratulations, your basic express app is now authenticated for users! Close your laptop and go and grab a beer to celebrate.