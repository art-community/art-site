# ART Examples

## Value & POJO
Value is a universal data model for representing data as JSON/XML/Protobuf/Tuple or yore custom data format.

For optimization ART ***is not*** using reflection for serialization and deserialization POJOs to/from Value.

Instead of it ART provides ValueMapping API and ValueMapping Generator (from ART application-gradle-plugin).

Code:
```java
import lombok.*;
import ru.art.entity.*;
import ru.art.entity.mapper.ValueFromModelMapper.*;
import ru.art.entity.mapper.ValueToModelMapper.*;
import ru.art.entity.xml.*;
import static ru.art.entity.Entity.*;
import static ru.art.json.descriptor.JsonEntityWriter.*;
import static ru.art.protobuf.descriptor.ProtobufEntityWriter.*;
import static ru.art.xml.descriptor.XmlEntityWriter.*;
import java.util.*;

public class MainModule {
    @Getter
    @Builder
    @ToString
    private static class Customer {
        private final String id;
        private String name;
        @Singular
        private List<String> orderIds;
    }

    public static void main(String[] args) {
        EntityFromModelMapper<Customer> fromCustomer = customer -> entityBuilder()
                .stringField("id", customer.getId())
                .stringField("name", customer.getName())
                .stringCollectionField("orderIds", customer.getOrderIds())
                .build();
        EntityToModelMapper<Customer> toCustomer = entity -> Customer.builder()
                .id(entity.getString("id"))
                .name(entity.getString("name"))
                .orderIds(entity.getStringList("orderIds"))
                .build();
        Customer customer = Customer.builder().id("1").name("Customer").orderId("1").orderId("2").build();
        System.out.println("Customer = " + customer);
        Entity entity = fromCustomer.map(customer);
        System.out.println("Customer entity from customer = " + entity);
        System.out.println("Customer entity as JSON = " + writeJson(entity));
        System.out.println("Customer entity as Protobuf = " + writeProtobuf(entity));
        System.out.println("Customer entity as XML = " + writeXml(XmlEntityFromEntityConverter.fromEntityAsTags(entity)));
        System.out.println("Customer entity to customer = " + toCustomer.map(entity));
    }
}
```

After running in output console you could see something like this:
```
Customer = MainModule.Customer(id=1, name=Customer, orderIds=[1, 2])
Customer entity from customer = Entity(fields={id=1, name=Customer, orderIds=[1, 2]}, fieldNames=[id, name, orderIds], type=ENTITY)
Customer entity as JSON = {"id":"1","name":"Customer","orderIds":["1","2"]}
Customer entity as Protobuf = value {
  type_url: "type.googleapis.com/ru.art.protobuf.entity.ProtobufEntity"
  value: "\nI\n\004name\022A\n=\n/type.googleapis.com/google.protobuf.StringValue\022\n\n\bCustomer\020\005\n\315\001\n\borderIds\022\300\001\n\273\001\n=type.googleapis.com/ru.art.protobuf.entity.ProtobufCollection\022z\b\005\022:\n6\n/type.googleapis.com/google.protobuf.StringValue\022\003\n\0011\020\005\022:\n6\n/type.googleapis.com/google.protobuf.StringValue\022\003\n\0012\020\005\020\003\n@\n\002id\022:\n6\n/type.googleapis.com/google.protobuf.StringValue\022\003\n\0011\020\005"
}
valueType: ENTITY

Customer entity as XML = <?xml version="1.0" encoding="UTF-8"?>

<id>1</id>
<name>Customer</name>
<orderIds>
<1></1>
<2></2></orderIds>

Customer entity to customer = MainModule.Customer(id=1, name=Customer, orderIds=[1, 2])
```

