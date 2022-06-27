# Exegete



Exegete is a transformer. It accepts Pulsar messages created from database modifications and publishes them as commands on Pulsar.  It is useful for developers who want to integrate legacy applications with microservices and Event Sourcing architecture.  If Event Sourcing is new to you and you want to understand what it’s useful for, Derek Comartin provides a very good introduction with examples.



## Environment

Apache Pulsar 2.8 or above

Requires at least Java 17 and Spring Boot 2.5.6

Wortel 0.2 (for commands)

Docker (for tests)



## Background

We originally created Exegete to help us move from monolithic to Event Sourcing architecture.  Event Sourcing architecture has two main components; an event store, basically a log of all the events that happened in a database, and an event bus, which routes events to the consumers that need to know about them.

We’d identified Pulsar as the best platform for doing Event Sourcing. Pulsar is a message broker – its primary function is to shunt messages between microservices.  Its message brokering capacity made it the perfect event bus and it had infinite message retention, meaning it could function as an event store in principle.  We really liked the fact that Pulsar supported transactions. Often in enterprise applications, a database update involves a synchronised group of operations, all of which must happen in order for the update to go ahead.

For example, imagine you transfer $20 between two bank accounts.  For this to be a valid transfer, your account must be debited $20 and the recipient account must be credited by exactly the same amount.  The fact that Pulsar supported transactions made it ideal for our application.

There was just one problem. You could only query messages by time. This meant that Pulsar couldn’t meaningfully store Aggregates. Aggregate is Event Sourcing jargon for an entity in a database model, such as Pet or Customer.  Because database queries are always asking questions about relations between entities (all the Pets bought by a Customer), an Event Sourcing application needs to sort events by Aggregate, not just time.



We bolted on MongoDB to function as an Aggregate store.  This led to another problem. Mongo DB was effectively doing duty as our event store while Pulsar was the event bus. We created a framework called Wortel to ensure that whenever one was updated, the other was too.

We got to the point where we’d created a shiny new application with Pulsar and MongoDB… but we were still using the old monolith!  We needed a way for the legacy application to talk to our new system. We hooked our legacy database to Debezium, a system that watches the database and sends a message when a CUD event happens.

Debezium could publish its messages to Pulsar, but in order for these to be of any use they needed to be turned into commands.  We created Exegete to do exactly this.

## How Exegete works

At the start we said that Exegete transforms messages into commands. In Event Sourcing a command is like an item on a to-do list.  Exegete gets its concept of a command from Wortel, through the commands API. Here, a command is implemented as Aggregate type plus C/U/D operation.

Exegete uses the Observer pattern to pick up messages from a given database. A transformer is defined for a given table. This transformer will be implemented for a specific command. For example, you might have a CreatePetTransformer listening for any time a Pet object is added to a pet store database. This transformer implements a “transform” method. This takes in a list of Data Payloads and returns a list of the corresponding commands. It also creates a unique Pulsar Topic for the commands to be routed down. Once in Pulsar, they can be picked up by Striker and published as events.

### Handling Transactions

Exegete stores database transactions as collections of messages with a unique transaction id. This allows the framework to represent database transactions as sets of commands.



### Validation and Error Handling

Exegete validates that the command is of the correct type through a condition checker.  This ensures that CreatePetTranformer does not respond to an Pet update message.

**Transactions**

Exegete allows up to fifty payloads in transactions. If you try to add any more, the message: “To many payloads in transaction” is printed to the log.

**Duplicate Pulsar messages**

To prevent Pulsar messages from being double counted, Exegete store the id of each message in MongoDB for one hour. For each message, Exegete checks its id against MongoDB. If it finds a match the message “Duplicate payload. Payload lsn:{}, Message id: {}, Table:{}" is printed to the log.





## Getting Started



Exegete can be downloaded through Maven. Paste the following Maven dependencies into your POM file.

```xml

<dependency>
    <groupId>com.transportexchangegroup</groupId>
    <artifactId>wortel-striker</artifactId>
    <version>2.3</version>
</dependency>
<dependency>
    <groupId>com.transportexchangegroup</groupId>
    <artifactId>wortel-trigger</artifactId>
    <version>2.3</version>
</dependency>
<dependency>
    <groupId>com.transportexchangegroup</groupId>
    <artifactId>exegete</artifactId>
    <version>0.0.9</version>
</dependency>

```

You can use Exegete with the following import statement:

```java
com.transportexchangegroup.exegete.*;
```

## Creating a Transformer

All transformers have to implement the ExegeteTransformer interface.
```java

package com.transportexchangegroup.fx.exegete.transformer.members;

import com.transportexchangegroup.exegete.api.ExegeteTransformer;
import com.transportexchangegroup.exegete.api.ExegeteTransformerConfig;
import com.transportexchangegroup.exegete.api.dto.CdcDataPayload;
import com.transportexchangegroup.exegete.util.JsonConverter;
import com.transportexchangegroup.fx.command.member.member.CreateMemberCommand;
import com.transportexchangegroup.fx.exegete.transformer.checkers.CreateConditionChecker;
import com.transportexchangegroup.fx.exegete.transformer.repository.UserService;
import lombok.Getter;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.Collection;
import java.util.List;

@Getter
@RequiredArgsConstructor
@Component
@Slf4j
public class CreateMemberTransformer implements ExegeteTransformer {

    private final JsonConverter jsonConverter;
    private final UserService userService;

    private final ExegeteTransformerConfig config = new ExegeteTransformerConfig(
            List.of("members", "users"),
            "member-commands",
            new CreateConditionChecker("members")
    );

    @Override
    public Collection<CreateMemberCommand> transform(List<CdcDataPayload> payloads) {
        CreateMemberCommand createMemberCommand = new MemberCommandBuilder(jsonConverter, userService)
                .buildCreateMemberCommand(payloads);
        return createMemberCommand == null ? null : List.of(createMemberCommand);
    }
}
```