---
layout: post
title: "Developing Super-Fast SPAs with choo"
description: "Follow along as we secure a simple, Choo based application using Auth0."
date: "2018-09-27 08:30"
author:
  name: "Hendrik Wallbaum"
  url: "hoverbaum"
  mail: "mail@hendrikwallbaum.de"
  avatar: "https://twitter.com/hoverbaum/profile_image?size=original"
---

**TL;DR:** Today we will get familiar with Choo as a super slim and fun frontend framework, build a friendly application and secure it using Auth0. Check the [working demo](https://auth0-secured-choo.netlify.com/) and [GitHub repo](https://github.com/HoverBaum/choo-auth0).

# A simple and fun Choo app secured with Auth0

Securing an application and authenticating users is a crucial part for every modern web app. Luckily the times of having to build your own authentication solution using PHP and MySQL databases are over. In this time an age we have great services that specialize in providing authentication and authorization as a service. By focusing on this single task services are able to provide a far better solution than anybody implementing a user system as part of their app ever could. And it is super easy to integrate these services thus everyone benefits.

To demonstrate how easily you can integrate such a service we are going to build a simple Choo (like the train goes choo choo) application. Users will be able to sign up and log in. After Logging in they will see an identiticon (liek teh standard GitHub avatars) generated just for them. Instead of building the user system ourselfes we will let [Auth0](https://auth0.com/) handle that for us.

You can check out the finished code in my [choo-auth0 repo](https://github.com/HoverBaum/choo-auth0), it uses choo ^6.13.0 and auth0-js ^9.7.2. Let's dive in.

## Introduction to Choo

[Choo 🚂🚋🚋🚋🚋🚋](https://github.com/choojs/choo) is a super simple, easy to get into, 4kb big frontend framework for fun functional programming. It comes with a great handbook walking you through [your first choo app](https://handbook.choo.io/your-first-choo-app/), explaining not only about choo but web development in general. Choo wants to be your next framework, not your last. Thus focussing on using JavaScript native features as much as possible aiming to be replaceable. You can also think of it as a best of Redux framework, providing a router, state management via events and rendering of your state.

The thing I like most about Choo is the core philosophy it is build around.

> Programming should be fun and light, not stern and stressful

A simple counter application in choo looks like this:

```javascript
var html = require('choo/html')
var devtools = require('choo-devtools')
var choo = require('choo')

var app = choo()
app.use(devtools())
app.use(countStore)
app.route('/', mainView)
app.mount('body')

function mainView (state, emit) {
  return html`
    <body>
      <h1>count is ${state.count}</h1>
      <button onclick=${() => emit('increment', 1))}>Increment</button>
    </body>
  `
}

function countStore (state, emitter) {
  state.count = 0
  emitter.on('increment', function (count) {
    state.count += count
    emitter.emit('render')
  })
}
```

Choo relies on Node for dependencies thus we use `require` here instead of `import`. You can also see that Choo provides `emit()` as a means for components to fire events. To handle those events you can define functions to use that take the state and emitter as input and react to events by updating the state.

## Scaffolding a Choo App

Setting up our Choo application is as simple as running `npx create-choo-app choo-auth0`. Similar to *create-react-app* this script will create a basic Choo application with all required dependencies in a folder of our choosing.

Take a look at the generated files. For our simple, authetnicated application presenting each user with an identicon we will:

- Create views in `views`
- Create an authentication store in `stores`
- Add all of the above in `index.js` to strap it together

You can run the application using `npm start`. Our generated application comes with a few parts we don't need, you can go ahead and:

- delete the assets folder.
- remove the `store/clicks.js` store.
- Maybe do some cleanup by removing `manifest.json` and `sw.js` which are super nice files for production bbut not needed for a walktrough.

Apart from a starting view and a 404 page Choo also comes bundled with [Tachyons](http://tachyons.io/) a lightweight CSS framework for layouting. It provides tons of classes to style and layout our application. You will likely find the classes provided by Tachyons to be quite intuitive. `paX` for example adds padding *X* (1-7) to an element.

### Different views

We are going to build two views for our application:

1. main - for logged out users, living on `/`
2. dashboard - to display after login, living on `/dashboard`

Let's go ahead and create static templates for our views. First we are going to create the view to log in from.

```javascript
// views/main.js
const html = require('choo/html')
const TITLE = 'secured-choo - main'
module.exports = view

function view (state, emit) {
  if (state.title !== TITLE) emit(state.events.DOMTITLECHANGE, TITLE)
  return html`
    <body class="lh-copy sans-serif">
      <main class="pa3 cf center">
        <div>
          <h1>Please login to see the application</h1>

          <button
            class="f5 black bg-animate hover-bg-black hover-white inline-flex items-center pa3 ba border-box bg-white pointer"
            onclick=${() => emit('auth:startAuthentication')}
          >
            Login
          </button>
        </div>        
      </main>
    </body>
  `
}
```

Note how we are using Tachyon classes to style our application and how we already decided on an event that will trigger authentication. We named it `auth:startAuthentication`. Later we will build a store that listens for this event to start the authentication process for our user.

Next up is a logged in view. Here we want to display an identicon. For that we will use [identicon.js](https://github.com/stewartlord/identicon.js). In this first step we will generate an identicon for a hardcoded value which we will later swap with the users ID. Don't forget to run `npm i --save identicon.js`.

```javascript
const html = require('choo/html')
const Identicon = require('identicon.js')
const TITLE = 'dashboard'
module.exports = view

function view (state, emit) {
  if (state.title !== TITLE) emit(state.events.DOMTITLECHANGE, TITLE)
  const userId = 'Not yet a real user ID'
  const avatarData = new Identicon(userId, 420).toString()
  return html`
    <body class="lh-copy sans-serif">
      <main class="pa3 cf center">
        <h1>Welcome to your dashboard 🎉</h1>

        <p>Here is an avatar generated just for you:</p>

        <img src="data:image/png;base64,${avatarData}">

        <p>Let's try to log out and in again.</p>

        <button
          class="f5 black bg-animate hover-bg-black hover-white inline-flex items-center pa3 ba border-box bg-white pointer"
          onclick=${() => emit('auth:logout')}
        >
          Logout
        </button>

        <p>Or <a href="/">go to the landing page</a> to see how the site interacts with logged in users.</p>

      </main>
    </body>
  `
}

```

The main view actually already existed in create-choo-app, to hook up our dashboard view make sure to add it to the application in `index.js` using 

```javascript
app.route('/dashboard', require('./views/dashboard'))
```

Again we already prepare events to log a user out and we are presenting him with an option to see the landig page as a logged in user (which we didn't implement yet). The second argument we are passing to `new Identicon()` is the dimension of the identicon we want to get out. Feel free to take a breather here and play around with values for our *userId*.

### Mock store data

Next up is the state of our application. For that we will define a store which handles our applications state. You can think of this as a build in Redux. Let's create a `stores/auth.js` store now. It will handle the two events we fired up above and provide data about a logged in user.

```javascript
// stores/auth.js
module.exports = authStore

function authStore (state, emitter) {
  // Our initial auth state.
  state.auth = {
    loggedIn: false,
    idToken: null,
    userId: null
  }

  // Mock login and logout actions for now.
  emitter.on('auth:startAuthentication', () => {
    state.auth.loggedIn = true
    state.auth.userId = 'Some totally random value for now' 
    emitter.emit(state.events.PUSHSTATE, '/dashboard')
  })

  emitter.on('auth:logout', () => {
    state.auth.loggedIn = false
    state.auth.userId = ''
    emitter.emit(state.events.PUSHSTATE, '/')
  })
}

```

Once we add this store to our app using `app.use(require('./stores/auth'))` we are good to go with our mock. Before we can see the result we need to update how we get the *userId* in our dashboard. 

```javascript
const userId = state.auth && state.auth.userId
```

Login and Logout are now working (though only hardcoded and mocked), clicking the login button will redirect you to the dashboard where you will see an identicon. You can also log out again to get back to the landing page.

This is a great opportunity to add some logic to our main page differentiating between logged in and out users. Depending on `state.auth.loggedIn` we will either display the login button or some text directing the user to their dashboard.

```javascript
${!state.auth.loggedIn ? html`
  <div>
    <h1>Please login to see the application</h1>

    <button
      class="f5 black bg-animate hover-bg-black hover-white inline-flex items-center pa3 ba border-box bg-white pointer"
      onclick=${() => emit('auth:startAuthentication')}
    >
      Login
    </button>
  </div>
` : html`
  <div>
    <h1>You are logged in</h1>  

    <a href="/dashboard">To your dashboard</a>
  </div>
`}
```

No magic here, just normal ES6 template literals using `${}` to introduce logic inside of strings and based on the *loggedIn* state using a [ternary operator](https://en.wikipedia.org/wiki/%3F:) returning one string or another.

-----

## Securing Your Choo App with Auth0

User authentication is at the same time a common and quite the hard problem to solve. A vast number of applications out there requires some sort of user authentication. With the most common use case being us wanting to save our users data so that only they can edit it. Luckily a common problem means that you can extract that problem from your over all application and simply pull it in as a service.

Once such service is Auth0. It is one of the longest standing and easiest to use authentication service that comes with a bunch of features and starts you off for free for your first users. Using Auth0 you can get started with your App and once it catches on Auth0 will scale with you to thousands of users and advanced features like [OAuth](https://auth0.com/docs/protocols/oauth2) secured APIs for third parties to intergrate with your application.

### Creating an Auth0 Application

Auth0 organizes projects as "applications". After signing into your account you can create a new application and configure it. This can be done in [your Auth0 Applications](https://manage.auth0.com/#/applications). Simply hit the "+ Create Application" button to get started.

![](create-app.png)

Come up with a name, something simple lie "choo test" will do here. Then select "Single Page Web Application" fro the "application type" and hit create. Next head over to your applications settings and make sure to note down the *Cleint ID* and *Domain* for your application (we are going to need those in a moment.

You also need to setup an Allowed Callback URL: `https://localhost:8080/dashboard´. Simply add it on a blank line in your applications settings "Allowed Callback URLs" section. Apart from that you can use the default settings, just make sure to save your changes at the bottom of the page.

### Configuring Auth0 on Choo

Now we are setup with an application in Auth0 we are ready to incorporate this into our Choo App. Luckily we are already fireing some actions when users hit the relevant buttons in our application for login and logout. 

To connect Auth0 with our Store we first need to initialize a WebAuth instanze in our `stores/auth.js`.

```javascript
const auth0 = require('auth0-js')
const webAuth = new auth0.WebAuth({
  domain: process.env.DOMAIN,
  clientID: process.env.CLIENT_ID,
  responseType: 'id_token',
  scope: 'choo:auth0',
  redirectUri: 'https://localhost:8080/dashboard'
})
```

This is copied from the [quick start guide for JavaScript](https://auth0.com/docs/quickstart/spa/vanillajs). Note however that we replaced the domain and clientID with environment variables. Feel free to hardcode yours here or pass them in via the environment, either way works. I opted for environment variables so that I wouldn't have to push secrets into Git which you should never do! Following that I find it good practice to use environment variables instead.

Next up is the logic to handle our application being called after a successful login. For that we are going to add a piece of code at the bottom of our store. Again inspired by the quick start guide Auth0 provides.

```javascript
// If we are serverside at this point return ebcause now we will look at authentication on the client.
if (typeof window === 'undefined') return

// Check local storage if we are logged in.
const storedIdToken = window.localStorage.idToken
const storedUserId = window.localStorage.userId
if (storedIdToken && storedUserId) {
  // Save authentication information to the state.
  state.auth.loggedIn = true
  state.auth.idToken = JSON.parse(storedIdToken)
  state.auth.userId = storedUserId
  emitter.emit(state.events.RENDER)
}

// On page load check if there is a hash that we should handle.
webAuth.parseHash((err, authResult) => {
  if (authResult) {
    // Save authentication information to localStorage.
    window.localStorage.idToken = JSON.stringify(authResult.idToken)
    // Some values are already strings and don't need to be stringified.
    window.localStorage.userId = authResult.idTokenPayload.sub
    console.log(authResult)

    // Now update the store.
    state.auth.loggedIn = true
    state.auth.idToken = authResult.idToken
    state.auth.userId = authResult.idTokenPayload.sub
    emitter.emit(state.events.RENDER)

    // Remove the hash after using is.
    emitter.emit(state.events.REPLACESTATE, window.location.pathname)
  } else if (err) {
    console.error(err)
  }

  // Re-direct not logged in users to login page.
  // In our case login is under '/'.
  if (!state.auth.loggedIn) {
    emitter.emit(state.events.REPLACESTATE, '/')
  }
})
```

Here we first check to only do this on the client and not during server side rendering. Then we check if we are already logged in as well as if we are called after a successful login and handle those cases accordingly. Now all we need to update are the event handlers for the button clicks that trigger login and logout.

```javascript
emitter.on('auth:startAuthentication', () => webAuth.authorize())

emitter.on('auth:logout', () => {
  window.localStorage.clear('token')
  state.auth.loggedIn = false
  emitter.emit(state.events.PUSHSTATE, '/')
})
```

Our logic to set the state already moved to the handling of being called after a successful login. And with that we are done. Congratulations, you just secured a Choo based application using Auth0!

It's amazing how we only had to add some logic inside the store to handle the authentication flows. Our views are totally untouched, they already handle all the states our application can be in. Just the ways in which we reach these states has changed.

## Conclusion and Next Steps

What a day! We build a little application in Choo greeting our visitors with a personalized identicon and then authenticated users through Auth0. The tedious task of setting up and creating an authentication system was taken care of for us and all we had to do is map authentication flows on to our state. Don't forget to check the [repo with the final code]((https://github.com/HoverBaum/choo-auth0) and the [demo](https://auth0-secured-choo.netlify.com/).

Moving forward have a look at the [Choo handbook](https://handbook.choo.io/your-first-choo-app/) for a better understanding of the framework. After you decided on the application you want to build your next read will likely be about [using and API with Auth0](https://auth0.com/docs/quickstart/spa/vanillajs/03-calling-an-api).

Now that we can stop worrying about authentication, go out and build something awesome.
