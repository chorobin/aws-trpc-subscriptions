# Work In Progress

This is still work in progress and nothing is concrete yet.

# Motivation

Subscriptions and real time data is a common requirement for apps. This repository is designed to provide the following solutions:

- Provide adapters for Amazon Websockets API Gateway to easily create $connect, $disconnect and a main handler in a type-safe manner
- Provide logic to persist subscriptions in Amazon DynamoDB using single table design
- To publish to a subscription in backend processes with type safety
- To filter subscriptions and only publish to a subscription based on ctx or inputs

# Watch the video

https://github.com/chorobin/aws-trpc-subscriptions

[![Watch the video]](https://github.com/chorobin/aws-trpc-subscriptions)

# Or read the an example if you prefer!

Initialise subscriptions and define your tRPC router

```typescript
export const subscriptions = initSubscriptions();

interface AppContext {
  readonly userId: string;
}

const t = initTRPC.context<AppContext>().create();

export const appRouter = t.router({
  mySubscription: t.procedure
    .input(
      z.object({
        name: z.string(),
      })
    )
    .subscription(subscriptions.resolver<string>()),
});
```

Define filters based on the routes

```typescript
export const appSubscriptions = subscriptions
  .router({ router: appRouter })
  .store({
    store: dynamodb({
      tableName: "your table name goes here",
      dynamoDBClient: new DynamoDBClient({}),
    }),
  })
  .routes.mySubscription.filter({
    name: "userIdAndName",
    ctx: {
      userId: true,
    },
    input: {
      name: true,
    },
  });
```

Create the adaptors ($connect, handler, $disconnect)

```typescript
export const main = appSubscriptions.connect();
```

```typescript
export const main = appSubscriptions.disconnect();
```

```typescript
export const main = appSubscriptions.handler();
```

Create the publisher

```typescript
export const publisher = appSubscriptions.publisher({
  endpoint: "your websocket api endpoint goes here",
});
```

Publish to the subscription in your backend processes (lambda etc).

```typescript
await publisher.routes.mySubscription.publish({
  data: "hi",
  filter: {
    name: "userIdAndName",
    input: {
      name: "name1",
    },
    ctx: {
      userId: "user1",
    },
  },
});
```

Subscribe on the client like any other tRPC subscription

```typescript
api.mySubscription.useSubscription(
  {
    name: "hello",
  },
  {
    onData: (data) => {
      // handle on data
    },
  }
);
```

# Usage with SST

We recommend SST to deploy serverless applications to AWS. It provdes a WebSocketApi construct to deploy to Api Gateway

First use the Table construct. A dynamodb table is required to persist connections and subscriptions.

```typescript
const table = new Table(stack, "Subscriptions", {
  primaryIndex: {
    partitionKey: "pk",
    sortKey: "sk",
  },
  fields: {
    pk: "string",
    sk: "string",
  },
  cdk: {
    table: {
      removalPolicy: RemovalPolicy.DESTROY,
    },
  },
  timeToLiveAttribute: "expireAt",
});
```

Fields pk and sk are required fields to be the partition key and sort key respectively. A expireAt field is used to delete connection and subscriptions which are older than 4 hours

Then define your web socket api construct

```typescript
const websocket = new WebSocketApi(stack, "WebsocketApi", {
  defaults: {
    function: {
      bind: [table],
    },
  },
  routes: {
    $connect: "./packages/functions/src/api/websocket/connect.main",
    $default: "./packages/functions/src/api/websocket/handler.main",
    $disconnect: "./packages/functions/src/api/websocket/disconnect.main",
  },
});
```

You should bind the subscription table to the web socket api so it can be used to connect, disconnect and handle subscriptions. $connect, $diconnect, $default reference lambdas which are created from the adaptors

Now you just need to use the SST's `sst/node` to connect the adaptors to your infrastructure

```typescript
//packages/functions/src/api/websocket/connect
export const main = appSubscriptions.connect();
```

```typescript
//packages/functions/src/api/websocket/disconnect
export const main = appSubscriptions.disconnect();
```

```typescript
//packages/functions/src/api/websocket/handler
export const main = appSubscriptions.handler();
```

To publish to a subscription in a lambda function you need to first use the function construct and bind it to the websocket and table. For example you could publish a message to the subscription in the consumer of an event bus

```typescript
const eventBus = new EventBus(stack, "EventBus");

eventBus.subscribe("myEvent", {
  handler: "./packages/functions/src/events/myEvent.main",
  bind: [websocket, table],
});
```

You can then wire up the publisher to the web socket api and table

```typescript
export const publisher = appSubscriptions.publisher({
  endpoint: WebSocketApi.WebsocketApi.httpsUrl,
});
```

And then you can publish the message in the event bus consumer

```typescript
export const main = EventHandler(Events.MyEvent, async (event) => {
  await publisher.routes.mySubscription.publish({
    data: "hi",
    filter: {
      name: "userIdAndName",
      input: {
        name: "Bob",
      },
      ctx: {
        userId: "user1",
      },
    },
  });
});
```

And of course you can use environment variables in NextJS to connect to the web socket api so the client can subscribe

```typescript
new NextjsSite(context.stack, "Web", {
  path: "./packages/web",
  environment: {
    NEXT_PUBLIC_HTTP_URL: http.url,
    NEXT_PUBLIC_WS_URL: websocket.url,
  },
});
```

And make your own provider to wire it up to the frontend. Notice we can support http or ws depending on if it is a subscription or not

```typescript
const wsClient = createWSClient({
  url: process.env.NEXT_PUBLIC_WS_URL ?? "",
});

export const Providers: React.FunctionComponent<ProviderProps> = ({
  children,
}) => {
  const [queryClient] = React.useState(() => new QueryClient());
  const [trpcClient] = React.useState(() =>
    api.createClient({
      links: [
        splitLink({
          condition: (op) => op.type === "subscription",
          true: wsLink({
            client: wsClient,
          }),
          false: httpBatchLink({
            url: `${process.env.NEXT_PUBLIC_HTTP_URL}/api` ?? "",
          }),
        }),
      ],
    })
  );
  return (
    <api.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    </api.Provider>
  );
};
```

Lets recap. This approach allows us to continue using AWS infrastructure and serverless. We can publish notifications in any lambda we deploy to AWS while keeping type safety. We can filter subscriptions based on defined filters on anything in input or ctx of the trpc subscription

# Deep Dive

## initSubscriptions

`initSubscriptions` initialiazes the subscriptions instance. The subscriptions instance allows you to create subscription resolvers and to attach a router to the subscriptions instance.

## subcriptions.resolver

`resolver` creates a function which returns a tRPC observable. There are some limitations to observables in serverless. We cannot create inifinite observables and they have to finish at some point. This is because serverless must also be stateless. Most of the time you probably just want to send a message to a subscriber, which `resolver` is perfect for. `resolver` is required as it wraps a dummy observable with hooks so parts of the observable can be executed depending on different events (started, stopped, data etc)

## subscriptions.router

`router` is required to let the subscriptions instance now about your router. This is important so we can infer your procedures to allow configuration of filtering. The typescript behind this works very similar to a tRPC client (using mapped types and proxies at runtime)

## subscriptions.routes.[procedure].filter

Defines what fields from input and ctx you can filter on when publishing to the subscription. A filter must have a unique name. `ctx` and `input` are combined into a composite sort key when a subscription is started.

## subscriptions.connect

Creates the connect lambda function to handle websocket connections. Connections are put into dynamodb

## subscriptions.handler

Creates the main lambda function for handling subscriptions starting and stopping. Subscriptions are put into dynamodb - records are created for configured filters with a composite sort key (e.g name#nameOfFilter#ctx#userId#valueOfUserId#input#name#valueOfName)

## subscriptions.disconnect

Creates the delete lambda functions for handling of deletion of connections. It deletes the connection record along with the subscriptions. `expireAt` is still necessary because this is a best effort

## subscriptions.publisher

Creates the publisher that expects the url of the WebSocketApi. This can then be used in your backend services to publish messages to a particular subscription.

## subscriptions.routes.[procedure].publish

Publishes data to a subscription. A configured filter can be used to only notify subcribers which are listening on those values. This works by the `begins_with` operator in dynamodb as the composite sort key for a record is constructed from the filter values passed.

# Limitations

## Broadcasting

While it is possible to broadcast to a limited number of connections with this approach, its generally recommended to restrict your subscriptions as much as possible using filters. For some use cases this is fine as it may be you just want a single user to know if a backend process has finished their action.

To support broadcasting effeciently an AWS IoT Pub Sub adapter should be developed but this would be more involved and might come with different trade-offs. For example, I don't know a way to use AWS IoT Pub Sub with a pure WebSocket client. In which case a client side adapter is also necessary. This is possible future work and is worth investigating.
