# gRPC Bidirectional Order Processing System

This system implements a bidirectional communication pattern where:
1. **Server sends order events** (with order numbers) to clients
2. **Client receives events**, fetches order data internally, and sends data back to server
3. **Client can request order status** from server
4. **Async processing** on both sides

## Architecture Flow

```
Server ──────────────► Client
   │    Order Events      │
   │   (Order Numbers)    │
   │                      │
   │                      ▼
   │                Client fetches order
   │                data internally
   │                      │
   │                      ▼
   ◄──────────────────── Client
    Order Data Response   │
                          │
Client ──────────────► Server
    Status Request        │
   (Order Number)         │
                          ▼
Client ◄────────────── Server
        Status Response
```

## Project Structure

```
order-processing-system/
├── order-server/
│   ├── src/main/java/com/example/orderserver/
│   │   ├── OrderServerApplication.java
│   │   ├── service/OrderEventService.java
│   │   ├── service/OrderDataService.java
│   │   ├── service/OrderStatusService.java
│   │   ├── model/Order.java
│   │   ├── repository/OrderRepository.java
│   │   └── scheduler/OrderEventScheduler.java
│   ├── src/main/proto/
│   │   └── order.proto
│   ├── src/main/resources/
│   │   └── application.yml
│   ├── build.gradle
│   └── settings.gradle
└── order-client/
    ├── src/main/java/com/example/orderclient/
    │   ├── OrderClientApplication.java
    │   ├── service/OrderEventClientService.java
    │   ├── service/OrderDataFetcherService.java
    │   ├── service/OrderStatusClientService.java
    │   ├── model/ClientOrder.java
    │   └── repository/ClientOrderRepository.java
    ├── src/main/proto/
    │   └── order.proto
    ├── src/main/resources/
    │   └── application.yml
    ├── build.gradle
    └── settings.gradle
```

## 1. Protocol Buffer Definition

### `src/main/proto/order.proto` (Same for both services)

```protobuf
syntax = "proto3";

package com.example.grpc;

option java_multiple_files = true;
option java_package = "com.example.grpc";
option java_outer_classname = "OrderProto";

// Order event - lightweight notification with order number
message OrderEvent {
  string event_id = 1;
  string event_type = 2;  // ORDER_CREATED, ORDER_UPDATED, ORDER_READY_FOR_PROCESSING
  string order_number = 3;
  int64 timestamp = 4;
  string priority = 5;  // HIGH, MEDIUM, LOW
  map<string, string> metadata = 6;
}

// Client subscription request
message OrderEventSubscriptionRequest {
  string client_id = 1;
  repeated string event_types = 2;
}

// Order data that client sends back to server after fetching internally
message OrderData {
  string order_number = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  double total_amount = 4;
  string shipping_address = 5;
  string order_status = 6;
  int64 order_date = 7;
  string client_id = 8;  // Which client processed this
  int64 processed_timestamp = 9;
}

message OrderItem {
  string item_id = 1;
  string item_name = 2;
  int32 quantity = 3;
  double price = 4;
}

// Client sends order data to server
message OrderDataSubmission {
  string submission_id = 1;
  string original_event_id = 2;
  OrderData order_data = 3;
  string processing_notes = 4;
}

// Server response to order data submission
message OrderDataSubmissionResponse {
  bool success = 1;
  string submission_id = 2;
  string message = 3;
  string server_order_status = 4;  // Updated status on server side
}

// Client requests order status from server
message OrderStatusRequest {
  string order_number = 1;
  string requesting_client_id = 2;
}

// Server responds with order status
message OrderStatusResponse {
  bool found = 1;
  string order_number = 2;
  string current_status = 3;
  string last_updated_by = 4;
  int64 last_updated_timestamp = 5;
  string additional_info = 6;
}

// Order Event Service - Server sends events to clients
service OrderEventService {
  rpc SubscribeToOrderEvents(OrderEventSubscriptionRequest) returns (stream OrderEvent);
}

// Order Data Service - Bidirectional data exchange
service OrderDataService {
  // Client sends order data back to server
  rpc SubmitOrderData(OrderDataSubmission) returns (OrderDataSubmissionResponse);
  
  // Client requests order status from server
  rpc GetOrderStatus(OrderStatusRequest) returns (OrderStatusResponse);
}
```