Alternative way to handle Value/Pojo mapping looks like this
```java
package ru.art.test.specification;

import lombok.*;
import ru.art.entity.Value;
import ru.art.entity.*;
import static java.util.Optional.*;
import static ru.art.core.checker.CheckerForEmptiness.isEmpty;
import static ru.art.core.extension.OptionalExtensions.*;
import static ru.art.core.factory.CollectionsFactory.setOf;
import static ru.art.entity.Entity.*;
import java.util.*;

public class MainModule {
    @Getter
    @ToString
    @AllArgsConstructor
    @EqualsAndHashCode
    static
    class Data {
        private final String dataValue;
    }

    @Getter
    @ToString
    @EqualsAndHashCode
    static
    class Request {
        @ToString.Exclude
        @EqualsAndHashCode.Exclude
        private final Entity entity;
        @Getter(lazy = true)
        private final String id = entity.getString("id");
        @Getter(lazy = true)
        private final Data data = unwrap(ofNullable(entity.getEntity("data"))
                .map(entity -> new Data(entity.getString("dataValue"))));

        Request(Value value) {
            this.entity = Value.asEntity(value);
        }

        public Optional<String> getFirstOrder() {
            List<String> orders = entity.getStringList("orders");
            if (isEmpty(orders)) return empty();
            return ofNullable(orders.get(0));
        }
    }

    @Getter
    @ToString
    @AllArgsConstructor
    @EqualsAndHashCode
    static
    class Response {
        private final String id;

        public Value toValue() {
            return entityBuilder().stringField("id", id).build();
        }
    }

    public static void main(String[] args) {
        Value requestEntity = entityBuilder()
                .stringField("id", "123")
                .stringCollectionField("orders", setOf("order1", "order2"))
                .entityField("data", entityBuilder()
                        .stringField("dataValue", "test")
                        .build())
                .build();
        Request request = new Request(requestEntity);
        Response response = new Response("456");
        Value responseEntity = response.toValue();
        System.out.println("Request entity = " + requestEntity);
        System.out.println("Request = " + request);
        request.getFirstOrder().ifPresent(order -> System.out.println("Request first order = " + order));
        System.out.println("Response = " + response);
        System.out.println("Response entity = " + responseEntity);
    }
}
```

After running this code you could see somethink like this:

```
Request entity = Entity(fields={id=123, orders=[order1, order2], data=Entity(fields={dataValue=test}, fieldNames=[dataValue], type=ENTITY)}, fieldNames=[id, orders, data], type=ENTITY)
Request = MainModule.Request(id=123, data=MainModule.Data(dataValue=test))
Request first order = order1
Response = MainModule.Response(id=456)
Response entity = Entity(fields={id=456}, fieldNames=[id], type=ENTITY)
```

## HTTP Serving
ART provides you functional to serving HTTP requests

Code:
```java
import static ru.art.config.extensions.activator.AgileConfigurationsActivator.*;
import static ru.art.entity.PrimitiveMapping.StringPrimitive.*;
import static ru.art.entity.StringParametersMapping.*;
import static ru.art.http.server.HttpServer.*;
import static ru.art.http.server.function.HttpServiceFunction.*;

public class MainModule {
    public static void main(String[] args) {
        useAgileConfigurations(MainModule.class.getName());
        httpGet("/hello")
                .fromPathParameters("param")
                .requestMapper(stringParameterToStringMapper("param").getToModel())
                .responseMapper(fromModel)
                .handle(requestParam -> "<h1>" + requestParam + "</h1>");
        startHttpServer().await();
    }
}
```

After running and open browser on url with path '/hello' from logs you could see something like this:

