# Thesis idea:
## Real-Time Fraud Detection System
This project focuses on identifying suspicious patterns (like multiple rapid transactions from different locations) as they happen.
- The Business Case: Online retailers lose billions to credit card fraud. Detecting fraud after the transaction is costly; blocking it in real-time saves revenue and improves customer trust.
- Technical Implementation:
  - Use Go to simulate a stream of transaction data.
  - Produce messages to a Kafka topic.
  - Build a Go-based Consumer that implements a "Rules Engine" (e.g., checking transaction frequency or velocity).
  - Store flagged transactions in a NoSQL database (like MongoDB or Redis) for a dashboard.
- Thesis Focus: Latency analysis. How fast can the system flag a transaction under high load?

### The Architecture
The system consists of three main components: the Data Simulator, the Kafka Backbone, and the Go Analysis Engine.

#### 1. The Producer (Transaction Simulator)
Since you won't have access to live bank feeds, you will write a Go program that generates realistic JSON transaction data.
- Fields: transaction_id, user_id, amount, timestamp, location, and card_type.
- The "Hook": Your simulator should occasionally inject "Fraudulent Patterns," such as:
  - The "Speed Demon": Three transactions for the same user_id in three different cities within 10 minutes.
  - The "Large Spender": A transaction 500% higher than the user's average.
  
#### 2. The Streaming Layer (Apache Kafka)
Kafka acts as the buffer. Even if your analysis engine slows down, Kafka holds the data so no transactions are lost.
- Topics: Use a raw-transactions topic for incoming data and a flagged-fraud topic for the output of your analysis.

#### 3. The Consumer (Go Analysis Engine)
This is the "brain" of your thesis. You will use Go’s concurrency primitives to process messages.
- Sliding Windows: Use a Go-based "Windowing" logic. For a specific user_id, look at the last 60 minutes of data stored in an in-memory cache (like Redis or a thread-safe Go Map).
- Concurrency: Use multiple goroutines to process different partitions of Kafka topics simultaneously.

#### Thesis Research Questions
A BSc thesis needs to answer a "How" or "Why." You can focus on one of these technical challenges:
1. Latency vs. Accuracy: How does the size of the "time window" (checking the last 5 minutes vs. the last 50 minutes) affect the system's ability to catch fraud in under 100ms?
2. State Management: How do you keep track of a user's previous transactions across multiple Go instances? (This introduces the need for an external state store like Redis).
3. Throughput Benchmarking: At what point does the Go consumer become a bottleneck? You can measure how many thousands of transactions per second (TPS) a single Go instance can handle before latency spikes.

#### Suggested Technology Stack
- Language: Go (using the segmentio/kafka-go or confluent-kafka-go libraries).
- Broker: Apache Kafka (running in Docker).
- State Store: Redis (for fast lookups of "last seen" user location/balance).
- Visualization: A simple Grafana dashboard showing the number of "Safe" vs. "Fraudulent" transactions in real-time.

### The Refined Transaction Model
We distinguish between the **Transaction** details and the **Account** state at that moment.
Fields
- **Transaction Metadata**: `TransactionID`, `UserID`, `Timestamp`.
- **Financials**: * `Amount`: The value in the local currency.
  - `Currency`: The currency of the purchase (e.g., USD, EUR, HUF).
  - `ExchangeRate`: The rate used to convert to the Account's base currency at $T_0$.
- **Account Context**:
  - `BaseCurrency`: The user’s home currency (e.g., a user from Germany has an account in EUR).
  - `AccountBalance`: Remaining balance in `BaseCurrency`.
- **Geographic Data**: IPAddress and Location (Country Code).

Go Implementation idea: The Simulator      
This snippet uses a "Weighted Random" approach to generate normal transactions, with a specific function to inject "Fraudulent" ones.
``` go
package main

import (
	"encoding/json"
	"time"
)

type Transaction struct {
	TransactionID  string    `json:"transaction_id"`
	UserID         string    `json:"user_id"`
	Amount         float64   `json:"amount"`          // Amount in local currency
	Currency       string    `json:"currency"`        // e.g., "USD"
	BaseCurrency   string    `json:"base_currency"`   // User's home currency, e.g., "EUR"
	ExchangeRate   float64   `json:"exchange_rate"`   // Local to Base rate
	Location       string    `json:"location"`        // ISO Country Code
	IPAddress      string    `json:"ip_address"`
	Timestamp      time.Time `json:"timestamp"`
	IsPossibleFraud bool     `json:"-"`               // Internal flag for validation
}

// GenerateNormalTransaction creates a typical user purchase
func GenerateNormalTransaction(userID string) Transaction {
	return Transaction{
		TransactionID: "TXN-" + time.Now().Format("150405"),
		UserID:        userID,
		Amount:        45.50,
		Currency:      "EUR",
		BaseCurrency:  "EUR",
		ExchangeRate:  1.0,
		Location:      "DE",
		IPAddress:     "192.168.1.1",
		Timestamp:     time.Now(),
	}
}
```

### Strategic Fraud Scenarios (Thesis "Edge Cases")
With the new `Currency` and `BaseCurrency` fields, you can now simulate **"Velocity Attacks"** involving cross-border friction:
1. The "Distance-Currency" Mismatch
A user typically transacts in **HUF** in **Budapest**. Suddenly, a transaction appears in **THB** (Thai Baht) from a **Bangkok** IP.
- Logic: If `Currency != BaseCurrency` AND `Location` distance from last `Location` > 1000km within 1 hour.
2. The "Balance Drain" (Micro-transactions)
Attackers often test a card with small amounts in different currencies to see if it’s active.
- Logic: 5 transactions under $2.00$ in 5 different currencies within 30 seconds.