## 2. Order Server Implementation

### `order-server/settings.gradle`

```gradle
rootProject.name = 'order-server'
```

### `order-server/build.gradle`

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
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.h2database:h2'
    
    // gRPC dependencies
    implementation "net.devh:grpc-spring-boot-starter:${grpcSpringBootStarterVersion}"
    implementation "io.grpc:grpc-protobuf:${grpcVersion}"
    implementation "io.grpc:grpc-stub:${grpcVersion}"
    implementation "com.google.protobuf:protobuf-java:${protobufVersion}"
    
    // JSON processing
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    
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

### `order-server/src/main/java/com/example/orderserver/OrderServerApplication.java`

```java
package com.example.orderserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
@EnableAsync
public class OrderServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServerApplication.class, args);
    }
}
```

### `order-server/src/main/java/com/example/orderserver/model/Order.java`

```java
package com.example.orderserver.model;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String orderNumber;
    
    private String customerId;
    private String status;
    private Double totalAmount;
    private String shippingAddress;
    
    @Column(name = "order_date")
    private LocalDateTime orderDate;
    
    @Column(name = "last_updated")
    private LocalDateTime lastUpdated;
    
    @Column(name = "last_updated_by")
    private String lastUpdatedBy;
    
    private String additionalInfo;

    // Constructors
    public Order() {}
    
    public Order(String orderNumber, String customerId, String status, Double totalAmount) {
        this.orderNumber = orderNumber;
        this.customerId = customerId;
        this.status = status;
        this.totalAmount = totalAmount;
        this.orderDate = LocalDateTime.now();
        this.lastUpdated = LocalDateTime.now();
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getOrderNumber() { return orderNumber; }
    public void setOrderNumber(String orderNumber) { this.orderNumber = orderNumber; }

    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }

    public String getStatus() { return status; }
    public void setStatus(String status) { 
        this.status = status;
        this.lastUpdated = LocalDateTime.now();
    }

    public Double getTotalAmount() { return totalAmount; }
    public void setTotalAmount(Double totalAmount) { this.totalAmount = totalAmount; }

    public String getShippingAddress() { return shippingAddress; }
    public void setShippingAddress(String shippingAddress) { this.shippingAddress = shippingAddress; }

    public LocalDateTime getOrderDate() { return orderDate; }
    public void setOrderDate(LocalDateTime orderDate) { this.orderDate = orderDate; }

    public LocalDateTime getLastUpdated() { return lastUpdated; }
    public void setLastUpdated(LocalDateTime lastUpdated) { this.lastUpdated = lastUpdated; }

    public String getLastUpdatedBy() { return lastUpdatedBy; }
    public void setLastUpdatedBy(String lastUpdatedBy) { 
        this.lastUpdatedBy = lastUpdatedBy;
        this.lastUpdated = LocalDateTime.now();
    }

    public String getAdditionalInfo() { return additionalInfo; }
    public void setAdditionalInfo(String additionalInfo) { this.additionalInfo = additionalInfo; }
}
```

### `order-server/src/main/java/com/example/orderserver/repository/OrderRepository.java`

```java
package com.example.orderserver.repository;

import com.example.orderserver.model.Order;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    Optional<Order> findByOrderNumber(String orderNumber);
    boolean existsByOrderNumber(String orderNumber);
}
```

### `order-server/src/main/java/com/example/orderserver/service/OrderEventService.java`

