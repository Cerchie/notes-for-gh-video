## Getting started with Kafka Streams: The Processing Process


Now, I'll pivot to building a Kafka Streams application that processes the 'github-pull_requests' records, creating an up-to-date ratio of open/closed pull requests.

If you want to follow along with this part, check out the GitHub repo in the links below. I’m going to focus on the domain logic given our timeframe, so if you really want to dive in, I suggest consulting the codebase. 



```java
static class MyProcessorSupplier implements ProcessorSupplier<String, GitHubPRInfo, String, GitHubPRStateCounter> {
}
```

Now I'll  write the ProcessorSupplier, using the class provided by KafkaStreams. I'll use the record shape supplied by GitHubPRInfo as well as the object provided by GitHubPrStateCounter in our models folder.

---

```java
static class MyProcessorSupplier implements ProcessorSupplier<String, GitHubPRInfo, String, GitHubPRStateCounter> { 
	static final String STORE_KEY = "state-counter";
}
```

Now, in Kafka Streams, any processing that involves state will store that state in a, well, a "store", which is held by RocksDB. The key by which we'll retrieve this state later will be named 'state_counter', because we'll be counting the open and closed state of pull requests to get the current ratio. 

---

```java
static class MyProcessorSupplier implements ProcessorSupplier<String, GitHubPRInfo, String, GitHubPRStateCounter> { 
	static final String STORE_KEY = "state-counter";
   @Override
   public Processor<String, GitHubPRInfo, String, GitHubPRStateCounter> get() {
		return new Processor<>() {
	}
}
```

Now, let's create a new processor instance to return. 

```java
static class MyProcessorSupplier implements ProcessorSupplier<String, GitHubPRInfo, String, GitHubPRStateCounter> { 
	static final String STORE_KEY = "state-counter";
   @Override
   public Processor<String, GitHubPRInfo, String, GitHubPRStateCounter> get() {
		return new Processor<>() {

    private KeyValueStore<String, GitHubPRStateCounter> kvStore;
                @Override
                public void init(final ProcessorContext<String, GitHubPRStateCounter> context) {
                    this.kvStore = context.getStateStore(STORE_NAME);

                    context.schedule(Duration.ofSeconds(1), PunctuationType.STREAM_TIME, timestamp -> {
                        GitHubPRStateCounter entry = kvStore.get(STORE_KEY);
                        System.out.printf("Store value %s%n", entry);
                        context.forward(new Record<>("pr", entry, timestamp));
                    });
	}
}

```

Then, I’ll initialize it with our state store. The init method schedules a punctuation, or a scheduled periodic action on the stream, to fire every second, printing the ratio in the store. You might also wonder about the `ProcessorContext`. This context instance includes metadata of the currently processed record and, in addition, allows us to schedule the punctuation. 

```java
static class MyProcessorSupplier implements ProcessorSupplier<String, GitHubPRInfo, String, GitHubPRStateCounter> { 
	static final String STORE_KEY = "state-counter";
   @Override
   public Processor<String, GitHubPRInfo, String, GitHubPRStateCounter> get() {
		return new Processor<>() {

    private KeyValueStore<String, GitHubPRStateCounter> kvStore;
                @Override
                public void init(final ProcessorContext<String, GitHubPRStateCounter> context) {
                    this.kvStore = context.getStateStore(STORE_NAME);

                    context.schedule(Duration.ofSeconds(1), PunctuationType.STREAM_TIME, timestamp -> {
                        GitHubPRStateCounter entry = kvStore.get(STORE_KEY);
                        System.out.printf("Store value %s%n", entry);
                        context.forward(new Record<>("pr", entry, timestamp));
                    });
	}
              @Override
                public void process(final Record<String, GitHubPRInfo> record) {

                }
            };
        }
}
```

Next, I’ll write the process method to take in the records, mark them as open or closed, increment the count, and stash them in the state store.

