# gRPC Spring Boot Services with Gradle - Server and Client with Event Streaming

This project consists of two Spring Boot services using Gradle:
1. **gRPC Server** - Receives and broadcasts events to subscribers
2. **gRPC Client** - Subscribes to events and sends scheduled events

## Project Structure

```
grpc-services/
├── grpc-server/
│   ├── src/main/java/com/example/grpcserver/
│   │   ├── GrpcServerApplication.java
│   │   ├── service/EventService.java
│   │   └── config/GrpcServerConfig.java
│   ├── src/main/proto/
│   │   └── event.proto
│   ├── src/main/resources/
│   │   └── application.yml
│   ├── build.gradle
│   └── settings.gradle
└── grpc-client/
    ├── src/main/java/com/example/grpcclient/
    │   ├── GrpcClientApplication.java
    │   ├── service/EventClientService.java
    │   ├── scheduler/EventScheduler.java
    │   └── config/GrpcClientConfig.java
    ├── src/main/proto/
    │   └── event.proto
    ├── src/main/resources/
    │   └── application.yml
    ├── build.gradle
    └── settings.gradle
```

## 1. Protocol Buffer Definition

### `src/main/proto/event.proto` (Same for both services)

```protobuf
syntax = "proto3";

package com.example.grpc;

option java_multiple_files = true;
option java_package = "com.example.grpc";
option java_outer_classname = "EventProto";

// Event message
message Event {
  string id = 1;
  string type = 2;
  string payload = 3;
  int64 timestamp = 4;
  string source = 5;
}

// Request for subscribing to events
message EventSubscriptionRequest {
  string subscriber_id = 1;
  repeated string event_types = 2;
}

// Request for sending an event
message SendEventRequest {
  Event event = 1;
}

// Response for sending an event
message SendEventResponse {
  bool success = 1;
  string message = 2;
}

// Event service definition
service EventService {
  // Subscribe to events (server streaming)
  rpc SubscribeToEvents(EventSubscriptionRequest) returns (stream Event);
  
  // Send an event
  rpc SendEvent(SendEventRequest) returns (SendEventResponse);
}
```

## 2. gRPC Server Implementation

### `grpc-server/settings.gradle`

```gradle
rootProject.name = 'grpc-server'
```

### `grpc-server/build.gradle`

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'com.google.protobuf' version '0.9.4'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = '17'
}

repositories {
    mavenCentral()
}

ext {
    grpcVersion = '1.58.0'
    grpcSpringBootStarterVersion = '2.15.0.RELEASE'
    protobufVersion = '3.24.4'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    // gRPC dependencies
    implementation "net.devh:grpc-spring-boot-starter:${grpcSpringBootStarterVersion}"
    implementation "io.grpc:grpc-protobuf:${grpcVersion}"
    implementation "io.grpc:grpc-stub:${grpcVersion}"
    implementation "com.google.protobuf:protobuf-java:${protobufVersion}"
    
    // Additional dependencies
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:${protobufVersion}"
    }
    plugins {
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:${grpcVersion}"
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}