```java
package com.example.orderserver.service;

import com.example.grpc.OrderEvent;
import com.example.grpc.OrderEventServiceGrpc;
import com.example.grpc.OrderEventSubscriptionRequest;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

@GrpcService
@Service
public class OrderEventService extends OrderEventServiceGrpc.OrderEventServiceImplBase {

    private static final Logger logger = LoggerFactory.getLogger(OrderEventService.class);
    
    // Thread-safe list to store active subscribers
    private final List<OrderEventSubscriber> subscribers = new CopyOnWriteArrayList<>();

    @Override
    public void subscribeToOrderEvents(OrderEventSubscriptionRequest request,
                                      StreamObserver<OrderEvent> responseObserver) {
        
        String clientId = request.getClientId();
        List<String> eventTypes = request.getEventTypesList();
        
        logger.info("New order event subscriber: {} subscribing to types: {}", clientId, eventTypes);
        
        // Create subscriber
        OrderEventSubscriber subscriber = new OrderEventSubscriber(clientId, eventTypes, responseObserver);
        subscribers.add(subscriber);
        
        logger.info("Total active order event subscribers: {}", subscribers.size());
        
        // Handle client disconnection (in real implementation, you'd use onError callback)
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            subscribers.remove(subscriber);
            logger.info("Subscriber {} disconnected. Active subscribers: {}", 
                       clientId, subscribers.size());
        }));
    }

    /**
     * Broadcast order event to all interested subscribers
     */
    public void broadcastOrderEvent(OrderEvent orderEvent) {
        logger.info("Broadcasting order event: {} for order: {}", 
                   orderEvent.getEventType(), orderEvent.getOrderNumber());
        
        int subscriberCount = 0;
        for (OrderEventSubscriber subscriber : subscribers) {
            if (subscriber.isInterestedInEvent(orderEvent.getEventType())) {
                try {
                    subscriber.getResponseObserver().onNext(orderEvent);
                    subscriberCount++;
                    logger.debug("Order event {} sent to subscriber {}", 
                               orderEvent.getEventId(), subscriber.getClientId());
                } catch (Exception e) {
                    logger.error("Error sending order event to subscriber {}: {}", 
                               subscriber.getClientId(), e.getMessage());
                    subscribers.remove(subscriber);
                }
            }
        }
        
        logger.info("Order event broadcast to {} subscribers", subscriberCount);
    }

    /**
     * Inner class to hold subscriber information
     */
    private static class OrderEventSubscriber {
        private final String clientId;
        private final List<String> eventTypes;
        private final StreamObserver<OrderEvent> responseObserver;

        public OrderEventSubscriber(String clientId, List<String> eventTypes, 
                                   StreamObserver<OrderEvent> responseObserver) {
            this.clientId = clientId;
            this.eventTypes = eventTypes;
            this.responseObserver = responseObserver;
        }

        public boolean isInterestedInEvent(String eventType) {
            return eventTypes.isEmpty() || eventTypes.contains(eventType);
        }

        public String getClientId() { return clientId; }
        public StreamObserver<OrderEvent> getResponseObserver() { return responseObserver; }
    }
}
```

### `order-server/src/main/java/com/example/orderserver/service/OrderDataService.java`