```java
static class MyProcessorSupplier implements ProcessorSupplier<String, GitHubPRInfo, String, GitHubPRStateCounter> { 
	static final String STORE_KEY = "state-counter";
   @Override
   public Processor<String, GitHubPRInfo, String, GitHubPRStateCounter> get() {
		return new Processor<>() {

    private KeyValueStore<String, GitHubPRStateCounter> kvStore;
                @Override
                public void init(final ProcessorContext<String, GitHubPRStateCounter> context) {
                    this.kvStore = context.getStateStore(STORE_NAME);

                    context.schedule(Duration.ofSeconds(1), PunctuationType.STREAM_TIME, timestamp -> {
                        GitHubPRStateCounter entry = kvStore.get(STORE_KEY);
                        System.out.printf("Store value %s%n", entry);
                        context.forward(new Record<>("pr", entry, timestamp));
                    });
	}
              @Override
                public void process(final Record<String, GitHubPRInfo> record) {
                    GitHubPRInfo prInfo = record.value();

                }
            };
        }
}
```
We’ll get the value of the record and start a chain of logic. 

```java
#note: zoom in here, but leave `prInfo = record.value()` line on the screen
                    if(!prInfo.state().equals(NO_STATE)) {
                        GitHubPRStateCounter stateCounter = kvStore.get(STORE_KEY);
                        if (stateCounter == null) {
                            stateCounter = new GitHubPRStateCounter();
                        }
                        if (prInfo.state().equalsIgnoreCase("open")) {
                            stateCounter.setOpen(stateCounter.getOpen() + 1);
                        } else {
                            stateCounter.setClosed(stateCounter.getClosed() + 1);
                        }
                        kvStore.put(STORE_KEY, stateCounter);
                    }
```

If there’s no state and the state counter is null, we instantiate a new state counter. 

```java
                    if(!prInfo.state().equals(NO_STATE)) {
                        GitHubPRStateCounter stateCounter = kvStore.get(STORE_KEY);
                        if (stateCounter == null) {
                            stateCounter = new GitHubPRStateCounter();
                        }
                        if (prInfo.state().equalsIgnoreCase("open")) {
                            stateCounter.setOpen(stateCounter.getOpen() + 1);
                        } else {
                            stateCounter.setClosed(stateCounter.getClosed() + 1);
                        }
                    }
```

If the pr state is either open or closed, we increment the open or closed counts. 

```java
                    if(!prInfo.state().equals(NO_STATE)) {
                        GitHubPRStateCounter stateCounter = kvStore.get(STORE_KEY);
                        if (stateCounter == null) {
                            stateCounter = new GitHubPRStateCounter();
                        }
                        if (prInfo.state().equalsIgnoreCase("open")) {
                            stateCounter.setOpen(stateCounter.getOpen() + 1);
                        } else {
                            stateCounter.setClosed(stateCounter.getClosed() + 1);
                        }
                        kvStore.put(STORE_KEY, stateCounter);
                    }
```

Then, we update the store.

```java

    static class MyProcessorSupplier implements ProcessorSupplier<String, GitHubPRInfo, String, GitHubPRStateCounter> {

        static final String STORE_KEY = "state-counter";

        @Override
        public Processor<String, GitHubPRInfo, String, GitHubPRStateCounter> get() {
            return new Processor<>() {
                private KeyValueStore<String, GitHubPRStateCounter> kvStore;

                @Override
                public void init(final ProcessorContext<String, GitHubPRStateCounter> context) {
                    this.kvStore = context.getStateStore(STORE_NAME);

                    context.schedule(Duration.ofSeconds(1), PunctuationType.STREAM_TIME, timestamp -> {
                        GitHubPRStateCounter entry = kvStore.get(STORE_KEY);
                        System.out.printf("Store value %s%n", entry);
                        context.forward(new Record<>("pr", entry, timestamp));
                    });
                }

                @Override
                public void process(final Record<String, GitHubPRInfo> record) {
                    GitHubPRInfo prInfo = record.value();
                    if(!prInfo.state().equals(NO_STATE)) {
                        GitHubPRStateCounter stateCounter = kvStore.get(STORE_KEY);
                        if (stateCounter == null) {
                            stateCounter = new GitHubPRStateCounter();
                        }
                        if (prInfo.state().equalsIgnoreCase("open")) {
                            stateCounter.setOpen(stateCounter.getOpen() + 1);
                        } else {
                            stateCounter.setClosed(stateCounter.getClosed() + 1);
                        }
                        kvStore.put(STORE_KEY, stateCounter);
                    }
                }
            };
        }

        @Override
        public Set<StoreBuilder<?>> stores() {
            return Collections.singleton(Stores.keyValueStoreBuilder(Stores.inMemoryKeyValueStore(STORE_NAME), Serdes.String(), prStateCounterSerde));
        }
    }

}
```

