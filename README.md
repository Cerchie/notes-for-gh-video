## Getting started with Kafka Streams: The Processing Process


Now, I'll pivot to building a Kafka Streams application that processes the 'github-pull_requests' events, creating an up-to-date ratio of open/closed pull requests.

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

Our store key will be named 'state_counter'. 

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

Then, I’ll initialize it with our state store. The init method schedules a punctuation to fire every second, printing the ratio in the store. 

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

Next, I’ll write the process method to  take in the events, mark them as open or closed, increment the count, and stash them in the state store.

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
SHOULD I ZOOM IN HERE I WONDER
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

 Next, I’m going to create a GitHubPRRatio class to hold this processing… process!. I’ll need some variables:

A logger for verifying that the data’s coming in, 
Variables working towards creating my custom serde,
And some strings – my storename, a no state variable, and some input and output topic names. 
The main method

Let’s wrap all this up in our main method. 

I’ll load the properties – and hold on, let’s make the method that loads them. This method points at the path to our properties file and returns the contents.

Let’s add a couple things to the properties down here– consumer group id, set the offset to earliest, and a max cache bytes config. 

I’ll call the  GitHubPrRatio method, to get the ratio coming in. 

Create a new Streams instance with our topology. What’s a topology? Hold on a sec, I’ll show you, but first, let’s tie this up.

Then we’ll set up a countdown latch so it waits for other threads before it starts… 

Then, in our try block, we’ll actually start the stream.

Our catch block will print the error and the stack trace. 


## The topology
Ok, so,  what’s a topology? It’s  a definition of the computational logic we need to process our data stream. What does it look like? Let’s make one. 

First, I’ll instantiate the builder, then I’ll create the stream. We specify the input topic in the Consumed.with method, as well as the deserializer we’ll use for the key and value.

 I’ll add a peek method so we can log what’s coming in, then chain on a valueMapper, which we’ll define in a second. 

Next, we’ll need to process the data. I’ll make that processor in a minute as well.

 I’ll chain on another peek method to log if our data is processed successfully, then we’ll send it to the output topic with the .to method.  

Now, I want to make the data coming in a more convenient shape, and I’ll use  ValueMapper for that. It’ll take the data coming in from the connector and give it a more convenient shape. I’ll use the GitHubPRInfo model for that.  

The 4 pieces of information we’re taking in as JsonNodes are the PR’s state, when it was created, and when it was closed, as well as the PR number. We then transform those to strings and ints, making sure to provide a case if the value happens to be null. 


## Wind the frog! 
Here’s the exciting bit! Let’s run the command which will start up our Kafka Streams instance… and we can see the data coming in! 