```java
package com.example.orderserver.service;

import com.example.grpc.*;
import com.example.orderserver.model.Order;
import com.example.orderserver.repository.OrderRepository;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Async;

import java.time.ZoneOffset;
import java.util.Optional;

@GrpcService
public class OrderDataService extends OrderDataServiceGrpc.OrderDataServiceImplBase {

    private static final Logger logger = LoggerFactory.getLogger(OrderDataService.class);

    @Autowired
    private OrderRepository orderRepository;

    @Override
    public void submitOrderData(OrderDataSubmission request, 
                               StreamObserver<OrderDataSubmissionResponse> responseObserver) {
        
        String orderNumber = request.getOrderData().getOrderNumber();
        String clientId = request.getOrderData().getClientId();
        
        logger.info("Received order data submission from client {} for order {}", clientId, orderNumber);
        
        // Process the order data asynchronously
        processOrderDataAsync(request);
        
        // Send immediate response
        OrderDataSubmissionResponse response = OrderDataSubmissionResponse.newBuilder()
                .setSuccess(true)
                .setSubmissionId(request.getSubmissionId())
                .setMessage("Order data received and being processed")
                .setServerOrderStatus("PROCESSING")
                .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
        
        logger.info("Order data submission response sent to client {}", clientId);
    }

    @Override
    public void getOrderStatus(OrderStatusRequest request, 
                              StreamObserver<OrderStatusResponse> responseObserver) {
        
        String orderNumber = request.getOrderNumber();
        String clientId = request.getRequestingClientId();
        
        logger.info("Received order status request from client {} for order {}", clientId, orderNumber);
        
        Optional<Order> orderOpt = orderRepository.findByOrderNumber(orderNumber);
        
        OrderStatusResponse.Builder responseBuilder = OrderStatusResponse.newBuilder()
                .setOrderNumber(orderNumber);
        
        if (orderOpt.isPresent()) {
            Order order = orderOpt.get();
            responseBuilder
                    .setFound(true)
                    .setCurrentStatus(order.getStatus())
                    .setLastUpdatedBy(order.getLastUpdatedBy() != null ? order.getLastUpdatedBy() : "SYSTEM")
                    .setLastUpdatedTimestamp(order.getLastUpdated().toEpochSecond(ZoneOffset.UTC))
                    .setAdditionalInfo(order.getAdditionalInfo() != null ? order.getAdditionalInfo() : "");
            
            logger.info("Order status found for {}: {}", orderNumber, order.getStatus());
        } else {
            responseBuilder
                    .setFound(false)
                    .setCurrentStatus("NOT_FOUND")
                    .setLastUpdatedBy("SYSTEM")
                    .setLastUpdatedTimestamp(System.currentTimeMillis() / 1000)
                    .setAdditionalInfo("Order not found in system");
            
            logger.warn("Order not found: {}", orderNumber);
        }
        
        responseObserver.onNext(responseBuilder.build());
        responseObserver.onCompleted();
    }

    @Async
    public void processOrderDataAsync(OrderDataSubmission submission) {
        try {
            // Simulate processing delay
            Thread.sleep(2000);
            
            OrderData orderData = submission.getOrderData();
            String orderNumber = orderData.getOrderNumber();
            
            // Update or create order in database
            Optional<Order> existingOrderOpt = orderRepository.findByOrderNumber(orderNumber);
            Order order;
            
            if (existingOrderOpt.isPresent()) {
                order = existingOrderOpt.get();
                logger.info("Updating existing order: {}", orderNumber);
            } else {
                order = new Order();
                order.setOrderNumber(orderNumber);
                logger.info("Creating new order: {}", orderNumber);
            }
            
            // Update order details from client data
            order.setCustomerId(orderData.getCustomerId());
            order.setStatus("PROCESSED_BY_CLIENT");
            order.setTotalAmount(orderData.getTotalAmount());
            order.setShippingAddress(orderData.getShippingAddress());
            order.setLastUpdatedBy(orderData.getClientId());
            order.setAdditionalInfo("Processed by client: " + orderData.getClientId() + 
                                  ". Items: " + orderData.getItemsCount());
            
            orderRepository.save(order);
            
            logger.info("Order {} successfully processed and updated in database", orderNumber);
            
        } catch (Exception e) {
            logger.error("Error processing order data: {}", e.getMessage(), e);
        }
    }
}
```

### `order-server/src/main/java/com/example/orderserver/scheduler/OrderEventScheduler.java`

```java
package com.example.orderserver.scheduler;

import com.example.grpc.OrderEvent;
import com.example.orderserver.service.OrderEventService;
import com.example.orderserver.model.Order;
import com.example.orderserver.repository.OrderRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.List;
import java.util.Random;

@Component
public class OrderEventScheduler {

    private static final Logger logger = LoggerFactory.getLogger(OrderEventScheduler.class);

    @Autowired
    private OrderEventService orderEventService;
    
    @Autowired
    private OrderRepository orderRepository;

    private final Random random = new Random();
    private final List<String> eventTypes = Arrays.asList(
        "ORDER_CREATED", "ORDER_READY_FOR_PROCESSING", "ORDER_UPDATED"
    );
    private final List<String> priorities = Arrays.asList("HIGH", "MEDIUM", "LOW");

    // Send order events every 20 seconds
    @Scheduled(fixedRate = 20000)
    public void sendOrderEvents() {
        try {
            String orderNumber = generateOrderNumber();
            String eventType = eventTypes.get(random.nextInt(eventTypes.size()));
            String priority = priorities.get(random.nextInt(priorities.size()));
            
            // Create order in database if it's a new order
            if ("ORDER_CREATED".equals(eventType)) {
                createSampleOrder(orderNumber);
            }
            
            // Create and broadcast order event
            OrderEvent orderEvent = OrderEvent.newBuilder()
                    .setEventId("evt_" + System.currentTimeMillis())
                    .setEventType(eventType)
                    .setOrderNumber(orderNumber)
                    .setTimestamp(System.currentTimeMillis())
                    .setPriority(priority)
                    .putMetadata("source", "order-system")
                    .putMetadata("region", "US-EAST")
                    .build();
            
            logger.info("Generating order event: {} for order: {} with priority: {}", 
                       eventType, orderNumber, priority);
            
            orderEventService.broadcastOrderEvent(orderEvent);
            
        } catch (Exception e) {
            logger.error("Error generating order event: {}", e.getMessage());
        }
    }

    private String generateOrderNumber() {
        return "ORD-" + System.currentTimeMillis() + "-" + random.nextInt(1000);
    }
    
    private void createSampleOrder(String orderNumber) {
        if (!orderRepository.existsByOrderNumber(orderNumber)) {
            Order order = new Order(
                orderNumber,
                "CUST-" + random.nextInt(1000),
                "CREATED",
                100.0 + (random.nextDouble() * 900.0)
            );
            order.setShippingAddress("123 Sample St, City, State");
            order.setAdditionalInfo("Sample order created by scheduler");
            
            orderRepository.save(order);
            logger.info("Sample order created: {}", orderNumber);
        }
    }
}
```