Lastly, we use the processor API to create our state store. We return a singleton with the store values built from our store builder. 


## The GitHub PR Ratio class

```java
public class GitHubPrRatio {
}
```

 Next, I’m going to create a GitHubPRRatio class to hold this processing… process!. I’ll need some variables:


```java
public class GitHubPrRatio {
    private static final Logger LOG = LoggerFactory.getLogger(GitHubPrRatio.class);

}
```
A logger for verifying that the data’s coming in, 
```java
public class GitHubPrRatio {
    // start with public class, then fill in variables in screen recording
    private static final Logger LOG = LoggerFactory.getLogger(GitHubPrRatio.class);
    static final Serde<GitHubPRStateCounter> prStateCounterSerde = StreamsSerde.serdeFor(GitHubPRStateCounter.class);
    final Serializer<JsonNode> jsonSerializer = new JsonSerializer();
    final Deserializer<JsonNode> jsonDeserializer = new JsonDeserializer();
    final Serde<JsonNode> jsonNodeSerde = Serdes.serdeFrom(jsonSerializer, jsonDeserializer);
}
```
Variables working towards creating my custom serde,
```java
public class GitHubPrRatio {
    // start with public class, then fill in variables in screen recording
    private static final Logger LOG = LoggerFactory.getLogger(GitHubPrRatio.class);
    static final Serde<GitHubPRStateCounter> prStateCounterSerde = StreamsSerde.serdeFor(GitHubPRStateCounter.class);
    final Serializer<JsonNode> jsonSerializer = new JsonSerializer();
    final Deserializer<JsonNode> jsonDeserializer = new JsonDeserializer();
    final Serde<JsonNode> jsonNodeSerde = Serdes.serdeFrom(jsonSerializer, jsonDeserializer);
    static final String STORE_NAME = "pr-state";
    static final String NO_STATE = "no-state";
    static final String INPUT_TOPIC = "github-pull_requests";
    static final String OUTPUT_TOPIC = "github-pull_requests"

}
```
And some strings – my storename, a no state variable, and some input and output topic names. 

## The main method

```java

    public static void main(final String[] args) throws Exception {

        final Properties props = GitHubPrRatio.loadEnvProperties("get-started.properties");
}
```

Let’s wrap all this up in our main method. 

```java
    public static Properties loadEnvProperties(String fileName) throws IOException {
        final Properties allProps = new Properties();
        try (final FileInputStream input = new FileInputStream(fileName)) {
            allProps.load(input);
        }
        return allProps;
    }
```

I’ll load the properties – and hold on, let’s make the method that loads them. This method points at the path to our properties file and returns the contents.


```java

    public static void main(final String[] args) throws Exception {

        final Properties props = GitHubPrRatio.loadEnvProperties("get-started.properties");

        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-github-87");
        props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 0);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

}

Let’s add a couple things to the properties down here– consumer group id, set the offset to earliest, and a max cache bytes config.


```java

    public static void main(final String[] args) throws Exception {

        final Properties props = GitHubPrRatio.loadEnvProperties("get-started.properties");

        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-github-87");
        props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 0);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        GitHubPrRatio gitHubPrRatio = new GitHubPrRatio();


}

I’ll call the  GitHubPrRatio method, to get the ratio coming in.


