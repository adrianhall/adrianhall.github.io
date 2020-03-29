---
title: "Introducing Zwift Routes - my latest app"
categories:
  - JavaScript
  - React
---

I wrote an app for figuring out which routes I need to ride on Zwift so that I can collect a badge.  It's optimized for a phone device (since that is where I used it).  You can use it for free.

## Where to get it.

You can find it at [Zwift Routes](https://adrianhall.github.io/zwift-routes) or [http://bit.ly/zwiftroutes](http://bit.ly/zwiftroutes).  Point your mobile phone browser to the site, then pin the site to your home screen.

![]({{ site.baseurl }}/assets/images/2020/2020-03-13-image1.png){: .center-image}

## How to use it

There are only two screens - the list screen and the details screen.  At the top of the list screen is a menu button that allows you to filter the list.  You can filter based on the world (and sport, then decide to include Watopia rides (since it is always available), event only rides, and completed rides.  You can sort the list across several dimensions, and decide what units (metric or imperial) to use.  I've included "route distance / elevation" and the "total distance / elevation" (which includes the lead-in), so you can decide whether you have time or not.

On the details page (obtained by clicking a route on the list page), you get several stats.  It will show a link to the Zwift Insider route (where you can get the VeloViewer route profile) and whether you have completed it or not.  Click on the grey circled X to mark the route as completed.

That's all there is to it!

## The future

I have several feature requests, including a "desktop" version and native Android / iOS apps.  If you have an idea for the app, let me know through [the issues](https://github.com/adrianhall/zwift-routes/issues).

## For the techies

The application is written in React, with the following libraries:

* [React Router](https://reacttraining.com/react-router/) for page routing.
* [Redux](https://redux.js.org/) for state management, using React Hooks.
* [Redux-persist](https://github.com/rt2zz/redux-persist) for persisting the settings.
* [Dexie](https://dexie.org/) for persisting the "route state" (i.e. completed)
* [Material-UI](https://material-ui.com/) for the UI components
* [Github Pages](https://pages.github.com/) for hosting.

Underneath, it's a fairly reasonable app, started with `create-react-app`.  Here are some of the problems I needed to solve along the way.

### Use pure components

One of the things I wanted to sort out in my mind first was the structure of the app.  I felt that having the lower level components be "pure" was a good idea.  Pure components don't hook into the state store, nor do they understand anything about the routing mechanism.  They just display the data and bubble the interactive events up to the top level.  Thus, for instance, I have a `RouteListItem` component that just displays the list components.  When a user clicks on the list item, it triggers an event that the parent component passes down to it as props.  All the routing and state management is done at the 
top level:

```javascript
const RouteListItem = ({ displayUnits, onClick, route }) => {
  const colors = route.isCompleted ? { color: green[500] } : color: grey[500];
  const icon = route.isCompleted
    ? <CompletedIcon fontSize="large" style={colors} />
    : <NotCompletedIcon fontSize="large" style={colors} />;

  const distance = fmt.formatDistance(route.routeDistance, displayUnits);
  const elevation = fmt.formatElevationGain(route.routeElevationGain, displayUnits);
  const difficulty = route.difficulty.toFixed(2);
  const secondaryText = `${distance}, ${elevation}, difficulty ${difficulty}`;

  return (
    <ListItem button divider onClick={onClick}>
      <ListItemAvatar>{icon}</ListItemAvatar>
      <ListItemText secondary={secondaryText}>
        <Typography variant="h6">{route.routeName}</Typography>
      </ListItemText>
    </ListItem>
  );
};

RouteListItem.propTypes = {
  displayUnits: PropTypes.oneOf(['imperial', 'metric']),
  onClick: PropTypes.func.isRequired,
  route: PropTypes.shape(ZwiftRoutePropTypes).isRequired
};

RouteListItem.defaultProps = {
  displayUnits: 'metric'
};

export default RouteListItem;
```

The nice thing about pure components is that they can be tested individually.  I don't need to provide very many mock services to test the components.

### Deploying to Github pages

I deploy the app to GitHub pages.  This has a small wrinkle when used with React Router.  You have to explicitly use the `HashRouter` instead of the `BrowserRouter`.  It's a simple change, but I lost maybe a half hour trying to figure this out.  (Yes, Stack Overflow saved me, but sometimes you need to figure it out yourself rather than being told the answer).

### Booting up the app

When I load the app up, I need to load the routes database and the route state.  The routes database is a HTTP `fetch` call.  The route state is in IndexedDB (more on that in a moment).  Doing this asynchronously means that the initial state of my route store is `[]` - no routes.  If you bookmark a route detail page (or you refresh when on the route detail page), it didn't know what to do and crashed.

To get around this, I provided a simple wrapper around the main application.  If there are no routes, then it displays a loading screen.  If there are routes, then it progresses to the next stage of showing the UI.  This allows me to block the crash from ever happening.

### Persisting state

The [redux-persist](https://www.npmjs.com/package/redux-persist) library is great for settings.  In fact, it's pretty good any time you want to load and save a complete section of the redux store.  However, if you want to store only partials, then you need something else.  My way feels really hacky.  First off, I created a `route-state-client`:

```javascript
import Dexie from 'dexie';

class RouteStateService {
  constructor() {
    this.database = new Dexie('zwiftroutes');
    this.database.version(1).stores({
      routeState: '&routeId,isCompleted'
    });
  }

  async loadRouteState() {
    const response = await this.database.routeState.toArray();
    return response;
  }

  async saveRouteState(action) {
    const obj = {
      routeId: action.routeId,
      isCompleted: action.routeUpdate.isCompleted
    };
    await this.database.routeState.put(obj);
  }
}

const routeServiceClient = new RouteStateService();

export default routeServiceClient;
```

This is a singleton.  When the app loads, it called `loadRouteState()`.  This returns a set of objects that look like this:

```json
[
  { "routeId": "some-uuid", "isCompleted": true },
  { "routeId": "some-other-uuid", "isCompleted": true }
]
```

This is then fed into the redux store as a load event sequentially, setting an `isLoading` flag to ensure that the application doesn't save the events back again.  If the `routeServiceClient` was a network service, then that would result in additional network traffic and backend consumption.  

On the save side, I have a piece of middleware:

```javascript
import { UPDATE_ROUTE_ACTION } from './reducers/routes';
import routeServiceClient from '../services/route-service';

/**
 * Redux middleware that saves the state of the routes to
 * the route state service when it changes.
 */
const saveRoutesMiddleware = () => (next) => async (action) => {
  const result = next(action);

  if (action.type === UPDATE_ROUTE_ACTION && action.isLoading !== true) {
    await routeServiceClient.saveRouteState(action);
  }

  return result;
};

export default saveRoutesMiddleware;
```

This looks for the route update actions and saves them once they make it through the redux dispatch process.  The save only happens when the `isLoading` flag is not set (i.e. when they aren't produced by a load operation).

As I said, this feels hacky.  I think I want to refactor this into a "RouteStore" that brings together all the potential network operations.  This would greatly simplify understanding the flow that gets the data where it needs to go.  I think having a "SettingsStore" would also ensure that the right things happen.

### Anything else?

This was my first big foray into React programming for a while, and I was impressed by how easy React Hooks was to use.  The impressive thing (at least for me) was how I could define the 
properties and then hook into the redux store rather than having to wrap the component and inject everything by properties.

As alway, my code is open-source and MIT licensed.  You can get it from [my repository](https://github.com/adrianhall/zwift-routes).

Enjoy!
