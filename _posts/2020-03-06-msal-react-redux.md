---
title: "Integrating Microsoft Login and MSAL with React and Redux"
categories:
  - JavaScript
  - React
---

I have a new app I am working on.  It's sort of a 1990's style text MUD, but I'm bringing it "up to this century" with a host of new features.  I'm writing the first front-end in React.  One of the problems - how to deal with signing in with my Microsoft account.

## Step 1: The App Registration

Microsoft accounts are managed through Azure Active Directory.  Sign into your Azure account, then go to [App registrations](https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app) and follow the quick start.

The only things you really need to know:

* You are implementing a web client.
* You need a redirect URI that is the same as your web app.  Locally, mine is `http://localhost:3000/modernmud`.  Later on, it will be something else and I'll add a redirect URI to the configuration at that point.

## Step 2: Create an MSAL configuration

I've create my React app with `create-react-app`.  Then I created a configuration file:

{% highlight json %}
{
  "msal": {
    "authority": "https://login.microsoftonline.com/common",
    "clientId": "YOUR_APPLICATION_ID_HERE",
    "scopes": [ "openid", "profile", "user.read" ]
  }
}
{% endhighlight %}

Replace the `YOUR_APPLICATION_ID_HERE` with the application Id of your app registration from the Azure Portal.  I called this file `config.json`.  I'll be using it to configure the rest of the application as well.

## Step 3: Create an authentication service

I like to abstract out the common code into reusable units.  One such unit is the "auth service".  Most of it is fairly easy to follow:

{% highlight javascript %}
import {
  ClientAuthError,
  InteractionRequiredAuthError,
  LogLevel, Logger,
  UserAgentApplication
} from 'msal';
import { Identity } from '../models';
import { InteractiveSignInRequired } from '../utils';
import config from '../assets/config.json';

/**
 * Logger method for the MSAL diagnostic logs.
 *
 * @param {LogLevel} logLevel
 * @param {string} message
 */
const msalLogger = (logLevel, message) => {
  switch (logLevel) {
    case LogLevel.Verbose:
      console.debug(`MSAL: ${message}`);
      break;
    case LogLevel.Info:
      console.info(`MSAL: ${message}`);
      break;
    case LogLevel.Warning:
      console.warn(`MSAL: ${message}`);
      break;
    case LogLevel.Error:
      console.error(`MSAL: ${message}`);
      break;
    default:
      console.log(`MSAL: ${message}`);
      break;
  }
};

/**
 * Configuration for the MSAL library.
 */
const msalConfig = {
  auth: {
    authority: config.msal.authority,
    clientId: config.msal.clientId,
    redirectUri: window.location.href
  },
  cache: {
    cacheLocation: 'sessionStorage',
    storeAuthStateInCookie: true
  },
  system: {
    logger: new Logger(msalLogger, { level: LogLevel.Verbose })
  }
};

/**
 * Configuration for operations that return a token.
 */
const signInOptions = {
  scopes: config.msal.scopes
};

console.debug('auth-service:: initializing MSAL: config = ', msalConfig);
const msalClient = new UserAgentApplication(msalConfig);
{% endhighlight %}

This is all setup.  I'm reading in the configuration file, creating the configuration object required by MSAL, and then creating the client.  I've added a little bit of code to do debug logging along the way.

Next, let's take a look at the `signIn()` and `signOut()` methods.  These will be called when the user clicks on a button to do something, so it's ok to do interactive stuff:

{% highlight javascript %}
/**
 * Acquire a token interactively.
 */
export async function signIn() {
  console.debug('Initiating interactive sign-in');
  const response = await msalClient.loginPopup(signInOptions);
  console.debug('tokenResponse = ', response);
  return new Identity(response);
}

/**
 * Sign out of the account
 */
export function signOut() {
  console.debug('Initiating MSAL sign-out');
  msalClient.signOut();
}
{% endhighlight %}

I'm using async/await instead of promises.  I like how this simplifies and improves the readability of the code.  

There is one method I have left until last.  It's the `getToken()` method.  I expect this to be run on a regular basis to get tokens, so it's the heaviest component:

{% highlight javascript %}
/**
 * Acquire a token asynchronously (and silently).
 *
 * @returns an Identity object
 * @throws InteractiveSignInRequired if the UI needs to be presented.
 * @throws Error on all other errors.
 */