```java

    public static void main(final String[] args) throws Exception {

        final Properties props = GitHubPrRatio.loadEnvProperties("get-started.properties");

        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-github-87");
        props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 0);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        GitHubPrRatio gitHubPrRatio = new GitHubPrRatio();

        final KafkaStreams streams = new KafkaStreams(gitHubPrRatio.topology(props), props);

}

Create a new Streams instance with our topology. What’s a topology? Hold on a sec, I’ll show you, but first, let’s tie this up.


```java

    public static void main(final String[] args) throws Exception {

        final Properties props = GitHubPrRatio.loadEnvProperties("get-started.properties");

        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-github-87");
        props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 0);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        GitHubPrRatio gitHubPrRatio = new GitHubPrRatio();

        final KafkaStreams streams = new KafkaStreams(gitHubPrRatio.topology(props), props);

        final CountDownLatch latch = new CountDownLatch(1);
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            streams.close(Duration.ofSeconds(5));
            latch.countDown();
        }));
}

Then we’ll set up a countdown latch so it waits for other threads before it starts…


```java

    public static void main(final String[] args) throws Exception {

        final Properties props = GitHubPrRatio.loadEnvProperties("get-started.properties");

        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-github-87");
        props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 0);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        GitHubPrRatio gitHubPrRatio = new GitHubPrRatio();

        final KafkaStreams streams = new KafkaStreams(gitHubPrRatio.topology(props), props);

        final CountDownLatch latch = new CountDownLatch(1);
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            streams.close(Duration.ofSeconds(5));
            latch.countDown();
        }));

        try {
            streams.start();
            latch.await();
        } 
    }
}

Then, in our try block, we’ll actually start the stream.


```java

    public static void main(final String[] args) throws Exception {

        final Properties props = GitHubPrRatio.loadEnvProperties("get-started.properties");

        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "streams-github-87");
        props.put(StreamsConfig.STATESTORE_CACHE_MAX_BYTES_CONFIG, 0);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        GitHubPrRatio gitHubPrRatio = new GitHubPrRatio();

        final KafkaStreams streams = new KafkaStreams(gitHubPrRatio.topology(props), props);

        final CountDownLatch latch = new CountDownLatch(1);
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            streams.close(Duration.ofSeconds(5));
            latch.countDown();
        }));

        try {
            streams.start();
            latch.await();
        } catch (final Throwable e) {
            e.printStackTrace();
            System.exit(1);
        }
        System.exit(0);
    }
}

Our catch block will print the error and the stack trace. 


## The topology

```java
    public Topology topology(Properties properties) {

}
```

Ok, so,  what’s a topology? It’s  a definition of the computational logic we need to process our data stream. What does it look like? Let’s make one.


```java
    public Topology topology(Properties properties) {

        final StreamsBuilder builder = new StreamsBuilder();

        builder.stream(INPUT_TOPIC, Consumed.with(Serdes.String(), jsonNodeSerde))
    }


}
```

First, I’ll instantiate the builder, then I’ll create the stream. We specify the input topic in the Consumed.with method, as well as the deserializer we’ll use for the key and value.


```java
    public Topology topology(Properties properties) {

        final StreamsBuilder builder = new StreamsBuilder();

        builder.stream(INPUT_TOPIC, Consumed.with(Serdes.String(), jsonNodeSerde))
                .peek((key, value) -> LOG.info("Incoming key[{}] value[{}]", key, value))
                .mapValues(valueMapper)
    }


}
```

 I’ll add a peek method so we can log what’s coming in, then chain on a valueMapper, which we’ll define in a second. 

 
```java
    public Topology topology(Properties properties) {

        final StreamsBuilder builder = new StreamsBuilder();

        builder.stream(INPUT_TOPIC, Consumed.with(Serdes.String(), jsonNodeSerde))
                .peek((key, value) -> LOG.info("Incoming key[{}] value[{}]", key, value))
                .mapValues(valueMapper)
                .process(new MyProcessorSupplier())
                .peek((key, value) -> LOG.info("Outgoing value key[{}] value[{}]", key, value))
                .to(OUTPUT_TOPIC, Produced.with(Serdes.String(), prStateCounterSerde));

        return builder.build(properties);
    }


}
```

