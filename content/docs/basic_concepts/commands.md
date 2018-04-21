---
title: "Commands"
lesson: 4
chapter: 2
cover: "https://unsplash.it/400/300/?random?BoldMage"
date: "01/07/2018"
category: "akkatecture"
type: "lesson"
tags:
    - basic-concepts
    - akkatecture
    - csharp
    - dotnet
---

Commands are the basic value objects, or models, that represent write operations that you can perform in your domain.

As an example, one might implement create this command for initiating a bank transfer from.

```csharp
public class TransferMoneyCommand : Command<AccountAggregate, AccountId>
{
    public Money Amount { get; }
    public DestinationAccountId { get; }

    public UserUpdatePasswordCommand(
        AccountId id,
        AccountId destinationAccountId,
        Money amount)
        : base(id)
    {
        Amount = amount;
        DestinationAccountId = destinationAccountId;
    }
}
```

> Note that the Money class is merely a value object created to hold the password and do basic validation. Read the article regarding value objects for more information. Also, you don’t have to use the default Akkatecture `Command<,>` implementation, you can create your own, it merely have to implement the `ICommand<,>` interface.

A command by itself doesn’t do anything and will be swollowed by the underlying actor as unprocessed. To make a command work, you need to implement at least command handler which is responsible for invoking the aggregate's command handler.


```csharp
    public class AccountAggregate : AggregateRoot<AccountAggregate, AccountAggregateId, AccountState>
    {
        public AccountAggregate(AccountAggregateId aggregateId)
            : base(aggregateId)
        {
            Command<TransferMoneyCommand>(Execute)
        }

        public bool Execute(TransferMoneyCommand command)
        {
          if(State.Balance < command.Amount)
          {
            //Domain Error, not enough money to send
          }
          if(Id == command.DestinationAccountId)
          {
            //Domain Error, cant send money to yourself
          }

          Emit(new MoneyTransferedEvent(command.Amount, command.DestinationAccountId));

          //tell akkas underlying actor that you handled the command
          return true;
        }
    }
```


## Ensure Idempotency

Detecting duplicate operations can be hard, especially if you have a distributed application, or simply a web application. Consider the following simplified scenario.

1. The user wants to send her money.
2. The user fills in the "send money form".
3. As user is impatient, or by accident, the user submits the for twice.
4. The first web request completes, is validated, and the money is sent. However, as the browser is waiting on the first web request, this result is ignored
5. The second web request either transfers money again since there is enough balance, or  throws a domain error as there is no more balance left in the account.
6. The user is presented with a error on the web page, or has accidently sent money twice when she only meant to send it once.

Since Akkatectures design decision dictates that aggregate roots exist as a singleton, we can deal with idempotency at the aggregate level.

We can redesign our command to look like this

```csharp
public class TransferMoneyCommand : Command<AccountAggregate, AccountId>
{
    public Money Amount { get; }
    public DestinationAccountId { get; }

    public UserUpdatePasswordCommand(
        AccountId id,
        ISourceId sourceId,
        AccountId destinationAccountId,
        Money amount)
        : base(id, sourceId)
    {
        Amount = amount;
        DestinationAccountId = destinationAccountId;
    }
}
```

Note the use of the other `protected` constructor of `Command<,>` that takes a `ISourceId` in addition to the aggregate root identity. This sourceId can be supplied from outside the aggregate boundary eg the API surface.
You can then use a circular buffer or "list of processed" commands within your aggregate root to reject previously seen commands.

## Easier ISourceId calculation
Ensuring the correct calculation of the command `ISourceId` can be somewhat cumbersome, which is why Akkatecture provides another base command you can use, the `DistinctCommand<,>`. By using the `DistinctCommand<,>` you merely have to implement the `GetSourceIdComponents()` and providing the `IEnumerable<byte[]>` that makes the command unique. The bytes is used to create a deterministic GUID to be used as an ISourceId.

```csharp
public class TransferMoneyCommand : DistinctCommand<AccountAggregate, AccountId>
{
    public Money Amount { get; }
    public DestinationAccountId { get; }

    public UserUpdatePasswordCommand(
        AccountId id,
        ISourceId sourceId,
        AccountId destinationAccountId,
        Money amount)
        : base(id)
    {
        Amount = amount;
        DestinationAccountId = destinationAccountId;
    }

    protected override IEnumerable<byte[]> GetSourceIdComponents()
    {
      yield return Amount.GetBytes();
      yield return DestinationAccountId.GetBytes();
    }
}
```

The `GetBytes()` merely returns the `Encoding.UTF8.GetBytes(...)` of the value object.