### `order-server/src/main/resources/application.yml`

```yaml
server:
  port: 8080

grpc:
  server:
    port: 9090

spring:
  application:
    name: order-server
  datasource:
    url: jdbc:h2:mem:orderdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: create-drop
    show-sql: false
  h2:
    console:
      enabled: true
      path: /h2-console

logging:
  level:
    com.example.orderserver: DEBUG
    net.devh.boot.grpc: DEBUG
```

## 3. Order Client Implementation

### `order-client/settings.gradle`

```gradle
rootProject.name = 'order-client'
```

### `order-client/build.gradle`

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
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'com.h2database:h2'
    
    // gRPC dependencies
    implementation "net.devh:grpc-client-spring-boot-starter:${grpcSpringBootStarterVersion}"
    implementation "io.grpc:grpc-protobuf:${grpcVersion}"
    implementation "io.grpc:grpc-stub:${grpcVersion}"
    implementation "com.google.protobuf:protobuf-java:${protobufVersion}"
    
    // JSON processing
    implementation 'com.fasterxml.jackson.core:jackson-databind'
    
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

### `order-client/src/main/java/com/example/orderclient/OrderClientApplication.java`

```java
package com.example.orderclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;

@SpringBootApplication
@EnableAsync
public class OrderClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderClientApplication.class, args);
    }
}
```

### `order-client/src/main/java/com/example/orderclient/model/ClientOrder.java`

```java
package com.example.orderclient.model;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "client_orders")
public class ClientOrder {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String orderNumber;
    
    private String customerId;
    private String customerName;
    private String customerEmail;
    private Double totalAmount;
    private String status;
    private String shippingAddress;
    private String itemsJson;  // Store items as JSON string
    
    @Column(name = "order_date")
    private LocalDateTime orderDate;
    
    @Column(name = "processed_date")
    private LocalDateTime processedDate;

    // Constructors
    public ClientOrder() {}
    
    public ClientOrder(String orderNumber, String customerId, Double totalAmount) {
        this.orderNumber = orderNumber;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        this.orderDate = LocalDateTime.now();
        this.status = "PENDING";
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getOrderNumber() { return orderNumber; }
    public void setOrderNumber(String orderNumber) { this.orderNumber = orderNumber; }

    public String getCustomerId() { return customerId; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }

    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }

    public String getCustomerEmail() { return customerEmail; }
    public void setCustomerEmail(String customerEmail) { this.customerEmail = customerEmail; }

    public Double getTotalAmount() { return totalAmount; }
    public void setTotalAmount(Double totalAmount) { this.totalAmount = totalAmount; }

    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }

    public String getShippingAddress() { return shippingAddress; }
    public void setShippingAddress(String shippingAddress) { this.shippingAddress = shippingAddress; }

    public String getItemsJson() { return itemsJson; }
    public void setItemsJson(String itemsJson) { this.itemsJson = itemsJson; }

    public LocalDateTime getOrderDate() { return orderDate; }
    public void setOrderDate(LocalDateTime orderDate) { this.orderDate = orderDate; }

    public LocalDateTime getProcessedDate() { return processedDate; }
    public void setProcessedDate(LocalDateTime processedDate) { this.processedDate = processedDate; }
}
```

### `order-client/src/main/java/com/example/orderclient/repository/ClientOrderRepository.java`

```java
package com.example.orderclient.repository;

import com.example.orderclient.model.ClientOrder;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface ClientOrderRepository extends JpaRepository<ClientOrder, Long> {