Next, we’ll need to process the data, using the processer we have already defined. 

```java
    public Topology topology(Properties properties) {

        final StreamsBuilder builder = new StreamsBuilder();

        builder.stream(INPUT_TOPIC, Consumed.with(Serdes.String(), jsonNodeSerde))
                .peek((key, value) -> LOG.info("Incoming key[{}] value[{}]", key, value))
                .mapValues(valueMapper)
                .process(new MyProcessorSupplier())
                .peek((key, value) -> LOG.info("Outgoing value key[{}] value[{}]", key, value))
                .to(OUTPUT_TOPIC, Produced.with(Serdes.String(), prStateCounterSerde));

        return builder.build(properties);
    }


}
```

 I’ll chain on another peek method to log if our data is processed successfully, then we’ll send it to the output topic with the .to method.  

 
```java
    public Topology topology(Properties properties) {

        final ValueMapper<JsonNode, GitHubPRInfo> valueMapper = jsonNode -> {

            return new GitHubPRInfo(state, createdAt, closedAt, prNumber);
        };

        final StreamsBuilder builder = new StreamsBuilder();

        builder.stream(INPUT_TOPIC, Consumed.with(Serdes.String(), jsonNodeSerde))
                .peek((key, value) -> LOG.info("Incoming key[{}] value[{}]", key, value))
                .mapValues(valueMapper)
                .process(new MyProcessorSupplier())
                .peek((key, value) -> LOG.info("Outgoing value key[{}] value[{}]", key, value))
                .to(OUTPUT_TOPIC, Produced.with(Serdes.String(), prStateCounterSerde));

        return builder.build(properties);
    }


}
```

Now, I want to make the data coming in a more convenient shape, and I’ll use a ValueMapper. It’ll take the data coming in from the connector and give it a more convenient shape. I’ll use the GitHubPRInfo model for doing that.  


```java
    public Topology topology(Properties properties) {

        final ValueMapper<JsonNode, GitHubPRInfo> valueMapper = jsonNode -> {
            JsonNode prStateVal = jsonNode.findValue("state");
            JsonNode prCreatedAtVal = jsonNode.findValue("created_at");
            JsonNode prClosedAtVal = jsonNode.findValue("closed_at");
            JsonNode prNumberVal = jsonNode.findValue("number");
            String state = prStateVal != null ? prStateVal.asText() : NO_STATE;
            String createdAt = prCreatedAtVal != null ? prCreatedAtVal.asText() : "no-created-at";
            String closedAt = prClosedAtVal != null ? prClosedAtVal.asText() : "no-closed-at";
            int prNumber = prNumberVal != null ? prNumberVal.asInt() : Integer.MIN_VALUE;

            return new GitHubPRInfo(state, createdAt, closedAt, prNumber);
        };

        final StreamsBuilder builder = new StreamsBuilder();

        builder.stream(INPUT_TOPIC, Consumed.with(Serdes.String(), jsonNodeSerde))
                .peek((key, value) -> LOG.info("Incoming key[{}] value[{}]", key, value))
                .mapValues(valueMapper)
                .process(new MyProcessorSupplier())
                .peek((key, value) -> LOG.info("Outgoing value key[{}] value[{}]", key, value))
                .to(OUTPUT_TOPIC, Produced.with(Serdes.String(), prStateCounterSerde));

        return builder.build(properties);
    }


}
```

The 4 pieces of information we’re taking in as JsonNodes are the PR’s state, when it was created, and when it was closed, as well as the PR number. We then transform those to strings and ints, making sure to provide a case if the value happens to be null. 


## Wind the frog! 

Here’s the exciting bit! Let’s run the command which will start up our Kafka Streams instance… and we can see the data coming in! 

(terminal recording, something like)

```bash
Store value GitHubPRStateCounter{open=138, closed=199}
Store value GitHubPRStateCounter{open=139, closed=199}
Store value GitHubPRStateCounter{open=140, closed=199}
Store value GitHubPRStateCounter{open=141, closed=199}
```
