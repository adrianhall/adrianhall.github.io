---
title: "Building a GraphQL API by Example: Restaurant Reviews (part 1: The Schema)"
categories:
  - AWS
tags:
  - GraphQL
---

I’m loving [GraphQL](https://graphql.org/) for my apps. It removes the ever shifting requirements that I implemented when I was doing “REST-like” backends and removes the complexity of having to do BFF ([Backend for your Frontend](https://samnewman.io/patterns/architectural/bff/)) and the maintenance headaches that come with such architectures.

Developing a GraphQL backend requires four steps:

1. Design a GraphQL schema that you will expose to the apps.
2. Design and implement a backend data store schema to store the data.
3. Implement resolvers to translate #1 to #2.
4. Implement authorization requirements for the API.

There is lots of information on how to develop data store schemas, irrespective of whether you are using SQL, NoSQL, more modern databases such as graph databases, or other services such as data stored in CRM applications. The backend is generally less of a concern.

But how do you start developing a GraphQL schema?

## My process

There is no “best practice” for developing a GraphQL schema. However, when I am working on an app, I’m generally working with the frontend and backend code concurrently. So here is what I do:

1. Work out what primitive data types I need.
2. Adjust anything that returns a list into a “paging” query.
3. Determine operations an app might reasonably require.
4. Write input types required for queries.
5. Determine real-time update requirements for user experience.
6. Determine authorization requirements for security.

To illustrate this, I’m going to take an example and work it through from the beginning. Simple apps (like chat apps, note taking apps, and so on) have been done ad nauseum, so I’m going to tackle something different — a restaurant review app.

## Step 1: The primitive types

When I’m thinking about the app, I want to do several things:

* As a restaurant owner, I want to be able to add a restaurant to the database.
* As an authenticated user, I want to be able to add a review to a restaurant listing with a rating.
* As an authenticated user, I want to be able to mark a restaurant as a favorite.
* As an authenticated user, I want to see a list of my favorite restaurants.
* As an unauthenticated user, I want to search for restaurants and see their average rating.
* As an authenticated user, I want to read the reviews of a particular restaurant.

These requirements have three basic types:

* A _User_ represents me (or another user) and I should be able to get lists of restaurants that I have added, restaurants that I have marked as a favorite, and reviews that I have written.
* A _Location_ represents a restaurant and I should be able to get the reviews for this location.
* A _Review_ is written by a user and has a specific rating and content.

Put into basic GraphQL:

```graphql
type User {
  id: ID!
  name: String!
  email: String
  locations: [Location]
  reviews: [Review]
  favorites: [Location]
}

type Location {
  id: ID!
  owner: User!
  name: String!
  longitude: Number
  latitude: Number
  address: String
  averageRating: Number
  favoritesCount: Number
  reviews: [Review]
}

type Review {
  id: ID!
  owner: User!
  location: Location!
  content: String!
  rating: Number!
}
```

Note that all three are interdependent. Each type has a linkage to the other two types. If I were doing a SQL schema, I would just be referencing the ID. However, in this case, I’m embedding the type directly so that I can do queries. For example, I could do:

```graphql
query getUser(id: ID!) {
  name
  email
  favorites {
    id
    name
    averageRating
  }
}
```

That gets my favorite restaurants and their rating easily. I can do similar things for locations and reviews. The flexibility of embedding the types instead of just sending the IDs translates to increased flexibility when developing the front end.

## Step 2: Paging Support

Right now, my types are returning arrays of types. When talking about front end development, this isn’t what is required. Front end developers want to only load what is shown on the screen, allowing for infinite scroll with minimal data transfer. To do this, I need to add paging support for the API.

There are basically two mechanisms you can use for this:

* A _limit_ and _nextToken_ allow you to start at the beginning and move page by page to the end. The _nextToken_ is a marker for the next page. In the client, you keep the previous pages cached so you can easily traverse the list.
* A _start_ and _end_ allow you to jump to any point in the list. This needs to allow for what happens when you specify out of bounds numbers and needs backend support for random access within the list.

Inevitably, the _limit/nextToken_ is more widely supported that _start/end_, so I recommend you use that mechanism unless there is a good reason (and backend data store support for) using _start/end_.

To implement:

* Create a _PagingConnection_ type for each primitive type that returns the items plus the nextToken.
* Update the elements that return an array to take a _PagingRequest_ object and return the relevant _PagingConnection_ type.

Let’s take an example of the _User_ type. Here is how I modified it:

```graphql
input PagingRequest {
  limit: Number
  nextToken: String
}

type User {
  id: ID!
  name: String!
  email: String
  locations(paging: PagingRequest): LocationPagingConnection
  reviews(paging: PagingRequest): ReviewPagingConnection
  favorites(paging: PagingRequest): LocationPagingConnection
}

type LocationPagingConnection {
  items: [Location]
  nextToken: String
}

type ReviewPagingConnection {
  items: [Review]
  nextToken: String
}
```

Note that the _limit_ and _nextToken_ are both optional in the _PagingRequest_. If _limit_ is not specified, then the backend chooses the number of elements to return. If _nextToken_ is not specified, the first page is requested. Similarly, if the _nextToken_ is not returned, then there are no more pages to be returned.

## Step 3: Determine operations

This is probably the area that I most often return to when developing an app, as it is driven by the requirements of the front end. However, I can see some operations:

* Get “my user” record (which will include the locations, reviews, and favorites under my record)
* Search for a location
* Add a location
* Add a review
* Mark a location as favorite

So that is three mutations and two queries. However, when I think about the search for a location, I can see two possibilities: searching by GPS coordinates, and searching by some name or address, so we will need to handle both of those:

```graphql
type Query {
  me: User!
  searchForLocation(byGPS: GPSInput, byAddress: AddressInput): LocationPagingConnection
}

type Mutation {
  addLocation(location: LocationInput): Location
  addReview(review: ReviewInput): Review
  addFavorite(locationId: ID!): Location
}
```

I’ve added new types here — several of them. These are [input types](https://graphql.org/graphql-js/mutations-and-input-types/) and they enable me to request different things from the user in an expansive manner. Right now, they are just placeholders to say “I need some parameters here”. I’ve already defined one input type — for paging support.

## Step 4: Write input types

There are a few reasons for using input types to specify parameters to mutations:

* You should never pass information that can be inferred from the context of the request, so the input type will be different from the returned type.
* You may want to use the same input type for several mutations (for example, using the same input type for create and update mutations).
* The number of parameters that are to be supported is large, causing readability issues within the query.

Whatever the reason, I recommend using input types for just about everything. I’ve required four input types in step 3, and now need to define them:

```graphql
input GPSInput {
  longitude: Number
  latitude: Number
  radius: Number
}

input AddressInput {
  street: String
  city: String
  state: String
  zipcode: String
}

input LocationInput {
  name: String
  address: AddressInput
}

input ReviewInput {
  locationId: ID!
  content: String!
  rating: Number!
}
```

note that I am reusing the _AddressInput_ in the _LocationInput_ — re-use like this is fully supported and reduces the size of the schema, which is a good thing.

Could there be enhancements? Sure! For example, I might want to search by name where the name “begins with” or “ends with” a certain string. That concept is probably supported by the backend, but it isn’t supported by the schema right now. If you want to see a good example of how to support this, check out the schema that is generated by [AWS Amplify model transforms]({% post_url 2018-08-29-build-a-graphql-service-the-easy-way %}).

## Step 5: Real-time Data Delivery

Adding real-time data delivery to your apps increases the “liveness” of the data, making your app more engaging. It does this by delivering updates to the app via a subscription that is communicated over a websocket. You need to decide which operations should be communicated immediately to clients, concentrating on those that provide an improvement in user experience.

For example, if I am looking at a location in the app, should I be immediately notified of a rating change or a new review? This is a feature that will show that the app is being used and the data is fresh, so it will be a net improvement to the user experience.

If I am browsing a list of locations on the web and mark one as a favorite, should that show up immediately on my mobile device? This might not be a core use case as the device can refresh the favorites list automatically when it wakes up. It’s unlikely the user will be looking at their phone while using the same app on the web.

In this case, I want to update the location being viewed by a user when it is updated since that includes the ratings and favorites count:

```graphql
type Subscription {
  updatedLocation(locationId: ID!): Location
  @aws_subscribe(mutations: [ "addReview", "addFavorite" ])
}
```

Note that here I’ve included a _locationId_ to indicate that I want to watch a specific location. This means going back to ensure that _addReview_ and _addFavorite_ are both asking for the _locationId_ so that it is tracked appropriately. You will see how I do this in the final version.

I’m using [AWS AppSync] for my subscriptions. If you are using a different GraphQL server, then you will have to refer to the documentation on how to set up real-time subscriptions.

## Step 6: Authentication and Authorization

Now that real-time is done, I need to work on authorization. This depends on the GraphQL server and the capabilities of that server. For example, I’m using AWS AppSync and the AWS_IAM authentication model. I’m doing this because I want both unauthenticated and authenticated access. As a result, some queries are available unauthenticated, and some queries are available only to authenticated users. All the mutations are available only to authenticated users, and I want to restrict subscriptions to authenticated users to ensure costs are kept low for the app.

In my case this is all done in IAM (and I’ll cover this topic in another blog post). If you want to learn more about authentication and authorization options in AWS AppSync, see [my blog on the topic]({% post_url 2018-06-01-how-developers-can-auth-with-aws-appsync %}).

## The final schema

So, what does the final schema look like?

```graphql
input PagingRequest {
  limit: Number
  nextToken: String
}

type User {
  id: ID!
  name: String!
  email: String
  locations(paging: PagingRequest): LocationPagingConnection
  reviews(paging: PagingRequest): ReviewPagingConnection
  favorites(paging: PagingRequest): LocationPagingConnection
}

type Location {
  id: ID!
  owner: User!
  name: String!
  longitude: Number
  latitude: Number
  address: String
  averageRating: Number
  favoritesCount: Number
  reviews(paging: PagingInput): ReviewPagingConnection
}

type Review {
  id: ID!
  owner: User!
  location: Location!
  content: String!
  rating: Number!
}

type LocationPagingConnection {
  items: [Location]
  nextToken: String
}

type ReviewPagingConnection {
  items: [Review]
  nextToken: String
}

input GPSInput {
  longitude: Number
  latitude: Number
  radius: Number
}

input AddressInput {
  street: String
  city: String
  state: String
  zipcode: String
}

input LocationInput {
  name: String
  address: AddressInput
}

input ReviewInput {
  content: String!
  rating: Number!
}

type Query {
  me: User!
  searchForLocation(byGPS: GPSInput, byAddress: AddressInput): LocationPagingConnection
}

type Mutation {
  addLocation(location: LocationInput): Location
  addReview(locationId: ID!, review: ReviewInput): Review
  addFavorite(locationId: ID!): Location
}

type Subscription {
  updatedLocation(locationId: ID!): Location
  @aws_subscribe(mutations: [ "addReview", "addFavorite" ])
}

schema {
  query: Query
  mutation: Mutation
  subscription: Subscription
}
```

Methodically building up the schema likes this allows you to make decisions quickly based on the requirements of the apps.

There are topics that I have not addressed here:

* How to handle versioning of the API?
* How to handle deprecated fields?
* How to manage combining multiple services together into a coherent single API?

These are all good topics and ones I intend to explore in the future. GraphQL is still a young API and I fully expect it to evolve over time. As it evolves, these questions will get answered, with advice, enhancements, and best practices being documented along the way.

{% include links.md %}