![](https://i.ibb.co/Zx8L6JJ/image.png)

## HTTP Communication
ART provides you functional for sending HTTP request

Code:
```java
import static ru.art.config.extensions.activator.AgileConfigurationsActivator.*;
import static ru.art.entity.PrimitiveMapping.StringPrimitive.*;
import static ru.art.http.client.communicator.HttpCommunicator.*;

public class MainModule {
    public static void main(String[] args) {
        useAgileConfigurations(MainModule.class.getName());
        httpCommunicator("http://example.com/")
                .responseMapper(toModel)
                .execute()
                .ifPresent(System.out::println);
    }
}
```

After running this code you could see something like this:
```html
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

    <meta charset="utf-8" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <style type="text/css">
    body {
        background-color: #f0f0f2;
        margin: 0;
        padding: 0;
        font-family: "Open Sans", "Helvetica Neue", Helvetica, Arial, sans-serif;
        
    }
    div {
        width: 600px;
        margin: 5em auto;
        padding: 50px;
        background-color: #fff;
        border-radius: 1em;
    }
    a:link, a:visited {
        color: #38488f;
        text-decoration: none;
    }
    @media (max-width: 700px) {
        body {
            background-color: #fff;
        }
        div {
            width: auto;
            margin: 0 auto;
            border-radius: 0;
            padding: 1em;
        }
    }
    </style>    
</head>

<body>
<div>
    <h1>Example Domain</h1>
    <p>This domain is established to be used for illustrative examples in documents. You may use this
    domain in examples without prior coordination or asking for permission.</p>
    <p><a href="http://www.iana.org/domains/example">More information...</a></p>
</div>
</body>
</html>
```

## GRPC Serving & Communication
ART provides GRPC functional API to serving and handling GRPC requests

Code:
```java
import static ru.art.config.extensions.activator.AgileConfigurationsActivator.*;
import static ru.art.core.constants.NetworkConstants.*;
import static ru.art.core.extension.NullCheckingExtensions.*;
import static ru.art.entity.PrimitiveMapping.StringPrimitive.*;
import static ru.art.grpc.client.communicator.GrpcCommunicator.*;
import static ru.art.grpc.server.GrpcServer.*;
import static ru.art.grpc.server.function.GrpcServiceFunction.*;
import static ru.art.grpc.server.module.GrpcServerModule.*;
import java.util.function.*;

public class MainModule {
    public static void main(String[] args) {
        useAgileConfigurations();
        grpc("myFunction")
                .responseMapper(fromModel)
                .produce(() -> "Hello, ART!");
        startGrpcServer();
        doIfNotNull(grpcCommunicator(LOCALHOST, grpcServerModule().getPort(), grpcServerModule().getPath())
                .functionId("myFunction")
                .responseMapper(toModel)
                .execute().getResponseData(), (Consumer<Object>) System.out::println);
    }
}
```
After running you could see something like this

`Hello, ART!`

## RSocket Serving & Communication
ART provides RSocket functional API to all rsocket methods:
* fireAndForget
* requestResponse
* requestStream
* requestChannel

Code:
```java
import ru.art.rsocket.constants.*;
import ru.art.service.model.*;
import static ru.art.config.extensions.activator.AgileConfigurationsActivator.*;
import static ru.art.core.constants.NetworkConstants.*;
import static ru.art.entity.PrimitiveMapping.StringPrimitive.*;
import static ru.art.rsocket.communicator.RsocketCommunicator.*;
import static ru.art.rsocket.function.RsocketServiceFunction.*;
import static ru.art.rsocket.module.RsocketModule.*;
import static ru.art.rsocket.server.RsocketServer.*;
import java.util.concurrent.*;

public class MainModule {
    public static void main(String[] args) throws InterruptedException {
        useAgileConfigurations();
        CountDownLatch latch = new CountDownLatch(1);
        rsocket("myFunction")
                .responseMapper(fromModel)
                .produce(() -> {
                    latch.countDown();
                    return "Hello, ART!";
                });
        startRsocketWebSocketServer();
        rsocketCommunicator(LOCALHOST, rsocketModule().getServerWebSocketPort(), RsocketModuleConstants.RsocketTransport.WEB_SOCKET)
                .functionId("myFunction")
                .responseMapper(toModel)
                .execute()
                .map(ServiceResponse::getResponseData)
                .subscribe(System.out::println);
        latch.await();
    }
}
```
After running you could see something like this

`Hello, ART!`
## Rocks DB
ART provides API for interact with RocksDB

Code:
```java
import static ru.art.core.factory.CollectionsFactory.*;
import static ru.art.entity.Entity.*;
import static ru.art.rocks.db.dao.RocksDbCollectionDao.*;
import static ru.art.rocks.db.dao.RocksDbPrimitiveDao.put;
import static ru.art.rocks.db.dao.RocksDbPrimitiveDao.*;
import static ru.art.rocks.db.dao.RocksDbValueDao.*;


public class MainModule {
    public static void main(String[] args) {
        put("stringKey", "string");
        putStrings("stringsKey", fixedArrayOf("string1", "string2"));
        putAsProtobuf("customer", entityBuilder()
                .stringField("id", "123")
                .entityField("order", entityBuilder().intField("price", 123).build())
                .build());
        getString("stringKey").ifPresent(string -> System.out.println("String from rocks = " + string));
        System.out.println("Strings from rocks = " + getStringList("stringsKey"));
        getAsProtobuf("customer").ifPresent(customer -> System.out.println("Customer from rocks = " + customer));
    }
}
```
After running you could see something like this
```
String from rocks = string
Strings from rocks = [string1, string2]
Customer from rocks = Entity(fields={id=123, order=Entity(fields={price=123}, fieldNames=[price], type=ENTITY)}, fieldNames=[id, order], type=ENTITY)
```

## Kafka Embedded Broker, stream & producer
ART module application-kafka-broker includes Kafka and Zookeeper and provides functional to startup and manage Kafka brokers.

Also modules application-kafka-consumer and application-kafka-producer provides you API to producing and consuming/streaming kafka messages.

Code:
```java
import ru.art.entity.*;
import static ru.art.config.extensions.activator.AgileConfigurationsActivator.*;
import static ru.art.core.constants.NetworkConstants.*;
import static ru.art.core.constants.StringConstants.*;
import static ru.art.entity.PrimitivesFactory.*;
import static ru.art.kafka.broker.embedded.EmbeddedKafkaBroker.*;
import static ru.art.kafka.broker.module.KafkaBrokerModule.*;
import static ru.art.kafka.consumer.configuration.KafkaStreamConfiguration.streamConfiguration;
import static ru.art.kafka.consumer.container.KafkaStreamContainer.streamContainer;
import static ru.art.kafka.consumer.module.KafkaConsumerModule.kafkaStreamsRegistry;
import static ru.art.kafka.consumer.starter.KafkaStreamsStarter.startKafkaStreams;
import static ru.art.kafka.producer.communicator.KafkaProducerCommunicator.*;
import static ru.art.kafka.producer.configuration.KafkaProducerConfiguration.*;

public class MainModule {
    public static void main(String[] args) {
        useAgileConfigurations();
        startKafkaBroker();
        kafkaProducerCommunicator(producerConfiguration()
                .clientId("producer")
                .topic("topic")
                .broker(LOCALHOST +
                        COLON +
                        kafkaBrokerModule().getKafkaBrokerConfiguration().getPort())
                .build())
                .pushKafkaRecord(stringPrimitive("Hello"), stringPrimitive("ART"));
        kafkaStreamsRegistry().createStream("stream", streamsBuilder -> streamContainer()
                .configuration(streamConfiguration()
                        .broker(LOCALHOST +
                                COLON +
                                kafkaBrokerModule().getKafkaBrokerConfiguration().getPort())
                        .build())
                .stream(streamsBuilder.<Primitive, Primitive>stream("topic")
                        .peek((key, value) -> System.out.println(key + COMMA + value)))
                .assemble());
        startKafkaStreams();
    }
}
```

After running this code you will se a lot of Kafka logs, and at the end something like this
```
Hello,ART
``` 

## Local Scheduler
ART provides functional for Scheduling tasks (deferred and periodic) with controlling task execution time and order

Code:
```java
import static java.time.Duration.*;
import static ru.art.config.extensions.activator.AgileConfigurationsActivator.*;
import static ru.art.core.constants.DateConstants.*;
import static ru.art.core.extension.DateExtensions.*;
import static ru.art.task.deferred.executor.IdentifiedRunnableFactory.*;
import static ru.art.task.deferred.executor.SchedulerModuleActions.*;
import java.util.*;
import java.util.concurrent.*;

public class MainModule {
    public static void main(String[] args) throws InterruptedException {
        useAgileConfigurations();
        CountDownLatch latch = new CountDownLatch(5);
        asynchronousPeriod(uniqueTask(() -> {
            System.out.println("Executed at " + format(HH_MM_SS_24H, new Date()));
            latch.countDown();
        }), ofSeconds(1));
        latch.await();
    }
}
```
After running this code you could see something like this:

```
Module: 'SCHEDULER_MODULE' was loaded in 257[ms]
Executed at 16:24:17
Executed at 16:24:18
Executed at 16:24:19
Executed at 16:24:20
Executed at 16:24:21
```

## Tarantool DB
ART provides API to interact with Tarantool

Code:
```java
import ru.art.config.extensions.tarantool.*;
import ru.art.tarantool.configuration.*;
import ru.art.tarantool.dao.*;
import ru.art.tarantool.module.*;
import static ru.art.config.extensions.activator.AgileConfigurationsActivator.*;
import static ru.art.core.context.Context.context;
import static ru.art.core.factory.CollectionsFactory.mapOf;
import static ru.art.entity.Entity.*;
import static ru.art.tarantool.dao.TarantoolDao.*;
import java.util.*;

public class MainModule {
    public static void main(String[] args) {
        useAgileConfigurations();
        TarantoolDao tarantool = tarantool(TarantoolConfiguration.builder()
                .connectionConfiguration(TarantoolConnectionConfiguration.builder()
                        .host("localhost")
                        .port(3301)
                        .username("user")
                        .password("password")
                        .build())
                .build());
        tarantool.get("entity", tarantool.put("entity", entityBuilder()
                .stringField("name", "Customer name")
                .entityField("order", entityBuilder()
                        .intField("price", 123)
                        .build())
                .build()).getLong("id"))
                .ifPresent(System.out::println);
    }
}
```

After running this code you could see something like this:
```
Module: 'SERVICE_MODULE' was loaded in 144[ms]
Module: 'CONFIG_MODULE' was loaded in 2[ms]
Module: 'LOGGING_MODULE' was loaded in 584[ms]
2019-08-30 16:29:39,074 INFO r.a.t.i.TarantoolInitializer [main] Tarantool 'example' with address 'localhost:3301' successfully connected 
Module: 'TARANTOOL_MODULE' was loaded in 332[ms]
Entity(fields={id=2, name=Customer name, order=Entity(fields={price=123}, fieldNames=[price], type=ENTITY)}, fieldNames=[id, name, order], type=ENTITY)
```

## Configurations
