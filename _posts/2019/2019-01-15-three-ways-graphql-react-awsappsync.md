---
title: "The three ways to execute a GraphQL query from React with AWS AppSync (and how to choose)"
categories:
  - Cloud
tags:
  - aws_appsync
  - graphql
---

[AWS AppSync] is a managed [GraphQL] service that can (and probably should) act as the data layer for your app. I want to take a look at how you can send a query to AWS AppSync from your React (or React Native) app.

You have three basic choices:
* Include a query within a component using [AWS Amplify].
* Wrap your query in the Connect component using AWS Amplify.
* Use the [Apollo Client](https://www.apollographql.com/docs/react/) with the AWS AppSync SDK.

Which do you choose depends on what your needs are. There is no "one size fits all".

Let's look at the options:

## Include a query within a component using AWS Amplify

The first version of the query utilizes the `API.graphql()` method from the [AWS Amplify library](https://aws-amplify.github.io/docs/js/api#amplify-graphql-client). You can execute queries, mutations, and subscriptions from this form. It's an async network call, so expect to deal with promises and errors. Here is the canonical form of a simple query:

{% highlight js %}
    public async componentWillMount() {
        try {
            const result = await API.graphql(graphqlOperation('{ me { id name } }'));
            console.log('componentWillMount: result = ', result);
            this.setState({ loading: false, data: result.data.me });
        } catch (err) {
            this.setState({ loading: false, errors: [ err.message ] });
        }
    }
{% endhighlight %}

There are a couple of notes here:

1. I'm using async/await to ensure I wait for the data.
2. I must use try/catch so that I can catch network and authentication errors.
3. The result has a data block with the fields that I requested:

    {% highlight js %}
    result = Object {
      "data": Object {
        "me": Object {
          "id": "ARO*****:CognitoIdentityCredentials",
          "name": null
        }
      }
    }
    {% endhighlight %}

All the fields you requested are returned, but the value may be null if no data is available.  If I've got a component that has to get data from the network or I am using a library to do all the calls (and mocking those calls for testability), this is a great way to do it. However, if I decide to incorporate the call within a component, I have to worry about how I am going to test that component. I'd more likely place the call within a library and swap out the library with a mocked call at that point.

I also have to deal with online/offline capabilities. The AWS Amplify library does not do offline at this point in time, so if I want that functionality, I'm going to have to go to a different library.

Pros:
* It's simple to set up an execute the query, mutation, or subscription.
* It works with the AWS Amplify CLI and configuration file, so configuration is a snap.
* It works easily with my preferred Flux implementation.
* Simplified codebase = less stuff to go wrong.

Cons:
* You will need to be careful to ensure that your components can be unit tested.
* There is no offline capabilities.

## Wrap your query in the AWS Amplify Connect component

The next step is to use the [`<Connect>` component](https://aws-amplify.github.io/docs/js/api#amplify-graphql-client) to wrap my component. The `<Connect>` component will give you loading and error conditions, so you can use those to handle network conditions. Let's say I have a component that is normally used like this:

{% highlight js %}
<UserBlock name={"Adrian Hall"}/>
{% endhighlight %}

I want to use the [`<Connect>` query](https://aws-amplify.github.io/docs/js/api#connect) to connect this to the me query that I was using before. I might do this:

{% highlight js %}
const UserBlockFunction = () => {
    return (
        <Connect query={graphqlOperation('{ me { id name } }')}>
            {(response) => {
                if (response.loading) {
                    return (<LoadingIndicator loading={response.loading}/>);
                } else if (response.data && response.data.me) {
                    return (<UserBlock name={response.data.me.name}/>);
                } else {
                    return (<ErrorIndicator errors={response.errors}/>);
                }
            }}
        </Connect>
    );
};

export default UserBlockFunction;
{% endhighlight %}

In this version, I've got three cases:
1. The query is loading (loading == true)
2. The query has returned data (response.data.me is defined)
3. The query ended in one or more errors (response.errors is defined)

I can use this to generate different output within my exported function for each condition.  I generally have the underlying components as one set of files, then I connect them to the GraphQL API as another set of files.

Pros:
* I can test my underlying components individually without resorting to network connectivity.
* I can hook individual parts of my component hierarchy as needed, resulting in much flexibility.
* The API is powerful, yet simple. That leads to elegant and readable code that is easy to debug.

Cons:
* It no longer works with my Flux configuration, so I now have two state systems to deal with.
* Still no offline support.

## Use the Apollo Client with the AWS AppSync SDK.

The final method is to bring in a heavyweight client like the Apollo Client. It took me some time to learn the Apollo Client and it's overkill for most situations. 

In this method, you create an AWS AppSync Client, then use that to configure the Apollo Client. Then wrap your entire app within the Apollo Client. Your connected components now have full knowledge of the Apollo Client, but your lower level components can remain oblivious (just like the `<Connect>` component I discussed above). Let's look at the same functionality as before. First, configure the client within your main app code:

{% highlight js %}
import gql from 'graphql-tag';
import AWSAppSyncClient, { AUTH_TYPE } from 'aws-appsync';
import aws_config from './aws-exports';
import App from './src/App';

const client = new AWSAppSyncClient({
  url: aws_config.aws_appsync_graphqlEndpoint,
  region: aws_config.aws_appsync_region,
  auth: {
    type: aws_config.aws_appsync_authenticationType,
    apiKey: aws_config.aws_appsync_apiKey,
  }
});

const WithProvider = () => (
  <ApolloProvider client={client}>
    <Rehydrated>
      <App />
    </Rehydrated>
  </ApolloProvider>
);

export default WithProvider;
{% endhighlight %}

Then create the connected component:

{% highlight js %}
import gql from 'graphql-tag';
import { graphql } from 'react-apollo';
import * as React from 'react';
import UserBlockComponent from '../components/UserBlock';

const query = gql`
  query me {
    me { id name }
  }
`;

class UserBlock extends React.Component {
  render() {
    if (this.props.loading) {
      return (<LoadingIndicator/>):
    } else if (props.errors) {
      return (<ErrorIndicator errors={this.props.errors}/>);
    } else {
      return (<UserBlockComponent name={this.props.name}/>);
    }
  }
}

export default graphql(query, {
  options: { 
    fetchPolicy: 'cache-and-network'
  },
  props: props => ({
    loading: props.loading,
    errors: props.errors,
    name: props.data.me.name || 'undefined'
  })
})(UserBlock);
{% endhighlight %}

More power, but more complexity and more places for things to go wrong.

Pros:
* Offline capabilities are available. Queries are cached and mutations are queued for later transmission
* I can test my underlying components individually without resorting to network connectivity.

Cons:
* This is a complex client that will take time to learn fully.
* Offline can introduce caching bugs (such as stale data) that just didn't exist before.
* I have to wrap a good portion of my app in the Apollo client, replacing the Flux implementation (or at least making it harder to implement and follow the data flow).

## Wrap Up

Here is the basic version that covers the advice I would give as of this writing:

* Use the Apollo Client with the AWS AppSync SDK if you need offline capabilities.
* Wrap your component in the AWS Amplify Connect component for the majority of online-only cases, then use `API.graphql()` for the mutations to send data to the server.
* Use `API.graphql()` only if you want to do a query outside of a React component.
* Keep an eye on the AWS Amplify library as they are always extending the functionality of the client.

{% include links.md %}