sourceSets {
    main {
        java {
            srcDirs 'build/generated/source/proto/main/grpc'
            srcDirs 'build/generated/source/proto/main/java'
        }
    }
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### `grpc-server/src/main/java/com/example/grpcserver/GrpcServerApplication.java`

```java
package com.example.grpcserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class GrpcServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(GrpcServerApplication.class, args);
    }
}
```

### `grpc-server/src/main/java/com/example/grpcserver/service/EventService.java`

```java
package com.example.grpcserver.service;

import com.example.grpc.Event;
import com.example.grpc.EventServiceGrpc;
import com.example.grpc.EventSubscriptionRequest;
import com.example.grpc.SendEventRequest;
import com.example.grpc.SendEventResponse;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;

@GrpcService
@Service
public class EventService extends EventServiceGrpc.EventServiceImplBase {

    private static final Logger logger = LoggerFactory.getLogger(EventService.class);
    
    // Thread-safe list to store active subscribers
    private final List<SubscriberInfo> subscribers = new CopyOnWriteArrayList<>();
    
    // Executor for handling async operations
    private final ScheduledExecutorService executor = Executors.newScheduledThreadPool(10);

    @Override
    public void subscribeToEvents(EventSubscriptionRequest request, 
                                  StreamObserver<Event> responseObserver) {
        
        String subscriberId = request.getSubscriberId();
        List<String> eventTypes = request.getEventTypesList();
        
        logger.info("New subscriber: {} subscribing to events: {}", subscriberId, eventTypes);
        
        // Create subscriber info
        SubscriberInfo subscriber = new SubscriberInfo(subscriberId, eventTypes, responseObserver);
        subscribers.add(subscriber);
        
        logger.info("Total active subscribers: {}", subscribers.size());
        
        // Handle client disconnection
        responseObserver.onError(new RuntimeException() {
            @Override
            public String getMessage() {
                subscribers.remove(subscriber);
                logger.info("Subscriber {} disconnected. Active subscribers: {}", 
                           subscriberId, subscribers.size());
                return "Client disconnected";
            }
        });
    }

    @Override
    public void sendEvent(SendEventRequest request, StreamObserver<SendEventResponse> responseObserver) {
        Event event = request.getEvent();
        
        logger.info("Received event: {} of type: {} from source: {}", 
                   event.getId(), event.getType(), event.getSource());
        
        // Broadcast event to all interested subscribers
        broadcastEventToSubscribers(event);
        
        // Send response
        SendEventResponse response = SendEventResponse.newBuilder()
                .setSuccess(true)
                .setMessage("Event successfully broadcast to " + getInterestedSubscribersCount(event) + " subscribers")
                .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
    
    private void broadcastEventToSubscribers(Event event) {
        subscribers.parallelStream()
                .filter(subscriber -> subscriber.isInterestedInEvent(event.getType()))
                .forEach(subscriber -> {
                    try {
                        subscriber.getResponseObserver().onNext(event);
                        logger.debug("Event {} sent to subscriber {}", event.getId(), subscriber.getSubscriberId());
                    } catch (Exception e) {
                        logger.error("Error sending event to subscriber {}: {}", 
                                   subscriber.getSubscriberId(), e.getMessage());
                        subscribers.remove(subscriber);
                    }
                });
    }
    
    private long getInterestedSubscribersCount(Event event) {
        return subscribers.stream()
                .filter(subscriber -> subscriber.isInterestedInEvent(event.getType()))
                .count();
    }

    // Inner class to hold subscriber information
    private static class SubscriberInfo {
        private final String subscriberId;
        private final List<String> eventTypes;
        private final StreamObserver<Event> responseObserver;

        public SubscriberInfo(String subscriberId, List<String> eventTypes, StreamObserver<Event> responseObserver) {
            this.subscriberId = subscriberId;
            this.eventTypes = eventTypes;
            this.responseObserver = responseObserver;
        }

        public boolean isInterestedInEvent(String eventType) {
            return eventTypes.isEmpty() || eventTypes.contains(eventType);
        }

        public String getSubscriberId() {
            return subscriberId;
        }

        public StreamObserver<Event> getResponseObserver() {
            return responseObserver;
        }
    }
}
```

### `grpc-server/src/main/java/com/example/grpcserver/config/GrpcServerConfig.java`

```java
package com.example.grpcserver.config;

import org.springframework.context.annotation.Configuration;

@Configuration
public class GrpcServerConfig {
    // Additional gRPC server configuration can be added here if needed
}
```

### `grpc-server/src/main/resources/application.yml`

```yaml
server:
  port: 8080

grpc:
  server:
    port: 9090
    
spring:
  application:
    name: grpc-server

logging:
  level:
    com.example.grpcserver: DEBUG
    net.devh.boot.grpc: DEBUG
```

## 3. gRPC Client Implementation

### `grpc-client/settings.gradle`

```gradle
rootProject.name = 'grpc-client'
```

### `grpc-client/build.gradle`

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'com.google.protobuf' version '0.9.4'
}

group = 'com.example'
version = '1.0.0'

java {
    sourceCompatibility = '17'
}

repositories {
    mavenCentral()
}

ext {
    grpcVersion = '1.58.0'
    grpcSpringBootStarterVersion = '2.15.0.RELEASE'
    protobufVersion = '3.24.4'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    // gRPC dependencies
    implementation "net.devh:grpc-client-spring-boot-starter:${grpcSpringBootStarterVersion}"
    implementation "io.grpc:grpc-protobuf:${grpcVersion}"
    implementation "io.grpc:grpc-stub:${grpcVersion}"
    implementation "com.google.protobuf:protobuf-java:${protobufVersion}"
    
    // Additional dependencies
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
    
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:${protobufVersion}"
    }
    plugins {
        grpc {
            artifact = "io.grpc:protoc-gen-grpc-java:${grpcVersion}"
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}

sourceSets {
    main {
        java {
            srcDirs 'build/generated/source/proto/main/grpc'
            srcDirs 'build/generated/source/proto/main/java'
        }
    }
}

tasks.named('test') {
    useJUnitPlatform()
}
```

### `grpc-client/src/main/java/com/example/grpcclient/GrpcClientApplication.java`

```java
package com.example.grpcclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class GrpcClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(GrpcClientApplication.class, args);
    }
}
```

### `grpc-client/src/main/java/com/example/grpcclient/service/EventClientService.java`

```java
package com.example.grpcclient.service;