export async function getToken() {
  const account = msalClient.getAccount();
  if (account) {
    try {
      const response = await msalClient.acquireTokenSilent(signInOptions);
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
{% endhighlight %}

There are a couple of reasons why an interactive sign-in is required.  The first is "because MSAL said so".  The second is because the features of the platform do not allow silent token requests.  The exception handling provides for both cases.  If there is an alternate `ClientAuthError`, then I print out the `errorCode`.  This allows me to decide if the error needs to be swallowed or thrown.

The `InteractiveSignInRequired()` class is a custom error class:

{% highlight javascript %}
export default class InteractiveSignInRequired extends Error {
  constructor() {
    super('Interactive Sign In Required');
    this.name = 'InteractiveSignInRequired';
  }
}
{% endhighlight %}

Finally, both `getToken()` and `signIn()` return the identity of the user.  It's a simple model, but I only store the bits I need:

{% highlight javascript %}
export default class Identity {
  constructor(tokenResponse) {
    this.account = tokenResponse.account;
    this.idToken = tokenResponse.idToken;
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
    return this.idToken.rawIdToken;
  }
}
{% endhighlight %}

I can use this authentication service module in other applications, including non-React apps.  It's a good abstraction of the functionality I need.

## Step 4: Create a redux reducer and actions

I've already got a redux store, and it's integrated into my app already.  There are plenty of tutorials for creating a redux store.  The important thing is to ensure that you integrate `redux-thunk` so you can do asynchronous actions.  The reducer is simple enough:

{% highlight javascript %}
import {
  NETWORK_ERROR,
  NETWORK_START,
  NETWORK_STOP,
  SIGN_IN,
  SIGN_OUT
} from '../actions/network';

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

I've got three things I want to track - errors, whether there is a network request in flight, and the current identity of the sign-in user.  I've got five actions that are possible to update these various values.  

The actions (which will be dispatched) match up to the three async methods in the authentication service:

{% highlight javascript %}
import * as authService from '../../services/auth-service';
import { InteractiveSignInRequired } from '../../utils';


export const NETWORK_ERROR  = Symbol.for('network.error');
export const NETWORK_START  = Symbol.for('network.start');
export const NETWORK_STOP   = Symbol.for('network.stop');
export const SIGN_IN        = Symbol.for('network.auth-sign-in');
export const SIGN_OUT       = Symbol.for('network.auth-sign-out');

const startNetwork          = () => ({ type: NETWORK_START });
const stopNetwork           = () => ({ type: NETWORK_STOP });
const networkError          = (error) => ({ type: NETWORK_ERROR, error });
const networkSignIn         = (identity) => ({ type: SIGN_IN, identity });
const networkSignOut        = () => ({ type: SIGN_OUT });

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

Redux-thunk takes the effort out of async actions.  A thunk is an action creator that returns a function.  The `dispatch` for the store is passed to the action creator so that the async operation can happen.  With these, I can just `dispatch(signIn())` to initiate the sign-in process, as an example.

## Step 5: Create a sign-in button

Here is the sign-in button code:

{% highlight javascript %}
import React from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import { faMicrosoft } from '@fortawesome/free-brands-svg-icons';
import { actions } from '../../../redux';
import styles from './SignInButton.module.scss';

const SignInButton = () => {
  const identity = useSelector((state) => state.network.identity);
  const dispatch = useDispatch();

  const onClickHandler = () => {
    dispatch(identity ? actions.signOut() : actions.signIn());
  };

  const buttonText = identity ? 'Sign out' : 'Sign in';
  const longText = `${buttonText} with Microsoft`;

  return (
    <button type="button" className={styles.signInButton} onClick={onClickHandler}>
      <FontAwesomeIcon icon={faMicrosoft} />
      <span className={styles.shortTitle}>{buttonText}</span>
      <span className={styles.longTitle}>{longText}</span>
    </button>
  );
};

export default SignInButton;
{% endhighlight %}

There have been two big changes in the React world over the past few years, and I love both of them.  The first is functions as components.  A lot of components were very simple with no state, and using functions really made them a lot simpler to write.  The other change was [React Hooks](https://reactjs.org/docs/hooks-intro.html).  It took me a little while to get used to hooks, but it all fell in place when I used them with Redux.  Instead of having to define a method that maps the state to props, define props, then wire up the store via the context with `connect` (which sounds a lot worse than what actually ended up happening), I add two function calls to my component.  I don't have to use classes for this.  The state is in the Redux store.

The logical equivalent of `mapStateToProps()` is `useSelector()`.  You basically say "this variable is stored in the state of the store here..." and React Hooks takes care of it.  `connect()` also passed in dispatch, and you can replace that with `useDispatch()`.  The nice thing is that you can keep the props that you want a higher level component to pass in separate from the things you want the store to provide.  

I made the button responsive.  There is a media query that says "on smaller screens, shorten the button title."  This is done by hiding the long text and showing the short text.

## So what happens?

Final thought, don't forget to `store.dispatch(initializeNetwork())` to get MSAL configured when you create your store!

When you start your application, the authentication service will be initialized and the service will attempt to acquire a token silently.  It's not a big deal is the service can't acquire a token.  it just means the user has to click on the sign-in button.

Place the `<SignInButton/>` somewhere on your page for the user to click.  When the user does click the `<SignInButton/>`, it will try to use the MSAL sign-in process.  This will pop-up a window, asking you to sign-in.  Once you've gone throuhg that process, the store will have an identity and the button will say 'Sign out' instead.

Along the way, you can put up a spinner when network operations are in progress by listening to the `networkRequests` value in the store, and put up an error sign when `lastError` is true.  I'll probably add an ability to clear the error as well when I do this.

One thing that hasn't happened yet?  I haven't got an `accessToken` - only an `idToken`.  To get an access token, I need to adjust the configuration of my app registration and add a scope to the list of scopes in the configuration.  For more information, see [Permissions and consent](https://docs.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent) in the Microsoft Identity documentation.


