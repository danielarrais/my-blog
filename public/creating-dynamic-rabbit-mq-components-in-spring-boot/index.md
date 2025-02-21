---
title: 'Creating Dynamic RabbitMQ Components in Spring Boot'
date: '2023-05-21'
spoiler: "Creating Dynamic RabbitMQ Components in Spring Boot"
---

# Creating Dynamic RabbitMQ Components in Spring Boot

Recently, I encountered the following challenge: listing queues used in a Spring Boot project within the `properties` file and creating them in RabbitMQ according to this list. Until then, I had been creating queues one by one by declaring a `@Bean` (**Example 1**), so that whenever the application starts, the necessary queues are created if they don't already exist. The problem with this approach is the need to modify code every time a new queue needs to be created.

**Example 1**
```java
@Bean  
public Queue example1Queue() {  
    log.info("example1Queue created");  
    return QueueBuilder.durable("example1Queue").build();  
}

@Bean  
public Queue example2Queue() {  
    log.info("example2Queue created");  
    return QueueBuilder.durable("example2Queue").build();  
}
```

Creating a `@Bean` that returns a `Queue` is the basic way to create queues with Spring, but if we need something more flexible and dynamic, we need another approach... we need the **`Declarables`** class.

## Declarables

The general idea behind this challenge was to gain flexibility and centralize queue information in a list within the project's properties file, without needing to modify Java code to configure a new queue. In this context, it would be useful to create multiple queues at once, and I discovered this is possible using the [`Declarables`](https://docs.spring.io/spring-amqp/api/org/springframework/amqp/core/Declarables.html) class. `Declarables` is a collection that supports `Declarable` implementations, such as `Bind`, `Queue`, and `Exchange` (and likely others). To use it, we follow the same logic as Example 1, declaring a `@Bean`, but instead of returning a specific component, we return a `Declarables` instance with everything we need to create. In **Example 2**, I recreate Example 1 using `Declarables`.

**Example 2**
```java
@Bean  
public Declarables queues() {  
    return buildQueues();  
}

private Declarables buildQueues() {  
    return new Declarables(  
        QueueBuilder.durable("example1Queue")  
                    .build(),  
        QueueBuilder.durable("example2Queue")  
                    .build()  
    );
}
```

See how interesting and simple it is, and how it opens up possibilities, such as getting a list of queues from anywhere, iterating through them, and building the queues. That's what we're going to do! But be careful! I warn that we should use this responsibly. For example, in my company's project, I created many things with it - Queues, Binds, and Exchanges - but separately, to avoid confusion. Flexibility should be added carefully to prevent chaos and keep the project maintainable.

Now that we know how to use the `Declarables` class, let's store our queue list in the project's properties file.

## Creating the Queue List in the Properties File

Now we'll store the queues in the project's properties file (using YAML) so we can retrieve them in our Bean and create them. For this, I thought of the structure shown in **Example 3**.

**Example 3**
```yaml
queues:  
    example1Queue:  
        name: example1Queue  
        tll: 1000
    example2Queue:  
        name: example2Queue
        tll: 1000
```

I used a key-value structure, but we could also use a string list (**Example 4**) with queue names or a list of objects without keys (**Example 5**). I adopted the structure from **Example 3** to allow us to reference queues in other locations, such as in `@RabbitListener` annotations (example: `@RabbitListener(queues = "${queues.example1Queue.name}")`). The suggested structure (and that of **Example 5**) also allows adding other queue-relevant properties, such as **ttl**.

**Example 4**
```yaml
queues: example1Queue, example2Queue
```

**Example 5**
```yaml
queues: 
    - 
        name: example1Queue  
        tll: 1000
    -
        name: example2Queue
        tll: 1000
```

Now that we have our queues registered in the properties file, we need to map them to a class so we can retrieve the information where needed.

## Retrieving Queue Configurations

To retrieve the values from the queue list, we'll create the `QueuesProperties` class (**Example 6**). It contains a class representing the `Queue` and a `Map` of `Queue`. The `Queue` class has the attributes `name` and `tll`. For the **queues** property, I tried using a **list** but it didn't work, so I created a `getQueues()` method that returns a list.

**Example 6**
```java
@Data 
public class QueuesProperties {  
    private Map<String, Queue> queues;  
 
    public Collection<Queue> getQueues() {  
        if (CollectionUtils.isEmpty(queues)) return new ArrayList<>();  
        return queues.values();  
    }
    
    @Data   
    public static class Queue {   
        private String name;  
        private int tll;  
    } 
}
```

After building our POJO to store the queues, we need to tell Spring that this class represents properties, so we can use dependency injection. For this, we use the `@Component` and `@ConfigurationProperties("propertie")` annotations (**Example 7**). In `@ConfigurationProperties`, we would need to specify our property prefix, but since it's at the top of the hierarchy, we don't need to.

**Example 7**
```java
@Data 
@Component    
@ConfigurationProperties
public class QueuesProperties {  
    ...
}
```

We can also validate our annotations using Spring Validation and prevent the application from running if any information is missing. We know, for example, that the `name` attribute of `Queue` should be mandatory, so we can annotate it with `@NotBlank` and annotate the classes with `@Validated`, so Spring understands it should validate them (**Example 8**).

**Example 8**
```java
...
@Data  
@Validated  
public static class Queue {  
    @NotBlank  
    private String name;  
    private int tll;  
}
...
```

After all this, our complete class looks like this:

**Example 9** ([QueuesProperties.java](https://github.com/danielarrais/multiple-queues-rabbit/blob/main/src/main/java/dev/danielarrais/multiplequeuesrabbit/broker/setup/QueuesProperties.java))
```java
@Data  
@Validated  
@Component    
@ConfigurationProperties
public class QueuesProperties {  
  
    private Map<String, Queue> queues;  
      
    public Collection<Queue> getQueues() {  
        if (CollectionUtils.isEmpty(queues)) return new ArrayList<>();  
        return queues.values();  
    }  
      
    @Data  
    @Validated  
    public static class Queue {  
        @NotBlank  
        private String name;  
        private int tll;  
    }  
}
```

Now we can inject our properties class using Spring's dependency injection in any supported class.

## Creating Our Queues

To create our queues, we'll create a class annotated with `@Component` called `QueuesInitializer`, where we'll declare our queue creation `@Bean`. We'll inject our properties using a final variable (it's necessary to create a constructor so Spring can instantiate it; for this, I use Lombok's `@RequiredArgsConstructor` annotation).

Then we declare our `@Bean` that returns a `Declarables` with our built queues. As shown in Example 10:

**Example 10** ([QueuesInitializer.java](https://github.com/danielarrais/multiple-queues-rabbit/blob/main/src/main/java/dev/danielarrais/multiplequeuesrabbit/broker/setup/QueuesInitializer.java))
```java
@Component  
@RequiredArgsConstructor  
public class QueuesInitializer {  
  
    private final QueuesProperties queuesProperties;  
      
    @Bean  
    public Declarables queues() {  
        return buildQueues();  
    }  
      
    private Declarables buildQueues() {  
        List<Queue> queues = queuesProperties.getQueues().stream().map(queue -> {  
            return QueueBuilder.durable(queue.getName())  
                               .ttl(queue.getTll())  
                               .build();  
        }).collect(Collectors.toList());  
          
        return new Declarables(queues);  
    }  
  
}
```

That's it, simple and easy.

# Repository
You can find the code presented in this [repository](https://github.com/danielarrais/multiple-queues-rabbit).
