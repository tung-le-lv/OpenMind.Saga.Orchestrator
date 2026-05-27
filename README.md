# Saga Orchestrator

A comprehensive implementation of the **Saga Orchestrator Pattern** for distributed transactions in a microservices architecture.

## Architecture Overview

This solution implements an **Order Placement Orchestrator** that coordinates a distributed transaction across multiple microservices:

```
┌─────────────────┐
│   Client/UI     │
└────────┬────────┘
         │
         ├─────────────────────────────────────────────────────┐
         │ 1. "create order"                                   │
         ▼                                                     │
┌─────────────────────────────────────────────────────────────┐│
│                       Order Service                         ││
└─────────────────────────────────────────────────────────────┘│
                                                               │
         ┌─────────────────────────────────────────────────────┘
         │ 2. "place order" (with orderId)
         ▼
┌─────────────────────────────────────────────────────────────┐
│              Order Placement Orchestrator                   │
│           (MassTransit State Machine Saga)                  │
│                                                             │
│  States: Initial → Validating → PaymentProcessing →         │
│          Fulfilling → SendingConfirmation → Completed       │
│                                                             │
│  Retryable States: ValidationFailed, PaymentNotPaid         │
│                                                             │
│  Error States: FulfillmentFailed → RefundingPayment →       │
│               SendingBackorderEmail → Cancelled             │
└──────────────────────┬──────────────────────────────────────┘
                       │
    ┌──────────────────┼──────────────────┬───────────────────┐
    │                  │                  │                   │
    ▼                  ▼                  ▼                   ▼
┌───────────┐   ┌───────────┐   ┌──────────────┐   ┌──────────┐
│  Order    │   │  Payment  │   │ Fulfillment  │   │  Email   │
│  Service  │   │  Service  │◄──│   Service    │   │ Service  │
│           │   │           │   │              │   │          │
└─────┬─────┘   └─────┬─────┘   └──────┬───────┘   └──────────┘
      │               │                │                
      ▼               ▼                ▼                
┌───────────┐   ┌───────────┐   ┌──────────────┐   
│  OrderDb  │   │ PaymentDb │   │FulfillmentDb │   
│ (MongoDB) │   │ (MongoDB) │   │  (MongoDB)   │   
└───────────┘   └───────────┘   └──────────────┘

                      ▲
                      │ 3. "retry payment" (when PaymentNotPaid)
                      │    POST /api/payments/retry/{orderId}
┌─────────────────────┴───┐
│   Client/UI             │
│   (on payment failure)  │
└─────────────────────────┘
```

## Features

### Saga Orchestrator Pattern
- **Centralized Workflow Management**: Single point of control for the entire order placement process
- **State Machine Implementation**: Using MassTransit's Automatonymous state machine
- **Compensating Transactions**: Automatic rollback capabilities (refunds, cancellations)
- **Queryable State**: Track order progress at any time

### Clean Architecture per Microservice
Each service follows Clean Architecture with:
- **Domain Layer**: Entities, Value Objects, Aggregates, Domain Events
- **Application Layer**: CQRS (Commands/Queries), Handlers, Validators
- **Infrastructure Layer**: Repositories, Message Consumers
- **API Layer**: Minimal APIs, Swagger documentation

### DDD Tactical Patterns
- **Aggregate Roots**: Order, Payment, Fulfillment, EmailNotification
- **Value Objects**: Money, Address, CustomerId, OrderId
- **Domain Events**: OrderValidated, PaymentCompleted, etc.
- **Strongly Typed IDs**: Type-safe identifiers
- **Smart Enums**: OrderStatus, PaymentStatus, FulfillmentStatus

### Technology Stack
- **.NET 10** with C# 13
- **MassTransit** for messaging and saga orchestration
- **MongoDB** for persistence
- **MediatR** for CQRS
- **FluentValidation** for request validation
- **Serilog** for structured logging

## Workflow Scenarios

### Happy Path
1. **Create Order** → Order is created in Order Service (stored in MongoDB)
2. **Place Order** → Saga validates order exists via Order Service
3. **Process Payment** → Payment is processed (synchronous)
4. **Fulfill Order** → Items are shipped (asynchronous)
5. **Send Confirmation** → Email notification sent (asynchronous)
6. **Complete** → Saga finishes successfully

### Payment Failure Path (Retryable)
1. **Create Order** → Order created in Order Service
2. **Place Order** → Order validated
3. **Process Payment** → Payment declined (expired card, etc.)
4. **Update Order** → Mark as PaymentFailed
5. **Send Notification** → Email customer about payment failure
6. **Await Retry** → Saga enters PaymentNotPaid state (awaiting payment retry)
7. **Retry Payment** → Customer retries via Payment API (`POST /api/payments/retry/{orderId}`)
8. **Continue Flow** → On success, saga continues to Fulfilling → SendingConfirmation → Completed