import com.example.grpc.*;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.client.inject.GrpcClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.util.Arrays;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

@Service
public class EventClientService {

    private static final Logger logger = LoggerFactory.getLogger(EventClientService.class);

    @GrpcClient("grpc-server")
    private EventServiceGrpc.EventServiceStub asyncStub;

    @GrpcClient("grpc-server")
    private EventServiceGrpc.EventServiceBlockingStub blockingStub;

    @PostConstruct
    public void init() {
        // Start subscribing to events when the service starts
        subscribeToEvents();
    }

    public void subscribeToEvents() {
        logger.info("Starting event subscription...");
        
        EventSubscriptionRequest request = EventSubscriptionRequest.newBuilder()
                .setSubscriberId("client-service-001")
                .addAllEventTypes(Arrays.asList("USER_CREATED", "ORDER_PLACED", "PAYMENT_PROCESSED"))
                .build();

        StreamObserver<Event> responseObserver = new StreamObserver<Event>() {
            @Override
            public void onNext(Event event) {
                logger.info("Received event: {} of type: {} from source: {} with payload: {}", 
                           event.getId(), event.getType(), event.getSource(), event.getPayload());
                
                // Process the received event
                processReceivedEvent(event);
            }

            @Override
            public void onError(Throwable throwable) {
                logger.error("Error in event subscription: {}", throwable.getMessage());
                
                // Implement reconnection logic
                reconnectAfterDelay();
            }

            @Override
            public void onCompleted() {
                logger.info("Event subscription completed");
            }
        };

        asyncStub.subscribeToEvents(request, responseObserver);
    }

    public void sendEvent(String eventType, String payload) {
        Event event = Event.newBuilder()
                .setId(java.util.UUID.randomUUID().toString())
                .setType(eventType)
                .setPayload(payload)
                .setTimestamp(System.currentTimeMillis())
                .setSource("client-service")
                .build();

        SendEventRequest request = SendEventRequest.newBuilder()
                .setEvent(event)
                .build();

        try {
            SendEventResponse response = blockingStub
                    .withDeadlineAfter(5, TimeUnit.SECONDS)
                    .sendEvent(request);
            
            if (response.getSuccess()) {
                logger.info("Event sent successfully: {}", response.getMessage());
            } else {
                logger.error("Failed to send event: {}", response.getMessage());
            }
        } catch (Exception e) {
            logger.error("Error sending event: {}", e.getMessage());
        }
    }

    private void processReceivedEvent(Event event) {
        // Process the event based on its type
        switch (event.getType()) {
            case "USER_CREATED":
                handleUserCreatedEvent(event);
                break;
            case "ORDER_PLACED":
                handleOrderPlacedEvent(event);
                break;
            case "PAYMENT_PROCESSED":
                handlePaymentProcessedEvent(event);
                break;
            default:
                logger.info("Unknown event type: {}", event.getType());
        }
    }

    private void handleUserCreatedEvent(Event event) {
        logger.info("Processing USER_CREATED event: {}", event.getPayload());
        // Add your business logic here
    }

    private void handleOrderPlacedEvent(Event event) {
        logger.info("Processing ORDER_PLACED event: {}", event.getPayload());
        // Add your business logic here
    }

    private void handlePaymentProcessedEvent(Event event) {
        logger.info("Processing PAYMENT_PROCESSED event: {}", event.getPayload());
        // Add your business logic here
    }

    private void reconnectAfterDelay() {
        new Thread(() -> {
            try {
                logger.info("Attempting to reconnect in 5 seconds...");
                Thread.sleep(5000);
                subscribeToEvents();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                logger.error("Reconnection interrupted");
            }
        }).start();
    }
}
```

### `grpc-client/src/main/java/com/example/grpcclient/scheduler/EventScheduler.java`

```java
package com.example.grpcclient.scheduler;

import com.example.grpcclient.service.EventClientService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Random;

@Component
public class EventScheduler {

    private static final Logger logger = LoggerFactory.getLogger(EventScheduler.class);

    @Autowired
    private EventClientService eventClientService;

    private final ObjectMapper objectMapper = new ObjectMapper();
    private final Random random = new Random();
    private final List<String> eventTypes = Arrays.asList("USER_CREATED", "ORDER_PLACED", "PAYMENT_PROCESSED");

