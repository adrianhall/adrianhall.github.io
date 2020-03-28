---
title: "Integrating Microsoft Login and MSAL with React and Redux"
categories:
  - JavaScript
  - React
---

I have a new app I am working on.  It's sort of a 1990's style text MUD, but I'm bringing it "up to this century" with a host of new features.  I'm writing the first front-end in [React](https://reactjs.org).  

So, what does a modern MUD app look like?  Well, I'm not into storing usernames and password any more, so I'm going to use a Microsoft OAuth service instead of a user database.  My front end application handles state through [Redux](https://redux.js.org).  

## Configuring Redux

I've got a store set up as follows:

{% highlight javascript %}
import { applyMiddleware, createStore } from 'redux';
import thunkMiddleware from 'redux-thunk';
import { createLogger } from 'redux-logger';
import appReducer from './reducers';
import * as actions from './actions';

const loggerMiddleware = createLogger({ collapse: true });
const store = createStore(
  appReducer,
  applyMiddleware(loggerMiddleware, thunkMiddleware)
);

export { actions, store };
{% endhighlight %}

This is about as basic a Redux store as it gets.  I've added [redux-logger](https://www.npmjs.com/package/redux-logger) for logging the Redux store state transitions, and [redux-thunk](https://www.npmjs.com/package/redux-thunk) for asynchronous actions.

> Redux is a uni-directional flow state storage system.  Your application initiates actions, which are then handled by reducers that mutate the state of the application.  This then causes rendering changes as a result of the change.  If you don't understand this flow, I recommend [reading this tutorial](https://daveceddia.com/redux-tutorial/).

This gets me thinking about state within the application.  I need three pieces of state within my application:

* The number of network requests in flight right now.  This is used to display a spinner in the title bar when there is something going on.
* The last network error that was encountered.  This is used to display a warning symbol to alert the user that something went wrong.
* The current user identity.

In Redux, I need to create an action identifier and an action creator for each state change I want to use:

{% highlight javascript %}
const NETWORK_ERROR  = Symbol.for('network.error');
const NETWORK_START  = Symbol.for('network.start');
const NETWORK_STOP   = Symbol.for('network.stop');
const SIGN_IN        = Symbol.for('network.auth-sign-in');
const SIGN_OUT       = Symbol.for('network.auth-sign-out');

const startNetwork   = () => ({ type: NETWORK_START });
const stopNetwork    = () => ({ type: NETWORK_STOP });
const networkError   = (error) => ({ type: NETWORK_ERROR, error });
const networkSignIn  = (identity) => ({ type: SIGN_IN, identity });
const networkSignOut = () => ({ type: SIGN_OUT });
{% endhighlight %}

I can use the `store.dispatch()` method to dispatch each action creator.  Depending on how you organize your Redux implementation, you may have to export some and not others.  As an example, for smaller apps, I tend to keep everything for one reducer or section of the state in one file, and export the actions as needed.

There are two types here - `error` and `identity`.  The `error` type is a basic JavaScript `Error`.  The `identity` type will be a new model called `Identity`:

{% highlight javascript %}
/**
 * Encapsulation of the identity of the user.
 */
export default class Identity {
  constructor(tokenResponse) {
    this.account = tokenResponse.account;
    this.rawIdToken = tokenResponse.idToken.rawIdToken;
  }

  get userId() {
    return this.account.accountIdentifier;
  }

  get emailAddress() {
    return this.account.userName;
  }

  get name() {
    return this.account.name;
  }

  get idToken() {
    return this.rawIdToken;
  }
}
{% endhighlight %}

This brings us to the reducer.  The purpose of the reducer is to create a new version of the state that is modified according to the action.  State is immutable in Redux, so you have to create a new one.

{% highlight javascript %}
const initialState = {
  networkRequests: 0,
  identity: null,
  lastError: null
};

export default function (state = initialState, action) {
  switch (action.type) {
    case NETWORK_ERROR:
      return { ...state, lastError: action.error };
    case NETWORK_START:
      return { ...state, networkRequests: state.networkRequests + 1 };
    case NETWORK_STOP:
      return { ...state, networkRequests: state.networkRequests - 1 };
    case SIGN_IN:
      return { ...state, identity: action.identity };
    case SIGN_OUT:
      return { ...state, identity: null };
    default:
      return state;
  }
}
{% endhighlight %}

## Setting up the UI

I like to abstract away the implementation details for the authentication service.  This means that I don't leak the authentication service implementation out into the rest of the application and can swap it out with a new implementation easily.

The way I do this is to design the abstraction first, then fill in the details.  So, what do I need?

* Some mechanism for the user to sign in interactively.
* Action creators that sign in and out interactively.
* An initialization method to handle silent authentication.

Let's take a look at the action creators first.  I'm using `redux-thunk` for asynchronous action creators.  This allows me to use async/await by returning a function that takes dispatch instead of the normal action.  Code will explain this better:

{% highlight javascript %}
/**
 * Public action for initializing the network module.  Tries to acquire
 * an auth token silently, and swallows an interactive sign-in required.
 */
export function initializeNetwork() {
  return async (dispatch) => {
    try {
      dispatch(startNetwork());
      const identity = await authService.getToken();
      dispatch(networkSignIn(identity));
      dispatch(stopNetwork());
    } catch (error) {
      dispatch(stopNetwork());
      if (!(error instanceof InteractiveSignInRequired)) {
        dispatch(networkError(error));
      }
    }
  };
}

/**
 * Action for initiating an interactive sign-in.
 */
export function signIn() {
  return async (dispatch) => {
    try {
      dispatch(startNetwork());
      const identity = await authService.signIn();
      dispatch(networkSignIn(identity));
      dispatch(stopNetwork());
    } catch (error) {
      dispatch(stopNetwork());
      dispatch(networkError(error));
    }
  };
}

/**
 * Action for initiating a sign-out.
 */
export function signOut() {
  return (dispatch) => {
    dispatch(startNetwork());
    authService.signOut();
    dispatch(stopNetwork());
    dispatch(networkSignOut());
  };
}
{% endhighlight %}

Make sure you add `store.dispatch(actions.initializeNetwork())` when you create your store.  This will dispatch an initialization action, which will then do an asynchronous silent token acquisition. 

In each of these action creators, we return a function.  The `redux-thunk` middleware will execute the function asynchronously, and then the various methods will get called.  The net result of this is that I only have to dispatch a `signIn()` action creator to get the authentication service to interactively sign in.

To initiate that, I've got a `SignInButton` component that I can use anywhere on my UI to allow the user to sign-in or sign-out:

{% highlight javascript %}
import React from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import { actions } from '../../../redux';
import authService from '../../../services/auth-service';
import styles from './SignInButton.module.scss';

const SignInButton = () => {
  const identity = useSelector((state) => state.network.identity);
  const dispatch = useDispatch();

  const onClickHandler = () => {
    dispatch(identity ? actions.signOut() : actions.signIn());
  };

  const buttonText = identity ? 'Sign out' : 'Sign in';
  const longText = `${buttonText} with ${authService.serviceName}`;

  return (
    <button type="button" className={styles.signInButton} onClick={onClickHandler}>
      <FontAwesomeIcon icon={authService.icon} />
      <span className={styles.shortTitle}>{buttonText}</span>
      <span className={styles.longTitle}>{longText}</span>
    </button>
  );
};

export default SignInButton;
{% endhighlight %}

A couple of notes about this component (which, as React components go, is fairly straight-forward):

1. I'm using [React Hooks](https://reactjs.org/docs/hooks-intro.html) to link into the Redux store.  It took me a while to figure out hooks, but once I saw it used with Redux, things fell into place.  I love this syntax now since I don't have to create extra props just to hook up the store.  It feels cleaner.
2. I get the authentication service name and icon from the authentication service (which we will delve into next).  This sort of encapsulation means I can keep a library of authentication services elsewhere and just re-use them.
3. I'm using a responsive layout.  The CSS for this component has a media selector that displays the long version above a certain width (768px in my case), and the short version below that same width.

Taking a quick step back:

* There is a `SignInButton` on the page somewhere.  When clicked, it triggers the `signIn()` action.
* The `signIn()` action triggers an interactive sign-in process.  When the response comes back, the store is updated with the new identity.
* This causes the UI in the `SignInButton` to re-render, switching the text to `Sign out`.
* If there is an error, the `lastError` is set in the store - I can trigger other UI components on this to display the error.

## The authentication service

The only thing remaining is to create the authentication service.  This needs two properties and three methods:

* Property: `serviceName`
* Property: `icon`
* Async method: `getIdentity()` to retrieve the identity silently
* Async method: `signIn()` to sign in interactively
* Method: `signOut()` to sign out

In addition, I need to set up an app registration in Azure Active Directory.  Microsoft login clients are managed through Azure Active Directory.  Sign into your Azure account, then go to [App registrations](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app) and follow the quick start.

The only things you really need to know:

* You are implementing a web client.
* You need a redirect URI that is the same as your web app.  Locally, mine is `http://localhost:3000/modernmud`.  Later on, it will be something else and I'll add a redirect URI to the configuration in the Azure portal at that point.
* You need the application or client ID from the Azure portal.

This last piece of information is placed in a JSON file:

{% highlight json %}
{
  "msal": {
    "authority": "https://login.microsoftonline.com/common",
    "clientId": "YOUR_APPLICATION_ID_HERE",
    "scopes": [ "openid", "profile", "user.read" ]
  }
}
{% endhighlight %}

Replace the `YOUR_APPLICATION_ID_HERE` with the application Id of your app registration from the Azure Portal.  I called this file `config.json`.  I'll be using it to configure the rest of the application as well, so I don't want it to be dedicated to MSAL.

Let's look at the `auth-service.js` which defines my authentication service:

{% highlight javascript %}
import { 
  ClientAuthError, InteractionRequiredAuthError, UserAgentApplication
} from 'msal';
import { faMicrosoft } from '@fortawesome/free-brands-svg-icons';
import { Identity } from '../models';
import { InteractiveSignInRequired } from '../utils';
import config from '../assets/config.json';

class AuthService {
  constructor(configuration) {
    this.signInOptions = {
      scopes: configuration.msal.scopes
    };

    this.msalConfig = {
      auth: {
        authority: configuration.msal.authority,
        clientId: configuration.msal.clientId,
        redirectUri: window.location.href
      },
      cache: {
        cacheLocation: 'sessionStorage',
        storeAuthStateInCookie: true
      }
    };

    this.msalClient = new UserAgentApplication(this.msalConfig);
    console.log('AuthService:: initialized: ', this.msalConfig);
  }

  get serviceName() { return 'Microsoft'; }

  get icon() { return faMicrosoft; }

  async signIn() {
    const response = await this.msalClient.loginPopup(this.signInOptions);
    return new Identity(response);
  }

  signOut() {
    this.msalClient.logout();
  }

  async getIdentity() {
    const account = this.msalClient.getAccount();
    if (account) {
      try {
        const response = await this.msalClient.acquireTokenSilent(this.signInOptions);
        return new Identity(response);
      } catch (error) {
        if (error instanceof InteractionRequiredAuthError) {
          throw new InteractiveSignInRequired();
        }
        if (error instanceof ClientAuthError) {
          // On mobile devices, ClientAuthError is sometimes thrown when we
          // can't do silent auth - this isn't generally an issue here.
          if (error.errorCode === 'block_token_requests') {
            throw new InteractiveSignInRequired();
          }
          console.warn('ClientAuthError: error code = ', error.errorCode);
        }
        throw error;
      }
    }
    throw new InteractiveSignInRequired();
  }
}

const authService = new AuthService(config);

export default authService;
{% endhighlight %}

Most of this is straight-forward, and dictated by the [MSAL.js](https://www.npmjs.com/package/msal) library.  The one that took a little time was the `getIdentity()` method.  This returns an `Identity` object or throws an error.  We want to isolate the rest of the application from the errors that are generated by MSAL, so I have created a new error called `InteractiveSignInRequired()` that I can throw.

There are two situations where this is needed:

1. MSAL doesn't have enough information to get a new token.
2. MSAL doesn't have the right permissions to get a new token.

This latter one takes some explaining.  There are situations (especially in mobile web) where you can't create an iframe to do the transport to get the token from the remote service silently.  In this case, you'll need to re-authenticate using an interactive sign-in process.  The code in the `getIdentity()` method handles both situations, but throws the original error if anything else happens.

## Wrap up

This is a fair amount of code, and there is still more to do.  I've only just managed to get an identity token, so I know about the user, but I haven't provided any permissions to my API as yet.  However, my app can now progress to the next stage.
