# Step 13: Authentication

[//]: # (head-end)


In the previous step we’ve set the ground for the authentication system in our app. We have a users collection which can be used to distinguish which data the client is authorized to view, and we have a context handler which can retrieve the current user logged in based on the received value of the `cookie` header. It’s definitely a good starting point, but it misses a lot of things.

In this chapter we will implement a cookie-based authentication system. There are many ways to implement an authentication system in an app, but cookie-based authentication is one of the most popular ones, hence we will go with that method. Essentially the authentication flow in our app should look very simple: a user will be able to sign-in with a dedicated screen, and if he doesn’t own an account he can use the sign-up screen to create a new one. The more complicated part in this flow is the back-end, which is the core of this chapter. So before we get into the implementation, we need to understand the authentication process:

- A user logs in with a username and a password.
The server compares the received username and password to the ones stored in the database.
- If the comparison was successful, the server will generate a token and will set it as a cookie.
- Each time a request is sent, the server will retrieve the username from the stored token on the cookie header and will send data back accordingly.

![auth-flow](https://user-images.githubusercontent.com/7648874/55929679-55e94200-5c50-11e9-9fe7-54ad6194a572.png)

The stored token will save us the hassle of re-specifying the username and password over and over again each and every request. It’s important to note that everything in the authentication process is encrypted, **sensitive information will never be stored or sent in its raw form**, otherwise data might be stolen in case of a DB breach or a request hijacking. This is what it means for our app:

- Passwords will always be stored in an encrypted form in the DB using an algorithm called [Bcrypt](https://en.wikipedia.org/wiki/Bcrypt). Bcrypt has the ability to compare the password in its raw form to the encrypted one, which can help us authorize the user.

- Tokens are self contained. That means that once we decode the encrypted string we can get a hold of the username string. This form of encrypted tokens is called [Json Web Token (JWT)](https://jwt.io/).

> We're not going to elaborate about the algorithm behind each encryption method because we want to focus more on practicality, although it's very much recommended to understand how each method works before proceeding.

The implementation will follow the principles above. Authentication is a hot topic in the GraphQL world and there are several ways of doing so. We will start with the back-end and set the infrastructure for authentication, and then we will move on to the front-end.

We’re gonna expose 2 new mutations from GraphQL Schema: `signIn` and `signUp`. `/sign-out` is unnecessary because it can be done locally by deleting the right cookie. Our back-end is gonna grow bigger so first we will separate the Express app from the Apollo Server instance, and extract the env vars to a dedicated module:

[{]: <helper> (diffStep 10.1 module="server")

#### [Server Step 10.1: Separate app into a different module](https://github.com/Urigo/WhatsApp-Clone-Server/commit/47bf939)

##### Added app.ts
```diff
@@ -0,0 +1,15 @@
+┊  ┊ 1┊import bodyParser from 'body-parser'
+┊  ┊ 2┊import cors from 'cors'
+┊  ┊ 3┊import cookieParser from 'cookie-parser'
+┊  ┊ 4┊import express from 'express'
+┊  ┊ 5┊import { origin } from './env'
+┊  ┊ 6┊
+┊  ┊ 7┊export const app = express()
+┊  ┊ 8┊
+┊  ┊ 9┊app.use(cors({ credentials: true, origin }))
+┊  ┊10┊app.use(bodyParser.json())
+┊  ┊11┊app.use(cookieParser())
+┊  ┊12┊
+┊  ┊13┊app.get('/_ping', (req, res) => {
+┊  ┊14┊  res.send('pong')
+┊  ┊15┊})
```

##### Added env.ts
```diff
@@ -0,0 +1,2 @@
+┊ ┊1┊export const origin = process.env.ORIGIN || 'http://localhost:3000'
+┊ ┊2┊export const port = process.env.PORT || 4000
```

##### Changed index.ts
```diff
@@ -1,23 +1,10 @@
 ┊ 1┊ 1┊import { ApolloServer, gql, PubSub } from 'apollo-server-express'
-┊ 2┊  ┊import bodyParser from 'body-parser'
-┊ 3┊  ┊import cors from 'cors'
-┊ 4┊  ┊import cookieParser from 'cookie-parser'
-┊ 5┊  ┊import express from 'express'
 ┊ 6┊ 2┊import http from 'http'
+┊  ┊ 3┊import { app } from './app'
 ┊ 7┊ 4┊import { users } from './db'
+┊  ┊ 5┊import { origin, port } from './env'
 ┊ 8┊ 6┊import schema from './schema'
 ┊ 9┊ 7┊
-┊10┊  ┊const app = express()
-┊11┊  ┊
-┊12┊  ┊const origin = process.env.ORIGIN || 'http://localhost:3000'
-┊13┊  ┊app.use(cors({ credentials: true, origin }))
-┊14┊  ┊app.use(bodyParser.json())
-┊15┊  ┊app.use(cookieParser())
-┊16┊  ┊
-┊17┊  ┊app.get('/_ping', (req, res) => {
-┊18┊  ┊  res.send('pong')
-┊19┊  ┊})
-┊20┊  ┊
 ┊21┊ 8┊const pubsub = new PubSub()
 ┊22┊ 9┊const server = new ApolloServer({
 ┊23┊10┊  schema,
```
```diff
@@ -36,8 +23,6 @@
 ┊36┊23┊const httpServer = http.createServer(app)
 ┊37┊24┊server.installSubscriptionHandlers(httpServer)
 ┊38┊25┊
-┊39┊  ┊const port = process.env.PORT || 4000
-┊40┊  ┊
 ┊41┊26┊httpServer.listen(port, () => {
 ┊42┊27┊  console.log(`Server is listening on port ${port}`)
 ┊43┊28┊})
```

[}]: #

We will first start with the `signIn` mutation, so we can test it against pre-defined user credentials, and then we will proceed to implementing the `signUp` mutation. It would be a lot easier to progress this way. For that we will install a couple of packages:

- `bcrypt` - which is responsible for running a one-way encryption against received passwords before they’re stored in the DB.
- `jsonwebtoken` - responsible for encrypting the logged-in username before it’s set as a cooky and decrypting it once it’s sent back with a request.

    $ yarn add bcrypt jsonwebtoken


    $ yarn add --dev @types/bcrypt @types/jsonwebtoken

And we will implement the `signIn` mutation:

[{]: <helper> (diffStep 10.2 files="schema" module="server")

#### [Server Step 10.2: Add signIn mutation](https://github.com/Urigo/WhatsApp-Clone-Server/commit/c6ac755)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -2,6 +2,9 @@
 ┊ 2┊ 2┊import { GraphQLDateTime } from 'graphql-iso-date'
 ┊ 3┊ 3┊import { User, Message, Chat, chats, messages, users } from '../db'
 ┊ 4┊ 4┊import { Resolvers } from '../types/graphql'
+┊  ┊ 5┊import { secret, expiration } from '../env'
+┊  ┊ 6┊import bcrypt from 'bcrypt'
+┊  ┊ 7┊import jwt from 'jsonwebtoken'
 ┊ 5┊ 8┊
 ┊ 6┊ 9┊const resolvers: Resolvers = {
 ┊ 7┊10┊  Date: GraphQLDateTime,
```
```diff
@@ -89,6 +92,27 @@
 ┊ 89┊ 92┊  },
 ┊ 90┊ 93┊
 ┊ 91┊ 94┊  Mutation: {
+┊   ┊ 95┊    signIn(root, { username, password}, { res }) {
+┊   ┊ 96┊
+┊   ┊ 97┊      const user = users.find(u => u.username === username)
+┊   ┊ 98┊
+┊   ┊ 99┊      if (!user) {
+┊   ┊100┊        throw new Error('user not found')
+┊   ┊101┊      }
+┊   ┊102┊
+┊   ┊103┊      const passwordsMatch = bcrypt.compareSync(password, user.password)
+┊   ┊104┊
+┊   ┊105┊      if (!passwordsMatch) {
+┊   ┊106┊        throw new Error('password is incorrect')
+┊   ┊107┊      }
+┊   ┊108┊
+┊   ┊109┊      const authToken = jwt.sign(username, secret)
+┊   ┊110┊
+┊   ┊111┊      res.cookie('authToken', authToken, { maxAge: expiration })
+┊   ┊112┊
+┊   ┊113┊      return user;
+┊   ┊114┊    },
+┊   ┊115┊
 ┊ 92┊116┊    addMessage(root, { chatId, content }, { currentUser, pubsub }) {
 ┊ 93┊117┊      if (!currentUser) return null
 ┊ 94┊118┊
```

##### Changed schema&#x2F;typeDefs.graphql
```diff
@@ -32,6 +32,7 @@
 ┊32┊32┊}
 ┊33┊33┊
 ┊34┊34┊type Mutation {
+┊  ┊35┊  signIn(username: String!, password: String!): User
 ┊35┊36┊  addMessage(chatId: ID!, content: String!): Message
 ┊36┊37┊  addChat(recipientId: ID!): Chat
 ┊37┊38┊  removeChat(chatId: ID!): ID
```

[}]: #

As you can see we use a special secret before we encrypt the username with JWT. The same secret will be used later on to decrypt the token back into username when getting requests. If someone malicious will get a hold of that password, he can fabricate an authentication token for every user that he wants, **thus it’s important to choose a strong secret**.

When building the context for our GraphQL resolvers, we will decode the received cookie with JWT using the same secret to determine the username who made the request. Once we have that username, we can simply retrieve the original user from the DB and define it on the context:

[{]: <helper> (diffStep 10.3 module="server")

#### [Server Step 10.3: Get current user from auth token](https://github.com/Urigo/WhatsApp-Clone-Server/commit/51cae3f)

##### Changed index.ts
```diff
@@ -1,18 +1,27 @@
 ┊ 1┊ 1┊import { ApolloServer, gql, PubSub } from 'apollo-server-express'
 ┊ 2┊ 2┊import http from 'http'
+┊  ┊ 3┊import jwt from 'jsonwebtoken'
 ┊ 3┊ 4┊import { app } from './app'
 ┊ 4┊ 5┊import { users } from './db'
-┊ 5┊  ┊import { origin, port } from './env'
+┊  ┊ 6┊import { origin, port, secret } from './env'
 ┊ 6┊ 7┊import schema from './schema'
 ┊ 7┊ 8┊
 ┊ 8┊ 9┊const pubsub = new PubSub()
 ┊ 9┊10┊const server = new ApolloServer({
 ┊10┊11┊  schema,
-┊11┊  ┊  context: ({ req, res }) => ({
-┊12┊  ┊    currentUser: users.find(u => u.id === req.cookies.currentUserId),
-┊13┊  ┊    pubsub,
-┊14┊  ┊    res,
-┊15┊  ┊  }),
+┊  ┊12┊  context: ({ req, res }) => {
+┊  ┊13┊    let currentUser;
+┊  ┊14┊    if (req.cookies.authToken) {
+┊  ┊15┊      const username = jwt.verify(req.cookies.authToken, secret) as string
+┊  ┊16┊      currentUser = username && users.find(u => u.username === username)
+┊  ┊17┊    }
+┊  ┊18┊
+┊  ┊19┊    return {
+┊  ┊20┊      currentUser,
+┊  ┊21┊      pubsub,
+┊  ┊22┊      res,
+┊  ┊23┊    }
+┊  ┊24┊  },
 ┊16┊25┊})
 ┊17┊26┊
 ┊18┊27┊server.applyMiddleware({
```

[}]: #

You might have noticed that the User schema has been updated, because we try to address the `user.username` property. The authentication in our app will be done with a username and a password; accordingly, we will update our User type definitions and the user documents in the users collection mock. The credentials that we’re going to store can actually be used to sign-in to our app:

[{]: <helper> (diffStep 10.4 module="server")

#### [Server Step 10.4: Update user schema to contain credentials](https://github.com/Urigo/WhatsApp-Clone-Server/commit/754bdf4)

##### Changed db.ts
```diff
@@ -1,6 +1,8 @@
 ┊1┊1┊export type User = {
 ┊2┊2┊  id: string
 ┊3┊3┊  name: string
+┊ ┊4┊  username: string
+┊ ┊5┊  password: string
 ┊4┊6┊  picture: string
 ┊5┊7┊}
 ┊6┊8┊
```
```diff
@@ -27,26 +29,36 @@
 ┊27┊29┊    {
 ┊28┊30┊      id: '1',
 ┊29┊31┊      name: 'Ray Edwards',
+┊  ┊32┊      username: 'ray',
+┊  ┊33┊      password: '$2a$08$NO9tkFLCoSqX1c5wk3s7z.JfxaVMKA.m7zUDdDwEquo4rvzimQeJm', // 111
 ┊30┊34┊      picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
 ┊31┊35┊    },
 ┊32┊36┊    {
 ┊33┊37┊      id: '2',
 ┊34┊38┊      name: 'Ethan Gonzalez',
+┊  ┊39┊      username: 'ethan',
+┊  ┊40┊      password: '$2a$08$xE4FuCi/ifxjL2S8CzKAmuKLwv18ktksSN.F3XYEnpmcKtpbpeZgO', // 222
 ┊35┊41┊      picture: 'https://randomuser.me/api/portraits/thumb/men/1.jpg',
 ┊36┊42┊    },
 ┊37┊43┊    {
 ┊38┊44┊      id: '3',
 ┊39┊45┊      name: 'Bryan Wallace',
+┊  ┊46┊      username: 'bryan',
+┊  ┊47┊      password: '$2a$08$UHgH7J8G6z1mGQn2qx2kdeWv0jvgHItyAsL9hpEUI3KJmhVW5Q1d.', // 333
 ┊40┊48┊      picture: 'https://randomuser.me/api/portraits/thumb/men/2.jpg',
 ┊41┊49┊    },
 ┊42┊50┊    {
 ┊43┊51┊      id: '4',
 ┊44┊52┊      name: 'Avery Stewart',
+┊  ┊53┊      username: 'avery',
+┊  ┊54┊      password: '$2a$08$wR1k5Q3T9FC7fUgB7Gdb9Os/GV7dGBBf4PLlWT7HERMFhmFDt47xi', // 444
 ┊45┊55┊      picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
 ┊46┊56┊    },
 ┊47┊57┊    {
 ┊48┊58┊      id: '5',
 ┊49┊59┊      name: 'Katie Peterson',
+┊  ┊60┊      username: 'katie',
+┊  ┊61┊      password: '$2a$08$6.mbXqsDX82ZZ7q5d8Osb..JrGSsNp4R3IKj7mxgF6YGT0OmMw242', // 555
 ┊50┊62┊      picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
 ┊51┊63┊    },
 ┊52┊64┊  ])
```
```diff
@@ -106,4 +118,4 @@
 ┊106┊118┊  ])
 ┊107┊119┊}
 ┊108┊120┊
-┊109┊   ┊resetDb()
+┊   ┊121┊resetDb()🚫↵
```

[}]: #

To test it out, we will run our front-end application and open the dev-console. Using the Apollo Client we will send a request to the `signIn` mutation. We can use the credentials of one of the users stored in the DB. As for now all our restricted routes are observing the `currentUserId` cookie. This is wrong and no longer relevant. Let’s change the `withAuth()` method to observe the `authToken` cookie so we can test our new mutation successfully:

[{]: <helper> (diffStep 13.1 module="client")

#### Client Step 13.1: Use authToken cookie

##### Changed src&#x2F;services&#x2F;auth.service.tsx
```diff
@@ -23,8 +23,8 @@
 ┊23┊23┊  };
 ┊24┊24┊};
 ┊25┊25┊
-┊26┊  ┊export const signIn = (currentUserId: string) => {
-┊27┊  ┊  document.cookie = `currentUserId=${currentUserId}`;
+┊  ┊26┊export const signIn = (authToken: string) => {
+┊  ┊27┊  document.cookie = `authToken=${authToken}`;
 ┊28┊28┊
 ┊29┊29┊  // This will become async in the near future
 ┊30┊30┊  return Promise.resolve();
```
```diff
@@ -33,12 +33,12 @@
 ┊33┊33┊export const signOut = () => {
 ┊34┊34┊  // "expires" represents the lifespan of a cookie. Beyond that date the cookie will
 ┊35┊35┊  // be deleted by the browser. "expires" cannot be viewed from "document.cookie"
-┊36┊  ┊  document.cookie = `currentUserId=;expires=${new Date(0)}`;
+┊  ┊36┊  document.cookie = `authToken=;expires=${new Date(0)}`;
 ┊37┊37┊
 ┊38┊38┊  // Clear cache
 ┊39┊39┊  return client.clearStore();
 ┊40┊40┊};
 ┊41┊41┊
 ┊42┊42┊export const isSignedIn = () => {
-┊43┊  ┊  return /currentUserId=.+(;|$)/.test(document.cookie);
+┊  ┊43┊  return /authToken=.+(;|$)/.test(document.cookie);
 ┊44┊44┊};🚫↵
```

[}]: #

Now we can perform the signIn. It would be a good idea to signIn with the first user - `ray`, since all the DB mock is built around him:

```js
mutation signIn(username: 'ray', password: '111') {
  id
}
```

Now if we would look at the value of `document.cookie` we should see a key named `authToken` with a JWT token and the `ChatsListScreen` should show the chats which are relevant to `ray`. To complete the sign-in flow we would need to update the `AuthScreen` and the `auth.service` to use username and password and the actual `sign-in` mutation we’ve just implemented.

To check if we’re authorized to visit a route, not only we would need to check if we have the `authToken` cookie defined, but we would also need to validate it against the server to see that it actually references a real user. For that we will implement `Query.me` which will send us back the current user logged in directly from the context:

[{]: <helper> (diffStep 10.5 module="server")

#### [Server Step 10.5: Add Query.me](https://github.com/Urigo/WhatsApp-Clone-Server/commit/234afe4)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -68,6 +68,10 @@
 ┊68┊68┊  },
 ┊69┊69┊
 ┊70┊70┊  Query: {
+┊  ┊71┊    me(root, args, { currentUser }) {
+┊  ┊72┊      return currentUser || null
+┊  ┊73┊    },
+┊  ┊74┊
 ┊71┊75┊    chats(root, args, { currentUser }) {
 ┊72┊76┊      if (!currentUser) return []
 ┊73┊77┊
```

##### Changed schema&#x2F;typeDefs.graphql
```diff
@@ -26,6 +26,7 @@
 ┊26┊26┊}
 ┊27┊27┊
 ┊28┊28┊type Query {
+┊  ┊29┊  me: User
 ┊29┊30┊  chats: [Chat!]!
 ┊30┊31┊  chat(chatId: ID!): Chat
 ┊31┊32┊  users: [User!]!
```

##### Added tests&#x2F;queries&#x2F;getMe.test.ts
```diff
@@ -0,0 +1,33 @@
+┊  ┊ 1┊import { createTestClient } from 'apollo-server-testing'
+┊  ┊ 2┊import { ApolloServer, gql } from 'apollo-server-express'
+┊  ┊ 3┊import schema from '../../schema'
+┊  ┊ 4┊import { users } from '../../db'
+┊  ┊ 5┊
+┊  ┊ 6┊describe('Query.me', () => {
+┊  ┊ 7┊  it('should fetch current user', async () => {
+┊  ┊ 8┊    const server = new ApolloServer({
+┊  ┊ 9┊      schema,
+┊  ┊10┊      context: () => ({
+┊  ┊11┊        currentUser: users[0],
+┊  ┊12┊      }),
+┊  ┊13┊    })
+┊  ┊14┊
+┊  ┊15┊    const { query } = createTestClient(server)
+┊  ┊16┊
+┊  ┊17┊    const res = await query({
+┊  ┊18┊      query: gql `
+┊  ┊19┊        query GetMe {
+┊  ┊20┊          me {
+┊  ┊21┊            id
+┊  ┊22┊            name
+┊  ┊23┊            picture
+┊  ┊24┊          }
+┊  ┊25┊        }
+┊  ┊26┊      `,
+┊  ┊27┊    })
+┊  ┊28┊
+┊  ┊29┊    expect(res.data).toBeDefined()
+┊  ┊30┊    expect(res.errors).toBeUndefined()
+┊  ┊31┊    expect(res.data).toMatchSnapshot()
+┊  ┊32┊  })
+┊  ┊33┊})
```

[}]: #

Now back to the `auth.service`, we will replace the `signIn()` method implementation with one that actually calls the `signIn` mutation in our API:

[{]: <helper> (diffStep 13.2 module="client")

#### Client Step 13.2: Update auth service to call signIn mutation

##### Changed src&#x2F;services&#x2F;auth.service.tsx
```diff
@@ -2,6 +2,7 @@
 ┊2┊2┊import { Redirect } from 'react-router-dom';
 ┊3┊3┊import client from '../client';
 ┊4┊4┊import { useCacheService } from './cache.service';
+┊ ┊5┊import gql from 'graphql-tag';
 ┊5┊6┊
 ┊6┊7┊export const withAuth = <P extends object>(Component: React.ComponentType<P>) => {
 ┊7┊8┊  return (props: any) => {
```
```diff
@@ -18,24 +19,30 @@
 ┊18┊19┊    useCacheService();
 ┊19┊20┊
 ┊20┊21┊    return (
-┊21┊  ┊      <Component {...props as P} />
+┊  ┊22┊        <Component {...props as P} />
 ┊22┊23┊    );
 ┊23┊24┊  };
 ┊24┊25┊};
 ┊25┊26┊
-┊26┊  ┊export const signIn = (authToken: string) => {
-┊27┊  ┊  document.cookie = `authToken=${authToken}`;
-┊28┊  ┊
-┊29┊  ┊  // This will become async in the near future
-┊30┊  ┊  return Promise.resolve();
+┊  ┊27┊export const signIn = ({ username, password }: { username: string, password: string}) => {
+┊  ┊28┊  return client.mutate({
+┊  ┊29┊    mutation: gql`
+┊  ┊30┊      mutation signIn($username: String!, $password: String!) {
+┊  ┊31┊        signIn(username: $username, password: $password) {
+┊  ┊32┊          id
+┊  ┊33┊        }
+┊  ┊34┊      }
+┊  ┊35┊    `,
+┊  ┊36┊    variables: {
+┊  ┊37┊      username,
+┊  ┊38┊      password
+┊  ┊39┊    }
+┊  ┊40┊  });
 ┊31┊41┊};
 ┊32┊42┊
 ┊33┊43┊export const signOut = () => {
-┊34┊  ┊  // "expires" represents the lifespan of a cookie. Beyond that date the cookie will
-┊35┊  ┊  // be deleted by the browser. "expires" cannot be viewed from "document.cookie"
 ┊36┊44┊  document.cookie = `authToken=;expires=${new Date(0)}`;
 ┊37┊45┊
-┊38┊  ┊  // Clear cache
 ┊39┊46┊  return client.clearStore();
 ┊40┊47┊};
```

[}]: #

And we will use the GraphQL query we’ve just implemented to check if the user actually exists within the DB before we proceed to the restricted route:

[{]: <helper> (diffStep 13.3 module="client")

#### Client Step 13.3: Validate auth token against the back-end on restricted route

##### Added src&#x2F;graphql&#x2F;queries&#x2F;me.query.ts
```diff
@@ -0,0 +1,11 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import * as fragments from '../fragments';
+┊  ┊ 3┊
+┊  ┊ 4┊export default gql`
+┊  ┊ 5┊  query Me {
+┊  ┊ 6┊    me {
+┊  ┊ 7┊      ...User
+┊  ┊ 8┊    }
+┊  ┊ 9┊  }
+┊  ┊10┊  ${fragments.user}
+┊  ┊11┊`;
```

##### Changed src&#x2F;services&#x2F;auth.service.tsx
```diff
@@ -1,9 +1,17 @@
 ┊ 1┊ 1┊import React from 'react';
+┊  ┊ 2┊import { useContext } from 'react';
 ┊ 2┊ 3┊import { Redirect } from 'react-router-dom';
 ┊ 3┊ 4┊import client from '../client';
+┊  ┊ 5┊import { useMeQuery, User } from '../graphql/types';
 ┊ 4┊ 6┊import { useCacheService } from './cache.service';
 ┊ 5┊ 7┊import gql from 'graphql-tag';
 ┊ 6┊ 8┊
+┊  ┊ 9┊const MyContext = React.createContext<User|null>(null);
+┊  ┊10┊
+┊  ┊11┊export const useMe = () => {
+┊  ┊12┊  return useContext(MyContext);
+┊  ┊13┊};
+┊  ┊14┊
 ┊ 7┊15┊export const withAuth = <P extends object>(Component: React.ComponentType<P>) => {
 ┊ 8┊16┊  return (props: any) => {
 ┊ 9┊17┊    if (!isSignedIn()) {
```
```diff
@@ -16,10 +24,24 @@
 ┊16┊24┊      );
 ┊17┊25┊    }
 ┊18┊26┊
+┊  ┊27┊    const { data, error, loading } = useMeQuery();
+┊  ┊28┊
 ┊19┊29┊    useCacheService();
 ┊20┊30┊
+┊  ┊31┊    if (loading) return null;
+┊  ┊32┊
+┊  ┊33┊    if (data === undefined) return null;
+┊  ┊34┊
+┊  ┊35┊    if (error || !data.me) {
+┊  ┊36┊      signOut();
+┊  ┊37┊
+┊  ┊38┊      return <Redirect to="/sign-in" />;
+┊  ┊39┊    }
+┊  ┊40┊
 ┊21┊41┊    return (
+┊  ┊42┊      <MyContext.Provider value={data.me}>
 ┊22┊43┊        <Component {...props as P} />
+┊  ┊44┊      </MyContext.Provider>
 ┊23┊45┊    );
 ┊24┊46┊  };
 ┊25┊47┊};
```
```diff
@@ -48,4 +70,4 @@
 ┊48┊70┊
 ┊49┊71┊export const isSignedIn = () => {
 ┊50┊72┊  return /authToken=.+(;|$)/.test(document.cookie);
-┊51┊  ┊};🚫↵
+┊  ┊73┊};
```

[}]: #

we will use the new query to try and fetch the user directly from the back-end, and we will only proceed if the user was actually found. In addition, we will replace the `signIn()` method to call `signIn` mutation:

[{]: <helper> (diffStep 13.4 module="client")

#### Client Step 13.4: Add username and password to AuthScreen

##### Changed src&#x2F;components&#x2F;AuthScreen&#x2F;index.tsx
```diff
@@ -114,21 +114,34 @@
 ┊114┊114┊` as typeof MaterialButton;
 ┊115┊115┊
 ┊116┊116┊const AuthScreen: React.FC<RouteComponentProps<any>> = ({ history }) => {
-┊117┊   ┊  const [userId, setUserId] = useState('');
+┊   ┊117┊  const [username, setUsername] = useState('');
+┊   ┊118┊  const [password, setPassword] = useState('');
+┊   ┊119┊  // eslint-disable-next-line
+┊   ┊120┊  const [error, setError] = useState('');
+┊   ┊121┊
+┊   ┊122┊  const onUsernameChange = useCallback(({ target }) => {
+┊   ┊123┊    setError('');
+┊   ┊124┊    setUsername(target.value);
+┊   ┊125┊  }, []);
 ┊118┊126┊
-┊119┊   ┊  const onUserIdChange = useCallback(({ target }) => {
-┊120┊   ┊    setUserId(target.value);
+┊   ┊127┊  const onPasswordChange = useCallback(({ target }) => {
+┊   ┊128┊    setError('');
+┊   ┊129┊    setPassword(target.value);
 ┊121┊130┊  }, []);
 ┊122┊131┊
 ┊123┊132┊  const maySignIn = useCallback(() => {
-┊124┊   ┊    return !!userId
-┊125┊   ┊  }, [userId]);
+┊   ┊133┊    return !!(username && password);
+┊   ┊134┊  }, [username, password]);
 ┊126┊135┊
 ┊127┊136┊  const handleSignIn = useCallback(() => {
-┊128┊   ┊    signIn(userId).then(() => {
-┊129┊   ┊      history.replace('/chats')
-┊130┊   ┊    })
-┊131┊   ┊  }, [userId, history]);
+┊   ┊137┊    signIn({ username, password })
+┊   ┊138┊      .then(() => {
+┊   ┊139┊        history.push('/chats')
+┊   ┊140┊      })
+┊   ┊141┊      .catch(error => {
+┊   ┊142┊        setError(error.message || error)
+┊   ┊143┊      });
+┊   ┊144┊  }, [username, password, history]);
 ┊132┊145┊
 ┊133┊146┊  return (
 ┊134┊147┊    <Container>
```
```diff
@@ -141,12 +154,21 @@
 ┊141┊154┊          <Legend>Sign in</Legend>
 ┊142┊155┊          <Section>
 ┊143┊156┊            <TextField
-┊144┊   ┊              data-testid="user-id-input"
-┊145┊   ┊              label="User ID"
-┊146┊   ┊              value={userId}
-┊147┊   ┊              onChange={onUserIdChange}
+┊   ┊157┊              className="AuthScreen-text-field"
+┊   ┊158┊              label="Username"
+┊   ┊159┊              value={username}
+┊   ┊160┊              onChange={onUsernameChange}
+┊   ┊161┊              margin="normal"
+┊   ┊162┊              placeholder="Enter your username"
+┊   ┊163┊            />
+┊   ┊164┊            <TextField
+┊   ┊165┊              className="AuthScreen-text-field"
+┊   ┊166┊              label="Password"
+┊   ┊167┊              type="password"
+┊   ┊168┊              value={password}
+┊   ┊169┊              onChange={onPasswordChange}
 ┊148┊170┊              margin="normal"
-┊149┊   ┊              placeholder="Enter current user ID"
+┊   ┊171┊              placeholder="Enter your password"
 ┊150┊172┊            />
 ┊151┊173┊          </Section>
 ┊152┊174┊          <Button
```

[}]: #

The behavior of the updated screen should be identical to what we had so far. To complete the flow we’ll need a way to signUp. When we signing-up we will need the following parameters: `name`, `username`, `password` and `passwordConfirm`. In addition we will need to run certain validations against the parameters:

- The name must be at least 3 and at most 50 characters long.
- The username must be at least 3 and at most 18 characters long.
- A password must be at least 8 and at most 30 characters long. In addition, it should contain English letters, numbers, and special characters.

For that we will implement a dedicated validations module:

[{]: <helper> (diffStep 10.6 files="validators" module="server")

#### [Server Step 10.6: Add signUp mutation](https://github.com/Urigo/WhatsApp-Clone-Server/commit/3e749d6)

##### Added validators.ts
```diff
@@ -0,0 +1,43 @@
+┊  ┊ 1┊export const validatePassword = (ctx: string, str: string) => {
+┊  ┊ 2┊  if (typeof str !== 'string') {
+┊  ┊ 3┊    throw TypeError(`${ctx} must be a string`)
+┊  ┊ 4┊  }
+┊  ┊ 5┊
+┊  ┊ 6┊  validateLength(ctx, str, 8, 30)
+┊  ┊ 7┊
+┊  ┊ 8┊  if (!/[a-zA-Z]+/.test(str)) {
+┊  ┊ 9┊    throw TypeError(`${ctx} must contain english letters`)
+┊  ┊10┊  }
+┊  ┊11┊
+┊  ┊12┊  if (!/\d+/.test(str)) {
+┊  ┊13┊    throw TypeError(`${ctx} must contain numbers`)
+┊  ┊14┊  }
+┊  ┊15┊
+┊  ┊16┊  if (!/[^\da-zA-Z]+/.test(str)) {
+┊  ┊17┊    throw TypeError(`${ctx} must contain special charachters`)
+┊  ┊18┊  }
+┊  ┊19┊}
+┊  ┊20┊
+┊  ┊21┊export const validateLength = (ctx: string, str: string, ...args: number[]) => {
+┊  ┊22┊  let min, max
+┊  ┊23┊
+┊  ┊24┊  if (args.length === 1) {
+┊  ┊25┊    min = 0
+┊  ┊26┊    max = args[0]
+┊  ┊27┊  } else {
+┊  ┊28┊    min = args[0]
+┊  ┊29┊    max = args[1]
+┊  ┊30┊  }
+┊  ┊31┊
+┊  ┊32┊  if (typeof str !== 'string') {
+┊  ┊33┊    throw TypeError(`${ctx} must be a string`)
+┊  ┊34┊  }
+┊  ┊35┊
+┊  ┊36┊  if (str.length < min) {
+┊  ┊37┊    throw TypeError(`${ctx} must be at least ${min} chars long`)
+┊  ┊38┊  }
+┊  ┊39┊
+┊  ┊40┊  if (str.length > max) {
+┊  ┊41┊    throw TypeError(`${ctx} must contain ${max} chars at most`)
+┊  ┊42┊  }
+┊  ┊43┊}
```

[}]: #

And we will implement the resolver and schema for the `signUp` mutation:

[{]: <helper> (diffStep 10.6 files="schema" module="server")

#### [Server Step 10.6: Add signUp mutation](https://github.com/Urigo/WhatsApp-Clone-Server/commit/3e749d6)

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -5,6 +5,7 @@
 ┊ 5┊ 5┊import { secret, expiration } from '../env'
 ┊ 6┊ 6┊import bcrypt from 'bcrypt'
 ┊ 7┊ 7┊import jwt from 'jsonwebtoken'
+┊  ┊ 8┊import { validateLength, validatePassword } from '../validators';
 ┊ 8┊ 9┊
 ┊ 9┊10┊const resolvers: Resolvers = {
 ┊10┊11┊  Date: GraphQLDateTime,
```
```diff
@@ -96,20 +97,20 @@
 ┊ 96┊ 97┊  },
 ┊ 97┊ 98┊
 ┊ 98┊ 99┊  Mutation: {
-┊ 99┊   ┊    signIn(root, { username, password}, { res }) {
+┊   ┊100┊    signIn(root, { username, password }, { res }) {
 ┊100┊101┊
 ┊101┊102┊      const user = users.find(u => u.username === username)
-┊102┊   ┊
+┊   ┊103┊
 ┊103┊104┊      if (!user) {
 ┊104┊105┊        throw new Error('user not found')
 ┊105┊106┊      }
-┊106┊   ┊
+┊   ┊107┊
 ┊107┊108┊      const passwordsMatch = bcrypt.compareSync(password, user.password)
-┊108┊   ┊
+┊   ┊109┊
 ┊109┊110┊      if (!passwordsMatch) {
 ┊110┊111┊        throw new Error('password is incorrect')
 ┊111┊112┊      }
-┊112┊   ┊
+┊   ┊113┊
 ┊113┊114┊      const authToken = jwt.sign(username, secret)
 ┊114┊115┊
 ┊115┊116┊      res.cookie('authToken', authToken, { maxAge: expiration })
```
```diff
@@ -117,6 +118,35 @@
 ┊117┊118┊      return user;
 ┊118┊119┊    },
 ┊119┊120┊
+┊   ┊121┊    signUp(root, { name, username, password, passwordConfirm }) {
+┊   ┊122┊
+┊   ┊123┊      validateLength('req.name', name, 3, 50)
+┊   ┊124┊      validateLength('req.username', name, 3, 18)
+┊   ┊125┊      validatePassword('req.password', password)
+┊   ┊126┊
+┊   ┊127┊      if (password !== passwordConfirm) {
+┊   ┊128┊        throw Error("req.password and req.passwordConfirm don't match")
+┊   ┊129┊      }
+┊   ┊130┊
+┊   ┊131┊      if (users.some(u => u.username === username)) {
+┊   ┊132┊        throw Error("username already exists")
+┊   ┊133┊      }
+┊   ┊134┊
+┊   ┊135┊      const passwordHash = bcrypt.hashSync(password, bcrypt.genSaltSync(8))
+┊   ┊136┊
+┊   ┊137┊      const user: User = {
+┊   ┊138┊        id: String(users.length + 1),
+┊   ┊139┊        password: passwordHash,
+┊   ┊140┊        picture: '',
+┊   ┊141┊        username,
+┊   ┊142┊        name,
+┊   ┊143┊      }
+┊   ┊144┊
+┊   ┊145┊      users.push(user)
+┊   ┊146┊
+┊   ┊147┊      return user
+┊   ┊148┊    },
+┊   ┊149┊
 ┊120┊150┊    addMessage(root, { chatId, content }, { currentUser, pubsub }) {
 ┊121┊151┊      if (!currentUser) return null
 ┊122┊152┊
```

##### Changed schema&#x2F;typeDefs.graphql
```diff
@@ -34,6 +34,7 @@
 ┊34┊34┊
 ┊35┊35┊type Mutation {
 ┊36┊36┊  signIn(username: String!, password: String!): User
+┊  ┊37┊  signUp(name: String!, username: String!, password: String!, passwordConfirm: String!): User
 ┊37┊38┊  addMessage(chatId: ID!, content: String!): Message
 ┊38┊39┊  addChat(recipientId: ID!): Chat
 ┊39┊40┊  removeChat(chatId: ID!): ID
```

[}]: #

Before encrypting the password we append a string called “salt” to it. Even though the passwords are stored encrypted in the DB, a hacker might use a dictionary of common passwords in their encrypted form to decipher the original password. When adding salt to a password which is essentially a random string, the hacker cannot use a dictionary anymore since he would need to know the salt. Hypothetically, the hacker can get a hold of the salt and re-generate the entire dictionary, however that would take too long because of the way Bcrypt is designed to work.

Going back to the client, we will implement a new `signUp()` method in the `auth.service` that will call the `signUp` mutation:

[{]: <helper> (diffStep 13.5 module="client")

#### Client Step 13.5: Add signUp() method to auth.service

##### Changed src&#x2F;services&#x2F;auth.service.tsx
```diff
@@ -62,6 +62,25 @@
 ┊62┊62┊  });
 ┊63┊63┊};
 ┊64┊64┊
+┊  ┊65┊export const signUp = ({ name, username, password, passwordConfirm }:
+┊  ┊66┊  {name: string, username: string, password: string, passwordConfirm: string}) => {
+┊  ┊67┊  return client.mutate({
+┊  ┊68┊    mutation: gql`
+┊  ┊69┊      mutation signUp($name: String!, $username: String!, $password: String!, $passwordConfirm: String!) {
+┊  ┊70┊        signUp(name: $name, username: $username, password: $password, passwordConfirm: $passwordConfirm) {
+┊  ┊71┊          id
+┊  ┊72┊        }
+┊  ┊73┊      }
+┊  ┊74┊    `,
+┊  ┊75┊    variables: {
+┊  ┊76┊      name,
+┊  ┊77┊      username,
+┊  ┊78┊      password,
+┊  ┊79┊      passwordConfirm
+┊  ┊80┊    }
+┊  ┊81┊  });
+┊  ┊82┊};
+┊  ┊83┊
 ┊65┊84┊export const signOut = () => {
 ┊66┊85┊  document.cookie = `authToken=;expires=${new Date(0)}`;
```

[}]: #

Now we will implement a dedicated `SignUpForm` that we can use to perform the sign-up. Instead of implementing a new screen, we will use the `AuthScreen` to alternate between the `SignInForm` and the `SignUpForm` using `AnimatedSwitch`. This way we can have a container component that is common for both forms, and we will be able to switch between the two very smoothly. We will first define a new `/sign-up` route in our router:

[{]: <helper> (diffStep 13.6 module="client")

#### Client Step 13.6: Split AuthScreen into SignInForm and SignUpForm

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;SignInForm.test.tsx
```diff
@@ -0,0 +1,81 @@
+┊  ┊ 1┊import { createMemoryHistory } from 'history';
+┊  ┊ 2┊import React from 'react';
+┊  ┊ 3┊import { cleanup, render, fireEvent, wait, waitForElement } from 'react-testing-library';
+┊  ┊ 4┊import SignInForm from './SignInForm';
+┊  ┊ 5┊
+┊  ┊ 6┊describe('SignInForm', () => {
+┊  ┊ 7┊  afterEach(cleanup);
+┊  ┊ 8┊  afterEach(() => fetch.resetMocks());
+┊  ┊ 9┊
+┊  ┊10┊  it('enables sign-in button when filled in', async () => {
+┊  ┊11┊    const history = createMemoryHistory();
+┊  ┊12┊
+┊  ┊13┊    {
+┊  ┊14┊      const { container, getByTestId } = render(<SignInForm history={history} />);
+┊  ┊15┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊16┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊17┊      const signInButton = getByTestId('sign-in-button') as HTMLButtonElement;
+┊  ┊18┊
+┊  ┊19┊      expect(signInButton.disabled).toEqual(true);
+┊  ┊20┊
+┊  ┊21┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊22┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊23┊
+┊  ┊24┊      await waitForElement(() => usernameInput);
+┊  ┊25┊      await waitForElement(() => passwordInput);
+┊  ┊26┊
+┊  ┊27┊      expect(signInButton.disabled).toEqual(false);
+┊  ┊28┊    }
+┊  ┊29┊  });
+┊  ┊30┊
+┊  ┊31┊  it('prints server error if input was wrong', async () => {
+┊  ┊32┊    const history = createMemoryHistory();
+┊  ┊33┊
+┊  ┊34┊    fetchMock.mockRejectOnce(new Error('sign-in failed'));
+┊  ┊35┊
+┊  ┊36┊    {
+┊  ┊37┊      const { container, getByTestId } = render(<SignInForm history={history} />);
+┊  ┊38┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊39┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊40┊      const signInButton = getByTestId('sign-in-button') as HTMLButtonElement;
+┊  ┊41┊      const errorMessage = getByTestId('error-message');
+┊  ┊42┊
+┊  ┊43┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊44┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊45┊
+┊  ┊46┊      await waitForElement(() => usernameInput);
+┊  ┊47┊      await waitForElement(() => passwordInput);
+┊  ┊48┊
+┊  ┊49┊      fireEvent.click(signInButton);
+┊  ┊50┊
+┊  ┊51┊      await waitForElement(() => errorMessage);
+┊  ┊52┊
+┊  ┊53┊      expect(errorMessage.innerHTML).toEqual('sign-in failed');
+┊  ┊54┊    }
+┊  ┊55┊  });
+┊  ┊56┊
+┊  ┊57┊  it('navigates to /chats if everything went right', async () => {
+┊  ┊58┊    const history = createMemoryHistory();
+┊  ┊59┊
+┊  ┊60┊    fetchMock.mockResponseOnce('success');
+┊  ┊61┊
+┊  ┊62┊    {
+┊  ┊63┊      const { container, getByTestId } = render(<SignInForm history={history} />);
+┊  ┊64┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊65┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊66┊      const signInButton = getByTestId('sign-in-button') as HTMLButtonElement;
+┊  ┊67┊
+┊  ┊68┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊69┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊70┊
+┊  ┊71┊      await waitForElement(() => usernameInput);
+┊  ┊72┊      await waitForElement(() => passwordInput);
+┊  ┊73┊
+┊  ┊74┊      fireEvent.click(signInButton);
+┊  ┊75┊
+┊  ┊76┊      await wait(() =>
+┊  ┊77┊        expect(history.location.pathname).toEqual('/chats')
+┊  ┊78┊      );
+┊  ┊79┊    }
+┊  ┊80┊  });
+┊  ┊81┊});
```

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;SignInForm.tsx
```diff
@@ -0,0 +1,83 @@
+┊  ┊ 1┊import React from 'react';
+┊  ┊ 2┊import { useCallback, useState } from 'react';
+┊  ┊ 3┊import { signIn } from '../../services/auth.service';
+┊  ┊ 4┊import {
+┊  ┊ 5┊  SignForm,
+┊  ┊ 6┊  ActualForm,
+┊  ┊ 7┊  Legend,
+┊  ┊ 8┊  Section,
+┊  ┊ 9┊  TextField,
+┊  ┊10┊  Button,
+┊  ┊11┊  ErrorMessage,
+┊  ┊12┊} from './form-components';
+┊  ┊13┊import { RouteComponentProps } from 'react-router-dom';
+┊  ┊14┊
+┊  ┊15┊const SignInForm: React.FC<RouteComponentProps<any>> = ({ history }) => {
+┊  ┊16┊  const [username, setUsername] = useState('');
+┊  ┊17┊  const [password, setPassword] = useState('');
+┊  ┊18┊  const [error, setError] = useState('');
+┊  ┊19┊
+┊  ┊20┊  const onUsernameChange = useCallback(({ target }) => {
+┊  ┊21┊    setError('');
+┊  ┊22┊    setUsername(target.value);
+┊  ┊23┊  }, []);
+┊  ┊24┊
+┊  ┊25┊  const onPasswordChange = useCallback(({ target }) => {
+┊  ┊26┊    setError('');
+┊  ┊27┊    setPassword(target.value);
+┊  ┊28┊  }, []);
+┊  ┊29┊
+┊  ┊30┊  const maySignIn = useCallback(() => {
+┊  ┊31┊    return !!(username && password);
+┊  ┊32┊  }, [username, password]);
+┊  ┊33┊
+┊  ┊34┊  const handleSignIn = useCallback(() => {
+┊  ┊35┊    signIn({ username, password })
+┊  ┊36┊      .then(() => {
+┊  ┊37┊        history.replace('/chats')
+┊  ┊38┊      })
+┊  ┊39┊      .catch(error => {
+┊  ┊40┊        setError(error.message || error)
+┊  ┊41┊      });
+┊  ┊42┊  }, [username, password, history]);
+┊  ┊43┊
+┊  ┊44┊  return (
+┊  ┊45┊    <SignForm>
+┊  ┊46┊      <ActualForm>
+┊  ┊47┊        <Legend>Sign in</Legend>
+┊  ┊48┊        <Section style={{ width: '100%' }}>
+┊  ┊49┊          <TextField
+┊  ┊50┊            data-testid="username-input"
+┊  ┊51┊            label="Username"
+┊  ┊52┊            value={username}
+┊  ┊53┊            onChange={onUsernameChange}
+┊  ┊54┊            margin="normal"
+┊  ┊55┊            placeholder="Enter your username"
+┊  ┊56┊          />
+┊  ┊57┊          <TextField
+┊  ┊58┊            data-testid="password-input"
+┊  ┊59┊            label="Password"
+┊  ┊60┊            type="password"
+┊  ┊61┊            value={password}
+┊  ┊62┊            onChange={onPasswordChange}
+┊  ┊63┊            margin="normal"
+┊  ┊64┊            placeholder="Enter your password"
+┊  ┊65┊          />
+┊  ┊66┊        </Section>
+┊  ┊67┊        <Button
+┊  ┊68┊          data-testid="sign-in-button"
+┊  ┊69┊          type="button"
+┊  ┊70┊          color="secondary"
+┊  ┊71┊          variant="contained"
+┊  ┊72┊          disabled={!maySignIn()}
+┊  ┊73┊          onClick={handleSignIn}
+┊  ┊74┊        >
+┊  ┊75┊          Sign in
+┊  ┊76┊        </Button>
+┊  ┊77┊        <ErrorMessage data-testid="error-message">{error}</ErrorMessage>
+┊  ┊78┊      </ActualForm>
+┊  ┊79┊    </SignForm>
+┊  ┊80┊  );
+┊  ┊81┊};
+┊  ┊82┊
+┊  ┊83┊export default SignInForm;
```

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;SignUpForm.test.tsx
```diff
@@ -0,0 +1,99 @@
+┊  ┊ 1┊import { createMemoryHistory } from 'history';
+┊  ┊ 2┊import React from 'react';
+┊  ┊ 3┊import { cleanup, render, fireEvent, wait, waitForElement } from 'react-testing-library';
+┊  ┊ 4┊import SignUpForm from './SignUpForm';
+┊  ┊ 5┊
+┊  ┊ 6┊describe('SignUpForm', () => {
+┊  ┊ 7┊  afterEach(cleanup);
+┊  ┊ 8┊  afterEach(() => fetch.resetMocks());
+┊  ┊ 9┊
+┊  ┊10┊  it('enables sign-up button when filled in', async () => {
+┊  ┊11┊    const history = createMemoryHistory();
+┊  ┊12┊
+┊  ┊13┊    {
+┊  ┊14┊      const { container, getByTestId } = render(<SignUpForm history={history} />);
+┊  ┊15┊      const nameInput = getByTestId('name-input').querySelector('input');
+┊  ┊16┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊17┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊18┊      const passwordConfirmInput = getByTestId('password-confirm-input').querySelector('input');
+┊  ┊19┊      const signUpButton = getByTestId('sign-up-button') as HTMLButtonElement;
+┊  ┊20┊
+┊  ┊21┊      expect(signUpButton.disabled).toEqual(true);
+┊  ┊22┊
+┊  ┊23┊      fireEvent.change(nameInput, { target: { value: 'User Name' } });
+┊  ┊24┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊25┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊26┊      fireEvent.change(passwordConfirmInput, { target: { value: 'password' } });
+┊  ┊27┊
+┊  ┊28┊      await waitForElement(() => nameInput);
+┊  ┊29┊      await waitForElement(() => usernameInput);
+┊  ┊30┊      await waitForElement(() => passwordInput);
+┊  ┊31┊      await waitForElement(() => passwordConfirmInput);
+┊  ┊32┊
+┊  ┊33┊      expect(signUpButton.disabled).toEqual(false);
+┊  ┊34┊    }
+┊  ┊35┊  });
+┊  ┊36┊
+┊  ┊37┊  it('prints server error if input was wrong', async () => {
+┊  ┊38┊    const history = createMemoryHistory();
+┊  ┊39┊
+┊  ┊40┊    fetchMock.mockRejectOnce(new Error('sign-up failed'));
+┊  ┊41┊
+┊  ┊42┊    {
+┊  ┊43┊      const { container, getByTestId } = render(<SignUpForm history={history} />);
+┊  ┊44┊      const nameInput = getByTestId('name-input').querySelector('input');
+┊  ┊45┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊46┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊47┊      const passwordConfirmInput = getByTestId('password-confirm-input').querySelector('input');
+┊  ┊48┊      const signUpButton = getByTestId('sign-up-button') as HTMLButtonElement;
+┊  ┊49┊      const errorMessage = getByTestId('error-message');
+┊  ┊50┊
+┊  ┊51┊      fireEvent.change(nameInput, { target: { value: 'User Name' } });
+┊  ┊52┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊53┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊54┊      fireEvent.change(passwordConfirmInput, { target: { value: 'password' } });
+┊  ┊55┊
+┊  ┊56┊      await waitForElement(() => nameInput);
+┊  ┊57┊      await waitForElement(() => usernameInput);
+┊  ┊58┊      await waitForElement(() => passwordInput);
+┊  ┊59┊      await waitForElement(() => passwordConfirmInput);
+┊  ┊60┊
+┊  ┊61┊      fireEvent.click(signUpButton);
+┊  ┊62┊
+┊  ┊63┊      await waitForElement(() => errorMessage);
+┊  ┊64┊
+┊  ┊65┊      expect(errorMessage.innerHTML).toEqual('sign-up failed');
+┊  ┊66┊    }
+┊  ┊67┊  });
+┊  ┊68┊
+┊  ┊69┊  it('navigates to /sign-in if everything went right', async () => {
+┊  ┊70┊    const history = createMemoryHistory();
+┊  ┊71┊
+┊  ┊72┊    fetchMock.mockResponseOnce('success');
+┊  ┊73┊
+┊  ┊74┊    {
+┊  ┊75┊      const { container, getByTestId } = render(<SignUpForm history={history} />);
+┊  ┊76┊      const nameInput = getByTestId('name-input').querySelector('input');
+┊  ┊77┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊78┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊79┊      const passwordConfirmInput = getByTestId('password-confirm-input').querySelector('input');
+┊  ┊80┊      const signUpButton = getByTestId('sign-up-button') as HTMLButtonElement;
+┊  ┊81┊
+┊  ┊82┊      fireEvent.change(nameInput, { target: { value: 'User Name' } });
+┊  ┊83┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊84┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊85┊      fireEvent.change(passwordConfirmInput, { target: { value: 'password' } });
+┊  ┊86┊
+┊  ┊87┊      await waitForElement(() => nameInput);
+┊  ┊88┊      await waitForElement(() => usernameInput);
+┊  ┊89┊      await waitForElement(() => passwordInput);
+┊  ┊90┊      await waitForElement(() => passwordConfirmInput);
+┊  ┊91┊
+┊  ┊92┊      fireEvent.click(signUpButton);
+┊  ┊93┊
+┊  ┊94┊      await wait(() =>
+┊  ┊95┊        expect(history.location.pathname).toEqual('/sign-in')
+┊  ┊96┊      );
+┊  ┊97┊    }
+┊  ┊98┊  });
+┊  ┊99┊});
```

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;SignUpForm.tsx
```diff
@@ -0,0 +1,126 @@
+┊   ┊  1┊import React from 'react';
+┊   ┊  2┊import { useCallback, useState } from 'react';
+┊   ┊  3┊import { signUp } from '../../services/auth.service';
+┊   ┊  4┊import {
+┊   ┊  5┊  SignForm,
+┊   ┊  6┊  ActualForm,
+┊   ┊  7┊  Legend,
+┊   ┊  8┊  Section,
+┊   ┊  9┊  TextField,
+┊   ┊ 10┊  Button,
+┊   ┊ 11┊  ErrorMessage,
+┊   ┊ 12┊} from './form-components';
+┊   ┊ 13┊import { RouteComponentProps } from 'react-router-dom';
+┊   ┊ 14┊
+┊   ┊ 15┊const SignUpForm: React.FC<RouteComponentProps<any>> = ({ history }) => {
+┊   ┊ 16┊  const [name, setName] = useState('');
+┊   ┊ 17┊  const [username, setUsername] = useState('');
+┊   ┊ 18┊  const [password, setPassword] = useState('');
+┊   ┊ 19┊  const [passwordConfirm, setPasswordConfirm] = useState('');
+┊   ┊ 20┊  const [error, setError] = useState('');
+┊   ┊ 21┊
+┊   ┊ 22┊  const updateName = useCallback(({ target }) => {
+┊   ┊ 23┊    setError('');
+┊   ┊ 24┊    setName(target.value);
+┊   ┊ 25┊  }, []);
+┊   ┊ 26┊
+┊   ┊ 27┊  const updateUsername = useCallback(({ target }) => {
+┊   ┊ 28┊    setError('');
+┊   ┊ 29┊    setUsername(target.value);
+┊   ┊ 30┊  }, []);
+┊   ┊ 31┊
+┊   ┊ 32┊  const updatePassword = useCallback(({ target }) => {
+┊   ┊ 33┊    setError('');
+┊   ┊ 34┊    setPassword(target.value);
+┊   ┊ 35┊  }, []);
+┊   ┊ 36┊
+┊   ┊ 37┊  const updatePasswordConfirm = useCallback(({ target }) => {
+┊   ┊ 38┊    setError('');
+┊   ┊ 39┊    setPasswordConfirm(target.value);
+┊   ┊ 40┊  }, []);
+┊   ┊ 41┊
+┊   ┊ 42┊  const maySignUp = useCallback(() => {
+┊   ┊ 43┊    return !!(name && username && password && password === passwordConfirm);
+┊   ┊ 44┊  }, [name, username, password, passwordConfirm]);
+┊   ┊ 45┊
+┊   ┊ 46┊  const handleSignUp = useCallback(() => {
+┊   ┊ 47┊    signUp({ username, password, passwordConfirm, name })
+┊   ┊ 48┊      .then(() => {
+┊   ┊ 49┊        history.replace('/sign-in')
+┊   ┊ 50┊      })
+┊   ┊ 51┊      .catch(error => {
+┊   ┊ 52┊        setError(error.message || error)
+┊   ┊ 53┊      });
+┊   ┊ 54┊  }, [name, username, password, passwordConfirm, history]);
+┊   ┊ 55┊
+┊   ┊ 56┊  return (
+┊   ┊ 57┊    <SignForm>
+┊   ┊ 58┊      <ActualForm>
+┊   ┊ 59┊        <Legend>Sign up</Legend>
+┊   ┊ 60┊        <Section
+┊   ┊ 61┊          style={{
+┊   ┊ 62┊            float: 'left',
+┊   ┊ 63┊            width: 'calc(50% - 10px)',
+┊   ┊ 64┊            paddingRight: '10px',
+┊   ┊ 65┊          }}
+┊   ┊ 66┊        >
+┊   ┊ 67┊          <TextField
+┊   ┊ 68┊            data-testid="name-input"
+┊   ┊ 69┊            label="Name"
+┊   ┊ 70┊            value={name}
+┊   ┊ 71┊            onChange={updateName}
+┊   ┊ 72┊            autoComplete="off"
+┊   ┊ 73┊            margin="normal"
+┊   ┊ 74┊          />
+┊   ┊ 75┊          <TextField
+┊   ┊ 76┊            data-testid="username-input"
+┊   ┊ 77┊            label="Username"
+┊   ┊ 78┊            value={username}
+┊   ┊ 79┊            onChange={updateUsername}
+┊   ┊ 80┊            autoComplete="off"
+┊   ┊ 81┊            margin="normal"
+┊   ┊ 82┊          />
+┊   ┊ 83┊        </Section>
+┊   ┊ 84┊        <Section
+┊   ┊ 85┊          style={{
+┊   ┊ 86┊            float: 'right',
+┊   ┊ 87┊            width: 'calc(50% - 10px)',
+┊   ┊ 88┊            paddingLeft: '10px',
+┊   ┊ 89┊          }}
+┊   ┊ 90┊        >
+┊   ┊ 91┊          <TextField
+┊   ┊ 92┊            data-testid="password-input"
+┊   ┊ 93┊            label="Password"
+┊   ┊ 94┊            type="password"
+┊   ┊ 95┊            value={password}
+┊   ┊ 96┊            onChange={updatePassword}
+┊   ┊ 97┊            autoComplete="off"
+┊   ┊ 98┊            margin="normal"
+┊   ┊ 99┊          />
+┊   ┊100┊          <TextField
+┊   ┊101┊            data-testid="password-confirm-input"
+┊   ┊102┊            label="Confirm password"
+┊   ┊103┊            type="password"
+┊   ┊104┊            value={passwordConfirm}
+┊   ┊105┊            onChange={updatePasswordConfirm}
+┊   ┊106┊            autoComplete="off"
+┊   ┊107┊            margin="normal"
+┊   ┊108┊          />
+┊   ┊109┊        </Section>
+┊   ┊110┊        <Button
+┊   ┊111┊          data-testid="sign-up-button"
+┊   ┊112┊          type="button"
+┊   ┊113┊          color="secondary"
+┊   ┊114┊          variant="contained"
+┊   ┊115┊          disabled={!maySignUp()}
+┊   ┊116┊          onClick={handleSignUp}
+┊   ┊117┊        >
+┊   ┊118┊          Sign up
+┊   ┊119┊        </Button>
+┊   ┊120┊        <ErrorMessage data-testid="error-message">{error}</ErrorMessage>
+┊   ┊121┊      </ActualForm>
+┊   ┊122┊    </SignForm>
+┊   ┊123┊  );
+┊   ┊124┊};
+┊   ┊125┊
+┊   ┊126┊export default SignUpForm;🚫↵
```

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;form-components.ts
```diff
@@ -0,0 +1,75 @@
+┊  ┊ 1┊import MaterialButton from '@material-ui/core/Button';
+┊  ┊ 2┊import MaterialTextField from '@material-ui/core/TextField';
+┊  ┊ 3┊import styled from 'styled-components';
+┊  ┊ 4┊
+┊  ┊ 5┊export const SignForm = styled.div `
+┊  ┊ 6┊  height: calc(100% - 265px);
+┊  ┊ 7┊`;
+┊  ┊ 8┊
+┊  ┊ 9┊export const ActualForm = styled.form `
+┊  ┊10┊  padding: 20px;
+┊  ┊11┊`;
+┊  ┊12┊
+┊  ┊13┊export const Section = styled.div `
+┊  ┊14┊  padding-bottom: 35px;
+┊  ┊15┊`;
+┊  ┊16┊
+┊  ┊17┊export const Legend = styled.legend `
+┊  ┊18┊  font-weight: bold;
+┊  ┊19┊  color: white;
+┊  ┊20┊`;
+┊  ┊21┊
+┊  ┊22┊export const Label = styled.label `
+┊  ┊23┊  color: white !important;
+┊  ┊24┊`;
+┊  ┊25┊
+┊  ┊26┊export const Input = styled.input `
+┊  ┊27┊  color: white;
+┊  ┊28┊
+┊  ┊29┊  &::placeholder {
+┊  ┊30┊    color: var(--primary-bg);
+┊  ┊31┊  }
+┊  ┊32┊`;
+┊  ┊33┊
+┊  ┊34┊export const TextField = styled(MaterialTextField) `
+┊  ┊35┊  width: 100%;
+┊  ┊36┊  position: relative;
+┊  ┊37┊
+┊  ┊38┊  > div::before {
+┊  ┊39┊    border-color: white !important;
+┊  ┊40┊  }
+┊  ┊41┊
+┊  ┊42┊  input {
+┊  ┊43┊    color: white !important;
+┊  ┊44┊
+┊  ┊45┊    &::placeholder {
+┊  ┊46┊      color: var(--primary-bg) !important;
+┊  ┊47┊    }
+┊  ┊48┊  }
+┊  ┊49┊
+┊  ┊50┊  label {
+┊  ┊51┊    color: white !important;
+┊  ┊52┊  }
+┊  ┊53┊` as typeof MaterialTextField;
+┊  ┊54┊
+┊  ┊55┊export const Button = styled(MaterialButton) `
+┊  ┊56┊  width: 100px;
+┊  ┊57┊  display: block !important;
+┊  ┊58┊  margin: auto !important;
+┊  ┊59┊  background-color: var(--secondary-bg) !important;
+┊  ┊60┊
+┊  ┊61┊  &[disabled] {
+┊  ┊62┊    color: #38a81c;
+┊  ┊63┊  }
+┊  ┊64┊
+┊  ┊65┊  &:not([disabled]) {
+┊  ┊66┊    color: white;
+┊  ┊67┊  }
+┊  ┊68┊` as typeof MaterialButton;
+┊  ┊69┊
+┊  ┊70┊export const ErrorMessage = styled.div `
+┊  ┊71┊  position: fixed;
+┊  ┊72┊  color: red;
+┊  ┊73┊  font-size: 15px;
+┊  ┊74┊  margin-top: 20px;
+┊  ┊75┊`;
```

##### Changed src&#x2F;components&#x2F;AuthScreen&#x2F;index.tsx
```diff
@@ -1,13 +1,13 @@
-┊ 1┊  ┊import MaterialButton from '@material-ui/core/Button';
-┊ 2┊  ┊import MaterialTextField from '@material-ui/core/TextField';
 ┊ 3┊ 1┊import React from 'react';
-┊ 4┊  ┊import { useCallback, useState } from 'react';
+┊  ┊ 2┊import { useMemo } from 'react';
+┊  ┊ 3┊import { Route } from 'react-router-dom';
 ┊ 5┊ 4┊import styled from 'styled-components';
-┊ 6┊  ┊import { signIn } from '../../services/auth.service';
+┊  ┊ 5┊import AnimatedSwitch from '../AnimatedSwitch';
+┊  ┊ 6┊import SignInForm from './SignInForm';
+┊  ┊ 7┊import SignUpForm from './SignUpForm';
 ┊ 7┊ 8┊import { RouteComponentProps } from 'react-router-dom';
 ┊ 8┊ 9┊
 ┊ 9┊10┊const Container = styled.div `
-┊10┊  ┊  height: 100%;
 ┊11┊11┊  background: radial-gradient(rgb(34, 65, 67), rgb(17, 48, 50)),
 ┊12┊12┊    url(/assets/chat-background.jpg) no-repeat;
 ┊13┊13┊  background-size: cover;
```
```diff
@@ -40,149 +40,48 @@
 ┊ 40┊ 40┊  bottom: 10px;
 ┊ 41┊ 41┊  left: 10px;
 ┊ 42┊ 42┊
-┊ 43┊   ┊  a {
+┊   ┊ 43┊  label {
 ┊ 44┊ 44┊    color: var(--secondary-bg);
 ┊ 45┊ 45┊  }
 ┊ 46┊ 46┊`;
 ┊ 47┊ 47┊
-┊ 48┊   ┊const SignInForm = styled.div `
-┊ 49┊   ┊  height: calc(100% - 265px);
-┊ 50┊   ┊`;
-┊ 51┊   ┊
-┊ 52┊   ┊const ActualForm = styled.form `
-┊ 53┊   ┊  padding: 20px;
-┊ 54┊   ┊`;
-┊ 55┊   ┊
-┊ 56┊   ┊const Section = styled.div `
-┊ 57┊   ┊  width: 100%;
-┊ 58┊   ┊  padding-bottom: 35px;
-┊ 59┊   ┊`;
-┊ 60┊   ┊
-┊ 61┊   ┊const Legend = styled.legend `
-┊ 62┊   ┊  font-weight: bold;
-┊ 63┊   ┊  color: white;
-┊ 64┊   ┊`;
-┊ 65┊   ┊
-┊ 66┊   ┊// eslint-disable-next-line
-┊ 67┊   ┊const Label = styled.label `
-┊ 68┊   ┊  color: white !important;
-┊ 69┊   ┊`;
-┊ 70┊   ┊
-┊ 71┊   ┊// eslint-disable-next-line
-┊ 72┊   ┊const Input = styled.input `
-┊ 73┊   ┊  color: white;
-┊ 74┊   ┊
-┊ 75┊   ┊  &::placeholder {
-┊ 76┊   ┊    color: var(--primary-bg);
-┊ 77┊   ┊  }
-┊ 78┊   ┊`;
-┊ 79┊   ┊
-┊ 80┊   ┊const TextField = styled(MaterialTextField) `
-┊ 81┊   ┊  width: 100%;
-┊ 82┊   ┊  position: relative;
-┊ 83┊   ┊
-┊ 84┊   ┊  > div::before {
-┊ 85┊   ┊    border-color: white !important;
-┊ 86┊   ┊  }
-┊ 87┊   ┊
-┊ 88┊   ┊  input {
-┊ 89┊   ┊    color: white !important;
-┊ 90┊   ┊
-┊ 91┊   ┊    &::placeholder {
-┊ 92┊   ┊      color: var(--primary-bg) !important;
+┊   ┊ 48┊const AuthScreen: React.FC<RouteComponentProps<any>> = ({ history, location }) => {
+┊   ┊ 49┊  const alternative = useMemo(() => {
+┊   ┊ 50┊    if (location.pathname === '/sign-in') {
+┊   ┊ 51┊      const handleSignUp = () => {
+┊   ┊ 52┊        history.replace('/sign-up');
+┊   ┊ 53┊      };
+┊   ┊ 54┊
+┊   ┊ 55┊      return (
+┊   ┊ 56┊        <Alternative>
+┊   ┊ 57┊          Don't have an account yet? <label onClick={handleSignUp}>Sign up!</label>
+┊   ┊ 58┊        </Alternative>
+┊   ┊ 59┊      );
 ┊ 93┊ 60┊    }
-┊ 94┊   ┊  }
-┊ 95┊   ┊
-┊ 96┊   ┊  label {
-┊ 97┊   ┊    color: white !important;
-┊ 98┊   ┊  }
-┊ 99┊   ┊` as typeof MaterialTextField;
-┊100┊   ┊
-┊101┊   ┊const Button = styled(MaterialButton) `
-┊102┊   ┊  width: 100px;
-┊103┊   ┊  display: block !important;
-┊104┊   ┊  margin: auto !important;
-┊105┊   ┊  background-color: var(--secondary-bg) !important;
-┊106┊   ┊
-┊107┊   ┊  &[disabled] {
-┊108┊   ┊    color: #38a81c;
-┊109┊   ┊  }
-┊110┊   ┊
-┊111┊   ┊  &:not([disabled]) {
-┊112┊   ┊    color: white;
-┊113┊   ┊  }
-┊114┊   ┊` as typeof MaterialButton;
-┊115┊   ┊
-┊116┊   ┊const AuthScreen: React.FC<RouteComponentProps<any>> = ({ history }) => {
-┊117┊   ┊  const [username, setUsername] = useState('');
-┊118┊   ┊  const [password, setPassword] = useState('');
-┊119┊   ┊  // eslint-disable-next-line
-┊120┊   ┊  const [error, setError] = useState('');
-┊121┊   ┊
-┊122┊   ┊  const onUsernameChange = useCallback(({ target }) => {
-┊123┊   ┊    setError('');
-┊124┊   ┊    setUsername(target.value);
-┊125┊   ┊  }, []);
-┊126┊   ┊
-┊127┊   ┊  const onPasswordChange = useCallback(({ target }) => {
-┊128┊   ┊    setError('');
-┊129┊   ┊    setPassword(target.value);
-┊130┊   ┊  }, []);
-┊131┊   ┊
-┊132┊   ┊  const maySignIn = useCallback(() => {
-┊133┊   ┊    return !!(username && password);
-┊134┊   ┊  }, [username, password]);
-┊135┊   ┊
-┊136┊   ┊  const handleSignIn = useCallback(() => {
-┊137┊   ┊    signIn({ username, password })
-┊138┊   ┊      .then(() => {
-┊139┊   ┊        history.push('/chats')
-┊140┊   ┊      })
-┊141┊   ┊      .catch(error => {
-┊142┊   ┊        setError(error.message || error)
-┊143┊   ┊      });
-┊144┊   ┊  }, [username, password, history]);
+┊   ┊ 61┊    else {
+┊   ┊ 62┊      const handleSignIn = () => {
+┊   ┊ 63┊        history.replace('/sign-in');
+┊   ┊ 64┊      };
+┊   ┊ 65┊
+┊   ┊ 66┊      return (
+┊   ┊ 67┊        <Alternative>
+┊   ┊ 68┊          Already have an accout? <label onClick={handleSignIn}>Sign in!</label>
+┊   ┊ 69┊        </Alternative>
+┊   ┊ 70┊      );
+┊   ┊ 71┊    }
+┊   ┊ 72┊  }, [location.pathname, history]);
 ┊145┊ 73┊
 ┊146┊ 74┊  return (
-┊147┊   ┊    <Container>
-┊148┊   ┊      <Intro>
+┊   ┊ 75┊    <Container className="AuthScreen Screen">
+┊   ┊ 76┊      <Intro className="AuthScreen-intro">
 ┊149┊ 77┊        <Icon src="assets/whatsapp-icon.png" className="AuthScreen-icon" />
 ┊150┊ 78┊        <Title className="AuthScreen-title">WhatsApp</Title>
 ┊151┊ 79┊      </Intro>
-┊152┊   ┊      <SignInForm>
-┊153┊   ┊        <ActualForm>
-┊154┊   ┊          <Legend>Sign in</Legend>
-┊155┊   ┊          <Section>
-┊156┊   ┊            <TextField
-┊157┊   ┊              className="AuthScreen-text-field"
-┊158┊   ┊              label="Username"
-┊159┊   ┊              value={username}
-┊160┊   ┊              onChange={onUsernameChange}
-┊161┊   ┊              margin="normal"
-┊162┊   ┊              placeholder="Enter your username"
-┊163┊   ┊            />
-┊164┊   ┊            <TextField
-┊165┊   ┊              className="AuthScreen-text-field"
-┊166┊   ┊              label="Password"
-┊167┊   ┊              type="password"
-┊168┊   ┊              value={password}
-┊169┊   ┊              onChange={onPasswordChange}
-┊170┊   ┊              margin="normal"
-┊171┊   ┊              placeholder="Enter your password"
-┊172┊   ┊            />
-┊173┊   ┊          </Section>
-┊174┊   ┊          <Button
-┊175┊   ┊            data-testid="sign-in-button"
-┊176┊   ┊            type="button"
-┊177┊   ┊            color="secondary"
-┊178┊   ┊            variant="contained"
-┊179┊   ┊            disabled={!maySignIn()}
-┊180┊   ┊            onClick={handleSignIn}
-┊181┊   ┊          >
-┊182┊   ┊            Sign in
-┊183┊   ┊          </Button>
-┊184┊   ┊        </ActualForm>
-┊185┊   ┊      </SignInForm>
+┊   ┊ 80┊      <AnimatedSwitch>
+┊   ┊ 81┊        <Route exact path="/sign-in" component={SignInForm} />
+┊   ┊ 82┊        <Route exact path="/sign-up" component={SignUpForm} />
+┊   ┊ 83┊      </AnimatedSwitch>
+┊   ┊ 84┊      {alternative}
 ┊186┊ 85┊    </Container>
 ┊187┊ 86┊  );
 ┊188┊ 87┊};
```

[}]: #

And then we will make the necessary changes in the `AuthScreen`:

[{]: <helper> (diffStep 13.6 module="client")

#### Client Step 13.6: Split AuthScreen into SignInForm and SignUpForm

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;SignInForm.test.tsx
```diff
@@ -0,0 +1,81 @@
+┊  ┊ 1┊import { createMemoryHistory } from 'history';
+┊  ┊ 2┊import React from 'react';
+┊  ┊ 3┊import { cleanup, render, fireEvent, wait, waitForElement } from 'react-testing-library';
+┊  ┊ 4┊import SignInForm from './SignInForm';
+┊  ┊ 5┊
+┊  ┊ 6┊describe('SignInForm', () => {
+┊  ┊ 7┊  afterEach(cleanup);
+┊  ┊ 8┊  afterEach(() => fetch.resetMocks());
+┊  ┊ 9┊
+┊  ┊10┊  it('enables sign-in button when filled in', async () => {
+┊  ┊11┊    const history = createMemoryHistory();
+┊  ┊12┊
+┊  ┊13┊    {
+┊  ┊14┊      const { container, getByTestId } = render(<SignInForm history={history} />);
+┊  ┊15┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊16┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊17┊      const signInButton = getByTestId('sign-in-button') as HTMLButtonElement;
+┊  ┊18┊
+┊  ┊19┊      expect(signInButton.disabled).toEqual(true);
+┊  ┊20┊
+┊  ┊21┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊22┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊23┊
+┊  ┊24┊      await waitForElement(() => usernameInput);
+┊  ┊25┊      await waitForElement(() => passwordInput);
+┊  ┊26┊
+┊  ┊27┊      expect(signInButton.disabled).toEqual(false);
+┊  ┊28┊    }
+┊  ┊29┊  });
+┊  ┊30┊
+┊  ┊31┊  it('prints server error if input was wrong', async () => {
+┊  ┊32┊    const history = createMemoryHistory();
+┊  ┊33┊
+┊  ┊34┊    fetchMock.mockRejectOnce(new Error('sign-in failed'));
+┊  ┊35┊
+┊  ┊36┊    {
+┊  ┊37┊      const { container, getByTestId } = render(<SignInForm history={history} />);
+┊  ┊38┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊39┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊40┊      const signInButton = getByTestId('sign-in-button') as HTMLButtonElement;
+┊  ┊41┊      const errorMessage = getByTestId('error-message');
+┊  ┊42┊
+┊  ┊43┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊44┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊45┊
+┊  ┊46┊      await waitForElement(() => usernameInput);
+┊  ┊47┊      await waitForElement(() => passwordInput);
+┊  ┊48┊
+┊  ┊49┊      fireEvent.click(signInButton);
+┊  ┊50┊
+┊  ┊51┊      await waitForElement(() => errorMessage);
+┊  ┊52┊
+┊  ┊53┊      expect(errorMessage.innerHTML).toEqual('sign-in failed');
+┊  ┊54┊    }
+┊  ┊55┊  });
+┊  ┊56┊
+┊  ┊57┊  it('navigates to /chats if everything went right', async () => {
+┊  ┊58┊    const history = createMemoryHistory();
+┊  ┊59┊
+┊  ┊60┊    fetchMock.mockResponseOnce('success');
+┊  ┊61┊
+┊  ┊62┊    {
+┊  ┊63┊      const { container, getByTestId } = render(<SignInForm history={history} />);
+┊  ┊64┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊65┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊66┊      const signInButton = getByTestId('sign-in-button') as HTMLButtonElement;
+┊  ┊67┊
+┊  ┊68┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊69┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊70┊
+┊  ┊71┊      await waitForElement(() => usernameInput);
+┊  ┊72┊      await waitForElement(() => passwordInput);
+┊  ┊73┊
+┊  ┊74┊      fireEvent.click(signInButton);
+┊  ┊75┊
+┊  ┊76┊      await wait(() =>
+┊  ┊77┊        expect(history.location.pathname).toEqual('/chats')
+┊  ┊78┊      );
+┊  ┊79┊    }
+┊  ┊80┊  });
+┊  ┊81┊});
```

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;SignInForm.tsx
```diff
@@ -0,0 +1,83 @@
+┊  ┊ 1┊import React from 'react';
+┊  ┊ 2┊import { useCallback, useState } from 'react';
+┊  ┊ 3┊import { signIn } from '../../services/auth.service';
+┊  ┊ 4┊import {
+┊  ┊ 5┊  SignForm,
+┊  ┊ 6┊  ActualForm,
+┊  ┊ 7┊  Legend,
+┊  ┊ 8┊  Section,
+┊  ┊ 9┊  TextField,
+┊  ┊10┊  Button,
+┊  ┊11┊  ErrorMessage,
+┊  ┊12┊} from './form-components';
+┊  ┊13┊import { RouteComponentProps } from 'react-router-dom';
+┊  ┊14┊
+┊  ┊15┊const SignInForm: React.FC<RouteComponentProps<any>> = ({ history }) => {
+┊  ┊16┊  const [username, setUsername] = useState('');
+┊  ┊17┊  const [password, setPassword] = useState('');
+┊  ┊18┊  const [error, setError] = useState('');
+┊  ┊19┊
+┊  ┊20┊  const onUsernameChange = useCallback(({ target }) => {
+┊  ┊21┊    setError('');
+┊  ┊22┊    setUsername(target.value);
+┊  ┊23┊  }, []);
+┊  ┊24┊
+┊  ┊25┊  const onPasswordChange = useCallback(({ target }) => {
+┊  ┊26┊    setError('');
+┊  ┊27┊    setPassword(target.value);
+┊  ┊28┊  }, []);
+┊  ┊29┊
+┊  ┊30┊  const maySignIn = useCallback(() => {
+┊  ┊31┊    return !!(username && password);
+┊  ┊32┊  }, [username, password]);
+┊  ┊33┊
+┊  ┊34┊  const handleSignIn = useCallback(() => {
+┊  ┊35┊    signIn({ username, password })
+┊  ┊36┊      .then(() => {
+┊  ┊37┊        history.replace('/chats')
+┊  ┊38┊      })
+┊  ┊39┊      .catch(error => {
+┊  ┊40┊        setError(error.message || error)
+┊  ┊41┊      });
+┊  ┊42┊  }, [username, password, history]);
+┊  ┊43┊
+┊  ┊44┊  return (
+┊  ┊45┊    <SignForm>
+┊  ┊46┊      <ActualForm>
+┊  ┊47┊        <Legend>Sign in</Legend>
+┊  ┊48┊        <Section style={{ width: '100%' }}>
+┊  ┊49┊          <TextField
+┊  ┊50┊            data-testid="username-input"
+┊  ┊51┊            label="Username"
+┊  ┊52┊            value={username}
+┊  ┊53┊            onChange={onUsernameChange}
+┊  ┊54┊            margin="normal"
+┊  ┊55┊            placeholder="Enter your username"
+┊  ┊56┊          />
+┊  ┊57┊          <TextField
+┊  ┊58┊            data-testid="password-input"
+┊  ┊59┊            label="Password"
+┊  ┊60┊            type="password"
+┊  ┊61┊            value={password}
+┊  ┊62┊            onChange={onPasswordChange}
+┊  ┊63┊            margin="normal"
+┊  ┊64┊            placeholder="Enter your password"
+┊  ┊65┊          />
+┊  ┊66┊        </Section>
+┊  ┊67┊        <Button
+┊  ┊68┊          data-testid="sign-in-button"
+┊  ┊69┊          type="button"
+┊  ┊70┊          color="secondary"
+┊  ┊71┊          variant="contained"
+┊  ┊72┊          disabled={!maySignIn()}
+┊  ┊73┊          onClick={handleSignIn}
+┊  ┊74┊        >
+┊  ┊75┊          Sign in
+┊  ┊76┊        </Button>
+┊  ┊77┊        <ErrorMessage data-testid="error-message">{error}</ErrorMessage>
+┊  ┊78┊      </ActualForm>
+┊  ┊79┊    </SignForm>
+┊  ┊80┊  );
+┊  ┊81┊};
+┊  ┊82┊
+┊  ┊83┊export default SignInForm;
```

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;SignUpForm.test.tsx
```diff
@@ -0,0 +1,99 @@
+┊  ┊ 1┊import { createMemoryHistory } from 'history';
+┊  ┊ 2┊import React from 'react';
+┊  ┊ 3┊import { cleanup, render, fireEvent, wait, waitForElement } from 'react-testing-library';
+┊  ┊ 4┊import SignUpForm from './SignUpForm';
+┊  ┊ 5┊
+┊  ┊ 6┊describe('SignUpForm', () => {
+┊  ┊ 7┊  afterEach(cleanup);
+┊  ┊ 8┊  afterEach(() => fetch.resetMocks());
+┊  ┊ 9┊
+┊  ┊10┊  it('enables sign-up button when filled in', async () => {
+┊  ┊11┊    const history = createMemoryHistory();
+┊  ┊12┊
+┊  ┊13┊    {
+┊  ┊14┊      const { container, getByTestId } = render(<SignUpForm history={history} />);
+┊  ┊15┊      const nameInput = getByTestId('name-input').querySelector('input');
+┊  ┊16┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊17┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊18┊      const passwordConfirmInput = getByTestId('password-confirm-input').querySelector('input');
+┊  ┊19┊      const signUpButton = getByTestId('sign-up-button') as HTMLButtonElement;
+┊  ┊20┊
+┊  ┊21┊      expect(signUpButton.disabled).toEqual(true);
+┊  ┊22┊
+┊  ┊23┊      fireEvent.change(nameInput, { target: { value: 'User Name' } });
+┊  ┊24┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊25┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊26┊      fireEvent.change(passwordConfirmInput, { target: { value: 'password' } });
+┊  ┊27┊
+┊  ┊28┊      await waitForElement(() => nameInput);
+┊  ┊29┊      await waitForElement(() => usernameInput);
+┊  ┊30┊      await waitForElement(() => passwordInput);
+┊  ┊31┊      await waitForElement(() => passwordConfirmInput);
+┊  ┊32┊
+┊  ┊33┊      expect(signUpButton.disabled).toEqual(false);
+┊  ┊34┊    }
+┊  ┊35┊  });
+┊  ┊36┊
+┊  ┊37┊  it('prints server error if input was wrong', async () => {
+┊  ┊38┊    const history = createMemoryHistory();
+┊  ┊39┊
+┊  ┊40┊    fetchMock.mockRejectOnce(new Error('sign-up failed'));
+┊  ┊41┊
+┊  ┊42┊    {
+┊  ┊43┊      const { container, getByTestId } = render(<SignUpForm history={history} />);
+┊  ┊44┊      const nameInput = getByTestId('name-input').querySelector('input');
+┊  ┊45┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊46┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊47┊      const passwordConfirmInput = getByTestId('password-confirm-input').querySelector('input');
+┊  ┊48┊      const signUpButton = getByTestId('sign-up-button') as HTMLButtonElement;
+┊  ┊49┊      const errorMessage = getByTestId('error-message');
+┊  ┊50┊
+┊  ┊51┊      fireEvent.change(nameInput, { target: { value: 'User Name' } });
+┊  ┊52┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊53┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊54┊      fireEvent.change(passwordConfirmInput, { target: { value: 'password' } });
+┊  ┊55┊
+┊  ┊56┊      await waitForElement(() => nameInput);
+┊  ┊57┊      await waitForElement(() => usernameInput);
+┊  ┊58┊      await waitForElement(() => passwordInput);
+┊  ┊59┊      await waitForElement(() => passwordConfirmInput);
+┊  ┊60┊
+┊  ┊61┊      fireEvent.click(signUpButton);
+┊  ┊62┊
+┊  ┊63┊      await waitForElement(() => errorMessage);
+┊  ┊64┊
+┊  ┊65┊      expect(errorMessage.innerHTML).toEqual('sign-up failed');
+┊  ┊66┊    }
+┊  ┊67┊  });
+┊  ┊68┊
+┊  ┊69┊  it('navigates to /sign-in if everything went right', async () => {
+┊  ┊70┊    const history = createMemoryHistory();
+┊  ┊71┊
+┊  ┊72┊    fetchMock.mockResponseOnce('success');
+┊  ┊73┊
+┊  ┊74┊    {
+┊  ┊75┊      const { container, getByTestId } = render(<SignUpForm history={history} />);
+┊  ┊76┊      const nameInput = getByTestId('name-input').querySelector('input');
+┊  ┊77┊      const usernameInput = getByTestId('username-input').querySelector('input');
+┊  ┊78┊      const passwordInput = getByTestId('password-input').querySelector('input');
+┊  ┊79┊      const passwordConfirmInput = getByTestId('password-confirm-input').querySelector('input');
+┊  ┊80┊      const signUpButton = getByTestId('sign-up-button') as HTMLButtonElement;
+┊  ┊81┊
+┊  ┊82┊      fireEvent.change(nameInput, { target: { value: 'User Name' } });
+┊  ┊83┊      fireEvent.change(usernameInput, { target: { value: 'username' } });
+┊  ┊84┊      fireEvent.change(passwordInput, { target: { value: 'password' } });
+┊  ┊85┊      fireEvent.change(passwordConfirmInput, { target: { value: 'password' } });
+┊  ┊86┊
+┊  ┊87┊      await waitForElement(() => nameInput);
+┊  ┊88┊      await waitForElement(() => usernameInput);
+┊  ┊89┊      await waitForElement(() => passwordInput);
+┊  ┊90┊      await waitForElement(() => passwordConfirmInput);
+┊  ┊91┊
+┊  ┊92┊      fireEvent.click(signUpButton);
+┊  ┊93┊
+┊  ┊94┊      await wait(() =>
+┊  ┊95┊        expect(history.location.pathname).toEqual('/sign-in')
+┊  ┊96┊      );
+┊  ┊97┊    }
+┊  ┊98┊  });
+┊  ┊99┊});
```

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;SignUpForm.tsx
```diff
@@ -0,0 +1,126 @@
+┊   ┊  1┊import React from 'react';
+┊   ┊  2┊import { useCallback, useState } from 'react';
+┊   ┊  3┊import { signUp } from '../../services/auth.service';
+┊   ┊  4┊import {
+┊   ┊  5┊  SignForm,
+┊   ┊  6┊  ActualForm,
+┊   ┊  7┊  Legend,
+┊   ┊  8┊  Section,
+┊   ┊  9┊  TextField,
+┊   ┊ 10┊  Button,
+┊   ┊ 11┊  ErrorMessage,
+┊   ┊ 12┊} from './form-components';
+┊   ┊ 13┊import { RouteComponentProps } from 'react-router-dom';
+┊   ┊ 14┊
+┊   ┊ 15┊const SignUpForm: React.FC<RouteComponentProps<any>> = ({ history }) => {
+┊   ┊ 16┊  const [name, setName] = useState('');
+┊   ┊ 17┊  const [username, setUsername] = useState('');
+┊   ┊ 18┊  const [password, setPassword] = useState('');
+┊   ┊ 19┊  const [passwordConfirm, setPasswordConfirm] = useState('');
+┊   ┊ 20┊  const [error, setError] = useState('');
+┊   ┊ 21┊
+┊   ┊ 22┊  const updateName = useCallback(({ target }) => {
+┊   ┊ 23┊    setError('');
+┊   ┊ 24┊    setName(target.value);
+┊   ┊ 25┊  }, []);
+┊   ┊ 26┊
+┊   ┊ 27┊  const updateUsername = useCallback(({ target }) => {
+┊   ┊ 28┊    setError('');
+┊   ┊ 29┊    setUsername(target.value);
+┊   ┊ 30┊  }, []);
+┊   ┊ 31┊
+┊   ┊ 32┊  const updatePassword = useCallback(({ target }) => {
+┊   ┊ 33┊    setError('');
+┊   ┊ 34┊    setPassword(target.value);
+┊   ┊ 35┊  }, []);
+┊   ┊ 36┊
+┊   ┊ 37┊  const updatePasswordConfirm = useCallback(({ target }) => {
+┊   ┊ 38┊    setError('');
+┊   ┊ 39┊    setPasswordConfirm(target.value);
+┊   ┊ 40┊  }, []);
+┊   ┊ 41┊
+┊   ┊ 42┊  const maySignUp = useCallback(() => {
+┊   ┊ 43┊    return !!(name && username && password && password === passwordConfirm);
+┊   ┊ 44┊  }, [name, username, password, passwordConfirm]);
+┊   ┊ 45┊
+┊   ┊ 46┊  const handleSignUp = useCallback(() => {
+┊   ┊ 47┊    signUp({ username, password, passwordConfirm, name })
+┊   ┊ 48┊      .then(() => {
+┊   ┊ 49┊        history.replace('/sign-in')
+┊   ┊ 50┊      })
+┊   ┊ 51┊      .catch(error => {
+┊   ┊ 52┊        setError(error.message || error)
+┊   ┊ 53┊      });
+┊   ┊ 54┊  }, [name, username, password, passwordConfirm, history]);
+┊   ┊ 55┊
+┊   ┊ 56┊  return (
+┊   ┊ 57┊    <SignForm>
+┊   ┊ 58┊      <ActualForm>
+┊   ┊ 59┊        <Legend>Sign up</Legend>
+┊   ┊ 60┊        <Section
+┊   ┊ 61┊          style={{
+┊   ┊ 62┊            float: 'left',
+┊   ┊ 63┊            width: 'calc(50% - 10px)',
+┊   ┊ 64┊            paddingRight: '10px',
+┊   ┊ 65┊          }}
+┊   ┊ 66┊        >
+┊   ┊ 67┊          <TextField
+┊   ┊ 68┊            data-testid="name-input"
+┊   ┊ 69┊            label="Name"
+┊   ┊ 70┊            value={name}
+┊   ┊ 71┊            onChange={updateName}
+┊   ┊ 72┊            autoComplete="off"
+┊   ┊ 73┊            margin="normal"
+┊   ┊ 74┊          />
+┊   ┊ 75┊          <TextField
+┊   ┊ 76┊            data-testid="username-input"
+┊   ┊ 77┊            label="Username"
+┊   ┊ 78┊            value={username}
+┊   ┊ 79┊            onChange={updateUsername}
+┊   ┊ 80┊            autoComplete="off"
+┊   ┊ 81┊            margin="normal"
+┊   ┊ 82┊          />
+┊   ┊ 83┊        </Section>
+┊   ┊ 84┊        <Section
+┊   ┊ 85┊          style={{
+┊   ┊ 86┊            float: 'right',
+┊   ┊ 87┊            width: 'calc(50% - 10px)',
+┊   ┊ 88┊            paddingLeft: '10px',
+┊   ┊ 89┊          }}
+┊   ┊ 90┊        >
+┊   ┊ 91┊          <TextField
+┊   ┊ 92┊            data-testid="password-input"
+┊   ┊ 93┊            label="Password"
+┊   ┊ 94┊            type="password"
+┊   ┊ 95┊            value={password}
+┊   ┊ 96┊            onChange={updatePassword}
+┊   ┊ 97┊            autoComplete="off"
+┊   ┊ 98┊            margin="normal"
+┊   ┊ 99┊          />
+┊   ┊100┊          <TextField
+┊   ┊101┊            data-testid="password-confirm-input"
+┊   ┊102┊            label="Confirm password"
+┊   ┊103┊            type="password"
+┊   ┊104┊            value={passwordConfirm}
+┊   ┊105┊            onChange={updatePasswordConfirm}
+┊   ┊106┊            autoComplete="off"
+┊   ┊107┊            margin="normal"
+┊   ┊108┊          />
+┊   ┊109┊        </Section>
+┊   ┊110┊        <Button
+┊   ┊111┊          data-testid="sign-up-button"
+┊   ┊112┊          type="button"
+┊   ┊113┊          color="secondary"
+┊   ┊114┊          variant="contained"
+┊   ┊115┊          disabled={!maySignUp()}
+┊   ┊116┊          onClick={handleSignUp}
+┊   ┊117┊        >
+┊   ┊118┊          Sign up
+┊   ┊119┊        </Button>
+┊   ┊120┊        <ErrorMessage data-testid="error-message">{error}</ErrorMessage>
+┊   ┊121┊      </ActualForm>
+┊   ┊122┊    </SignForm>
+┊   ┊123┊  );
+┊   ┊124┊};
+┊   ┊125┊
+┊   ┊126┊export default SignUpForm;🚫↵
```

##### Added src&#x2F;components&#x2F;AuthScreen&#x2F;form-components.ts
```diff
@@ -0,0 +1,75 @@
+┊  ┊ 1┊import MaterialButton from '@material-ui/core/Button';
+┊  ┊ 2┊import MaterialTextField from '@material-ui/core/TextField';
+┊  ┊ 3┊import styled from 'styled-components';
+┊  ┊ 4┊
+┊  ┊ 5┊export const SignForm = styled.div `
+┊  ┊ 6┊  height: calc(100% - 265px);
+┊  ┊ 7┊`;
+┊  ┊ 8┊
+┊  ┊ 9┊export const ActualForm = styled.form `
+┊  ┊10┊  padding: 20px;
+┊  ┊11┊`;
+┊  ┊12┊
+┊  ┊13┊export const Section = styled.div `
+┊  ┊14┊  padding-bottom: 35px;
+┊  ┊15┊`;
+┊  ┊16┊
+┊  ┊17┊export const Legend = styled.legend `
+┊  ┊18┊  font-weight: bold;
+┊  ┊19┊  color: white;
+┊  ┊20┊`;
+┊  ┊21┊
+┊  ┊22┊export const Label = styled.label `
+┊  ┊23┊  color: white !important;
+┊  ┊24┊`;
+┊  ┊25┊
+┊  ┊26┊export const Input = styled.input `
+┊  ┊27┊  color: white;
+┊  ┊28┊
+┊  ┊29┊  &::placeholder {
+┊  ┊30┊    color: var(--primary-bg);
+┊  ┊31┊  }
+┊  ┊32┊`;
+┊  ┊33┊
+┊  ┊34┊export const TextField = styled(MaterialTextField) `
+┊  ┊35┊  width: 100%;
+┊  ┊36┊  position: relative;
+┊  ┊37┊
+┊  ┊38┊  > div::before {
+┊  ┊39┊    border-color: white !important;
+┊  ┊40┊  }
+┊  ┊41┊
+┊  ┊42┊  input {
+┊  ┊43┊    color: white !important;
+┊  ┊44┊
+┊  ┊45┊    &::placeholder {
+┊  ┊46┊      color: var(--primary-bg) !important;
+┊  ┊47┊    }
+┊  ┊48┊  }
+┊  ┊49┊
+┊  ┊50┊  label {
+┊  ┊51┊    color: white !important;
+┊  ┊52┊  }
+┊  ┊53┊` as typeof MaterialTextField;
+┊  ┊54┊
+┊  ┊55┊export const Button = styled(MaterialButton) `
+┊  ┊56┊  width: 100px;
+┊  ┊57┊  display: block !important;
+┊  ┊58┊  margin: auto !important;
+┊  ┊59┊  background-color: var(--secondary-bg) !important;
+┊  ┊60┊
+┊  ┊61┊  &[disabled] {
+┊  ┊62┊    color: #38a81c;
+┊  ┊63┊  }
+┊  ┊64┊
+┊  ┊65┊  &:not([disabled]) {
+┊  ┊66┊    color: white;
+┊  ┊67┊  }
+┊  ┊68┊` as typeof MaterialButton;
+┊  ┊69┊
+┊  ┊70┊export const ErrorMessage = styled.div `
+┊  ┊71┊  position: fixed;
+┊  ┊72┊  color: red;
+┊  ┊73┊  font-size: 15px;
+┊  ┊74┊  margin-top: 20px;
+┊  ┊75┊`;
```

##### Changed src&#x2F;components&#x2F;AuthScreen&#x2F;index.tsx
```diff
@@ -1,13 +1,13 @@
-┊ 1┊  ┊import MaterialButton from '@material-ui/core/Button';
-┊ 2┊  ┊import MaterialTextField from '@material-ui/core/TextField';
 ┊ 3┊ 1┊import React from 'react';
-┊ 4┊  ┊import { useCallback, useState } from 'react';
+┊  ┊ 2┊import { useMemo } from 'react';
+┊  ┊ 3┊import { Route } from 'react-router-dom';
 ┊ 5┊ 4┊import styled from 'styled-components';
-┊ 6┊  ┊import { signIn } from '../../services/auth.service';
+┊  ┊ 5┊import AnimatedSwitch from '../AnimatedSwitch';
+┊  ┊ 6┊import SignInForm from './SignInForm';
+┊  ┊ 7┊import SignUpForm from './SignUpForm';
 ┊ 7┊ 8┊import { RouteComponentProps } from 'react-router-dom';
 ┊ 8┊ 9┊
 ┊ 9┊10┊const Container = styled.div `
-┊10┊  ┊  height: 100%;
 ┊11┊11┊  background: radial-gradient(rgb(34, 65, 67), rgb(17, 48, 50)),
 ┊12┊12┊    url(/assets/chat-background.jpg) no-repeat;
 ┊13┊13┊  background-size: cover;
```
```diff
@@ -40,149 +40,48 @@
 ┊ 40┊ 40┊  bottom: 10px;
 ┊ 41┊ 41┊  left: 10px;
 ┊ 42┊ 42┊
-┊ 43┊   ┊  a {
+┊   ┊ 43┊  label {
 ┊ 44┊ 44┊    color: var(--secondary-bg);
 ┊ 45┊ 45┊  }
 ┊ 46┊ 46┊`;
 ┊ 47┊ 47┊
-┊ 48┊   ┊const SignInForm = styled.div `
-┊ 49┊   ┊  height: calc(100% - 265px);
-┊ 50┊   ┊`;
-┊ 51┊   ┊
-┊ 52┊   ┊const ActualForm = styled.form `
-┊ 53┊   ┊  padding: 20px;
-┊ 54┊   ┊`;
-┊ 55┊   ┊
-┊ 56┊   ┊const Section = styled.div `
-┊ 57┊   ┊  width: 100%;
-┊ 58┊   ┊  padding-bottom: 35px;
-┊ 59┊   ┊`;
-┊ 60┊   ┊
-┊ 61┊   ┊const Legend = styled.legend `
-┊ 62┊   ┊  font-weight: bold;
-┊ 63┊   ┊  color: white;
-┊ 64┊   ┊`;
-┊ 65┊   ┊
-┊ 66┊   ┊// eslint-disable-next-line
-┊ 67┊   ┊const Label = styled.label `
-┊ 68┊   ┊  color: white !important;
-┊ 69┊   ┊`;
-┊ 70┊   ┊
-┊ 71┊   ┊// eslint-disable-next-line
-┊ 72┊   ┊const Input = styled.input `
-┊ 73┊   ┊  color: white;
-┊ 74┊   ┊
-┊ 75┊   ┊  &::placeholder {
-┊ 76┊   ┊    color: var(--primary-bg);
-┊ 77┊   ┊  }
-┊ 78┊   ┊`;
-┊ 79┊   ┊
-┊ 80┊   ┊const TextField = styled(MaterialTextField) `
-┊ 81┊   ┊  width: 100%;
-┊ 82┊   ┊  position: relative;
-┊ 83┊   ┊
-┊ 84┊   ┊  > div::before {
-┊ 85┊   ┊    border-color: white !important;
-┊ 86┊   ┊  }
-┊ 87┊   ┊
-┊ 88┊   ┊  input {
-┊ 89┊   ┊    color: white !important;
-┊ 90┊   ┊
-┊ 91┊   ┊    &::placeholder {
-┊ 92┊   ┊      color: var(--primary-bg) !important;
+┊   ┊ 48┊const AuthScreen: React.FC<RouteComponentProps<any>> = ({ history, location }) => {
+┊   ┊ 49┊  const alternative = useMemo(() => {
+┊   ┊ 50┊    if (location.pathname === '/sign-in') {
+┊   ┊ 51┊      const handleSignUp = () => {
+┊   ┊ 52┊        history.replace('/sign-up');
+┊   ┊ 53┊      };
+┊   ┊ 54┊
+┊   ┊ 55┊      return (
+┊   ┊ 56┊        <Alternative>
+┊   ┊ 57┊          Don't have an account yet? <label onClick={handleSignUp}>Sign up!</label>
+┊   ┊ 58┊        </Alternative>
+┊   ┊ 59┊      );
 ┊ 93┊ 60┊    }
-┊ 94┊   ┊  }
-┊ 95┊   ┊
-┊ 96┊   ┊  label {
-┊ 97┊   ┊    color: white !important;
-┊ 98┊   ┊  }
-┊ 99┊   ┊` as typeof MaterialTextField;
-┊100┊   ┊
-┊101┊   ┊const Button = styled(MaterialButton) `
-┊102┊   ┊  width: 100px;
-┊103┊   ┊  display: block !important;
-┊104┊   ┊  margin: auto !important;
-┊105┊   ┊  background-color: var(--secondary-bg) !important;
-┊106┊   ┊
-┊107┊   ┊  &[disabled] {
-┊108┊   ┊    color: #38a81c;
-┊109┊   ┊  }
-┊110┊   ┊
-┊111┊   ┊  &:not([disabled]) {
-┊112┊   ┊    color: white;
-┊113┊   ┊  }
-┊114┊   ┊` as typeof MaterialButton;
-┊115┊   ┊
-┊116┊   ┊const AuthScreen: React.FC<RouteComponentProps<any>> = ({ history }) => {
-┊117┊   ┊  const [username, setUsername] = useState('');
-┊118┊   ┊  const [password, setPassword] = useState('');
-┊119┊   ┊  // eslint-disable-next-line
-┊120┊   ┊  const [error, setError] = useState('');
-┊121┊   ┊
-┊122┊   ┊  const onUsernameChange = useCallback(({ target }) => {
-┊123┊   ┊    setError('');
-┊124┊   ┊    setUsername(target.value);
-┊125┊   ┊  }, []);
-┊126┊   ┊
-┊127┊   ┊  const onPasswordChange = useCallback(({ target }) => {
-┊128┊   ┊    setError('');
-┊129┊   ┊    setPassword(target.value);
-┊130┊   ┊  }, []);
-┊131┊   ┊
-┊132┊   ┊  const maySignIn = useCallback(() => {
-┊133┊   ┊    return !!(username && password);
-┊134┊   ┊  }, [username, password]);
-┊135┊   ┊
-┊136┊   ┊  const handleSignIn = useCallback(() => {
-┊137┊   ┊    signIn({ username, password })
-┊138┊   ┊      .then(() => {
-┊139┊   ┊        history.push('/chats')
-┊140┊   ┊      })
-┊141┊   ┊      .catch(error => {
-┊142┊   ┊        setError(error.message || error)
-┊143┊   ┊      });
-┊144┊   ┊  }, [username, password, history]);
+┊   ┊ 61┊    else {
+┊   ┊ 62┊      const handleSignIn = () => {
+┊   ┊ 63┊        history.replace('/sign-in');
+┊   ┊ 64┊      };
+┊   ┊ 65┊
+┊   ┊ 66┊      return (
+┊   ┊ 67┊        <Alternative>
+┊   ┊ 68┊          Already have an accout? <label onClick={handleSignIn}>Sign in!</label>
+┊   ┊ 69┊        </Alternative>
+┊   ┊ 70┊      );
+┊   ┊ 71┊    }
+┊   ┊ 72┊  }, [location.pathname, history]);
 ┊145┊ 73┊
 ┊146┊ 74┊  return (
-┊147┊   ┊    <Container>
-┊148┊   ┊      <Intro>
+┊   ┊ 75┊    <Container className="AuthScreen Screen">
+┊   ┊ 76┊      <Intro className="AuthScreen-intro">
 ┊149┊ 77┊        <Icon src="assets/whatsapp-icon.png" className="AuthScreen-icon" />
 ┊150┊ 78┊        <Title className="AuthScreen-title">WhatsApp</Title>
 ┊151┊ 79┊      </Intro>
-┊152┊   ┊      <SignInForm>
-┊153┊   ┊        <ActualForm>
-┊154┊   ┊          <Legend>Sign in</Legend>
-┊155┊   ┊          <Section>
-┊156┊   ┊            <TextField
-┊157┊   ┊              className="AuthScreen-text-field"
-┊158┊   ┊              label="Username"
-┊159┊   ┊              value={username}
-┊160┊   ┊              onChange={onUsernameChange}
-┊161┊   ┊              margin="normal"
-┊162┊   ┊              placeholder="Enter your username"
-┊163┊   ┊            />
-┊164┊   ┊            <TextField
-┊165┊   ┊              className="AuthScreen-text-field"
-┊166┊   ┊              label="Password"
-┊167┊   ┊              type="password"
-┊168┊   ┊              value={password}
-┊169┊   ┊              onChange={onPasswordChange}
-┊170┊   ┊              margin="normal"
-┊171┊   ┊              placeholder="Enter your password"
-┊172┊   ┊            />
-┊173┊   ┊          </Section>
-┊174┊   ┊          <Button
-┊175┊   ┊            data-testid="sign-in-button"
-┊176┊   ┊            type="button"
-┊177┊   ┊            color="secondary"
-┊178┊   ┊            variant="contained"
-┊179┊   ┊            disabled={!maySignIn()}
-┊180┊   ┊            onClick={handleSignIn}
-┊181┊   ┊          >
-┊182┊   ┊            Sign in
-┊183┊   ┊          </Button>
-┊184┊   ┊        </ActualForm>
-┊185┊   ┊      </SignInForm>
+┊   ┊ 80┊      <AnimatedSwitch>
+┊   ┊ 81┊        <Route exact path="/sign-in" component={SignInForm} />
+┊   ┊ 82┊        <Route exact path="/sign-up" component={SignUpForm} />
+┊   ┊ 83┊      </AnimatedSwitch>
+┊   ┊ 84┊      {alternative}
 ┊186┊ 85┊    </Container>
 ┊187┊ 86┊  );
 ┊188┊ 87┊};
```

[}]: #

> Note how we used the `/sign-(in|up)` pattern to define the `signUp` mutation. This is because the request will be further redirected in the `AuthScreen`.

The authentication flow is complete! To test it out, you can create a new user, log in with it and start chatting with other users.

------------

TODO: maybe use HttpOnly in cookie
save user’s id or data in localStorage so we can guess if a user is logged in or not and later on invalidate it
we send a password in its raw form over the wire
Error message: “req.password and req.passwordConfirm don't match” looks odd, should be “Passwords don’t match”
Why is `const maySignUp = useCallback(() => {` a useCallback and not useMemo for example? We do use a value there.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](https://github.com/Urigo/WhatsApp-Clone-Tutorial/tree/master@0.1.0/.tortilla/manuals/views/step12.md) | [Next Step >](https://github.com/Urigo/WhatsApp-Clone-Tutorial/tree/master@0.1.0/.tortilla/manuals/views/step14.md) |
|:--------------------------------|--------------------------------:|

[}]: #