    // Send a scheduled event every 30 seconds
    @Scheduled(fixedRate = 30000)
    public void sendScheduledEvent() {
        try {
            String eventType = eventTypes.get(random.nextInt(eventTypes.size()));
            String payload = generateEventPayload(eventType);
            
            logger.info("Sending scheduled event of type: {}", eventType);
            eventClientService.sendEvent(eventType, payload);
            
        } catch (Exception e) {
            logger.error("Error sending scheduled event: {}", e.getMessage());
        }
    }

    // Send heartbeat event every 2 minutes
    @Scheduled(fixedRate = 120000)
    public void sendHeartbeat() {
        try {
            Map<String, Object> heartbeatData = new HashMap<>();
            heartbeatData.put("clientId", "client-service-001");
            heartbeatData.put("status", "ALIVE");
            heartbeatData.put("timestamp", System.currentTimeMillis());
            
            String payload = objectMapper.writeValueAsString(heartbeatData);
            
            logger.info("Sending heartbeat event");
            eventClientService.sendEvent("HEARTBEAT", payload);
            
        } catch (Exception e) {
            logger.error("Error sending heartbeat event: {}", e.getMessage());
        }
    }

    private String generateEventPayload(String eventType) throws Exception {
        Map<String, Object> payloadData = new HashMap<>();
        
        switch (eventType) {
            case "USER_CREATED":
                payloadData.put("userId", "user_" + random.nextInt(1000));
                payloadData.put("email", "user" + random.nextInt(1000) + "@example.com");
                payloadData.put("name", "User " + random.nextInt(1000));
                break;
                
            case "ORDER_PLACED":
                payloadData.put("orderId", "order_" + random.nextInt(10000));
                payloadData.put("userId", "user_" + random.nextInt(1000));
                payloadData.put("amount", 100.0 + (random.nextDouble() * 900.0));
                payloadData.put("items", Arrays.asList("item1", "item2", "item3"));
                break;
                
            case "PAYMENT_PROCESSED":
                payloadData.put("paymentId", "payment_" + random.nextInt(10000));
                payloadData.put("orderId", "order_" + random.nextInt(10000));
                payloadData.put("amount", 50.0 + (random.nextDouble() * 500.0));
                payloadData.put("status", "SUCCESS");
                break;
        }
        
        return objectMapper.writeValueAsString(payloadData);
    }
}
```

### `grpc-client/src/main/java/com/example/grpcclient/config/GrpcClientConfig.java`

```java
package com.example.grpcclient.config;

import org.springframework.context.annotation.Configuration;

@Configuration
public class GrpcClientConfig {
    // Additional gRPC client configuration can be added here if needed
}
```

### `grpc-client/src/main/resources/application.yml`

```yaml
server:
  port: 8081

grpc:
  client:
    grpc-server:
      address: 'static://localhost:9090'
      negotiation-type: PLAINTEXT

spring:
  application:
    name: grpc-client

logging:
  level:
    com.example.grpcclient: DEBUG
    net.devh.boot.grpc: DEBUG
```

## 4. How to Run the Services

### Prerequisites
- Java 17 or higher
- Gradle 7.x or higher

### Steps to Run

1. **Start the gRPC Server:**
   ```bash
   cd grpc-server
   ./gradlew bootRun
   ```
   The server will start on port 8080 (HTTP) and 9090 (gRPC).

2. **Start the gRPC Client:**
   ```bash
   cd grpc-client
   ./gradlew bootRun
   ```
   The client will start on port 8081 and automatically connect to the server.

### What Happens

1. **Server starts** and begins listening for gRPC connections on port 9090
2. **Client starts** and immediately subscribes to events from the server
3. **Client scheduler** begins sending events every 30 seconds and heartbeat every 2 minutes
4. **Server receives** events and broadcasts them to all interested subscribers
5. **Client receives** the broadcasted events (including its own) and processes them

### Key Features

- **Event Streaming**: Server-side streaming for real-time event delivery
- **Event Broadcasting**: Server broadcasts events to all interested subscribers
- **Automatic Reconnection**: Client automatically reconnects if connection is lost
- **Scheduled Events**: Client sends different types of events on a schedule
- **Event Processing**: Client processes different event types with specific handlers
- **Thread Safety**: Uses thread-safe collections for managing subscribers
- **Error Handling**: Comprehensive error handling and logging

### Testing

You can test the system by:
1. Starting both services
2. Watching the logs to see events being sent and received
3. Adding more client instances to see multi-subscriber functionality
4. Stopping and restarting services to test reconnection

The services will automatically generate and exchange various event types including USER_CREATED, ORDER_PLACED, PAYMENT_PROCESSED, and HEARTBEAT events.