### Out of Stock Path
1. **Create Order** → Order created in Order Service
2. **Place Order** → Order validated
3. **Process Payment** → Payment successful
4. **Fulfill Order** → Items out of stock
5. **Update Order** → Mark as BackOrdered
6. **Refund Payment** → Compensating transaction
7. **Send Notification** → Email customer about backorder & refund
8. **Cancel** → Saga ends with cancelled state

## Getting Started

### Prerequisites
- .NET 10 SDK
- Docker & Docker Compose
- MongoDB (or use Docker)

### Local Development

1. **Start MongoDB**:
```bash
docker run -d -p 27017:27017 --name mongodb mongo:7.0
```

2. **Build the solution**:
```bash
dotnet build
```

3. **Run all services** (in separate terminals):
```bash
# Terminal 1 - Orchestrator
cd src/Services/Orchestrator/OpenMind.Orchestrator.Api
dotnet run

# Terminal 2 - Order
cd src/Services/Order/OpenMind.Order.Api
dotnet run --urls=http://localhost:5001

# Terminal 3 - Payment
cd src/Services/Payment/OpenMind.Payment.Api
dotnet run --urls=http://localhost:5002

# Terminal 4 - Fulfillment
cd src/Services/Fulfillment/OpenMind.Fulfillment.Api
dotnet run --urls=http://localhost:5003

# Terminal 5 - Email
cd src/Services/Email/OpenMind.Email.Api
dotnet run --urls=http://localhost:5004
```

### Using Docker Compose
```bash
docker-compose up --build
```

## API Endpoints

### Step 1: Create Order (Order Service - Port 5001)
```bash
POST http://localhost:5001/api/orders
Content-Type: application/json

{
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "items": [
    {
      "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa7",
      "productName": "Laptop",
      "quantity": 1,
      "unitPrice": 999.99
    }
  ],
  "shippingAddress": {
    "street": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zipCode": "10001",
    "country": "USA"
  }
}
```

### Step 2: Place Order / Start Saga (Orchestrator - Port 5000)
```bash
POST http://localhost:5000/api/orders/{orderId}/place
```

### Get Order Status
```bash
GET http://localhost:5000/api/orders/{orderId}/status
```

### Retry Payment
When payment fails, the saga enters `PaymentNotPaid` state. Use this endpoint to retry:
```bash
POST http://localhost:5002/api/payments/retry/{orderId}
```
On success, the saga automatically continues from PaymentNotPaid → Fulfilling → SendingConfirmation → Completed.

### List All Orders
```bash
GET http://localhost:5000/api/orders?page=1&pageSize=10
```

### Health Checks
```bash
GET http://localhost:5000/health   # Orchestrator
GET http://localhost:5001/health   # Order
GET http://localhost:5002/health   # Payment
GET http://localhost:5003/health   # Fulfillment
GET http://localhost:5004/health   # Email
```

## Simulating Different Scenarios

The services include built-in simulation for testing:
- **Payment Service**: 90% success rate (10% random failures)
- **Fulfillment Service**: 85% in-stock rate (15% out-of-stock)
- **Email Service**: 98% delivery rate

Run multiple order requests to see different workflow paths execute.

## Configuration

### MongoDB Connection
Each service has its own database:
```json
{
  "MongoDB": {
    "ConnectionString": "mongodb://localhost:27019",
    "DatabaseName": "ServiceNameDb"
  }
}
```

## Key Concepts

### Saga State Machine States
- `Initial` - Starting state
- `Validating` - Validating order exists
- `PaymentProcessing` - Processing payment
- `Fulfilling` - Shipping order (async)
- `SendingConfirmation` - Sending success email (async)
- `RefundingPayment` - Compensating transaction
- `SendingPaymentFailedEmail` - Error notification
- `SendingBackorderEmail` - Backorder notification
- `Completed` - Successfully completed
- `Cancelled` - Cancelled (with compensation)
- `Failed` - Unrecoverable failure

### Integration Events
Commands (from Orchestrator):
- `ValidateOrderCommand`
- `ProcessPaymentCommand`
- `FulfillOrderCommand`
- `SendOrderConfirmationEmailCommand`
- `RefundPaymentCommand`

Events (to Orchestrator):
- `OrderValidatedEvent`
- `PaymentCompletedEvent` / `PaymentFailedEvent`
- `OrderShippedEvent` / `FulfillmentFailedEvent`
- `EmailSentEvent`

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
