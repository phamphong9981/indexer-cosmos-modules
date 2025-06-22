# Crawl Transaction Service

The Crawl Transaction Service is responsible for crawling, processing, and managing blockchain transactions on Cosmos-based networks. It consists of four main services that work together to provide comprehensive transaction tracking, decoding, and analysis.

## Architecture Overview

```mermaid
graph TB
    A[CrawlTxService] --> B[Database]
    C[CoinTransferService] --> B
    D[HandleAuthzTxService] --> B
    E[UploadTxRawLogToS3] --> F[S3 Storage]
    G[RPC/LCD] --> A
    H[Chain Registry] --> A
    I[Event Attributes] --> A
    
    subgraph "Job Queues"
        J[CRAWL_TRANSACTION]
        K[HANDLE_TRANSACTION]
        L[HANDLE_COIN_TRANSFER]
        M[HANDLE_AUTHZ_TX]
        N[UPLOAD_TX_RAW_LOG_TO_S3]
    end
    
    A --> J
    A --> K
    C --> L
    D --> M
    E --> N
```

## Services

### 1. CrawlTxService

**Service Name**: `v1.crawl-transaction`

**Purpose**: Crawls raw transactions from blockchain and decodes them into structured data.

#### Key Features

- **Transaction Crawling**: Fetches raw transaction data from RPC endpoints
- **Message Decoding**: Decodes transaction messages using chain registry
- **Event Processing**: Extracts and processes transaction events
- **Batch Processing**: Efficiently processes multiple transactions
- **Registry Integration**: Uses chain-specific message type registries
- **Checkpoint Management**: Maintains processing checkpoints for consistency

#### Job Queues

| Job Name | Description | Frequency |
|----------|-------------|-----------|
| `CRAWL_TRANSACTION` | Crawl raw transactions from blockchain | Configurable interval |
| `HANDLE_TRANSACTION` | Process and decode transaction data | Configurable interval |

#### Data Flow

```mermaid
sequenceDiagram
    participant A as CrawlTxService
    participant B as Database
    participant C as RPC/LCD
    participant D as Chain Registry
    participant E as Event Attributes

    loop CRAWL_TRANSACTION Job
        A->>B: Get BlockCheckpoint for CrawlTransaction
        activate B
        B-->>A: Return BlockCheckpoint
        deactivate B
        
        alt not found BlockCheckpoint
            A->>A: Set checkpoint = startBlock config
        end
        
        A->>A: endBlock = min(crawlTxCheckpoint + numberOfBlockPerCall, crawlBlockCheckpoint)
        
        A->>C: Get raw transactions from RPC (tx_search)
        activate C
        C->>A: Return list of raw transactions
        deactivate C
        
        alt transactions exist
            loop For each transaction
                A->>D: Decode transaction using registry
                activate D
                D->>D: Find message type in registry
                alt type found
                    D->>D: Decode transaction messages
                else type not found
                    D->>D: Log error for unknown type
                end
                D-->>A: Return decoded transaction
                deactivate D
                
                A->>A: Extract sender and receiver addresses
                A->>A: Map events to message indices
                A->>A: Prepare transaction data for insertion
            end
            
            A->>B: Insert decoded transactions to DB
            A->>B: Update checkpoint = endBlock
        end
    end
    
    loop HANDLE_TRANSACTION Job
        A->>B: Get BlockCheckpoint for HandleTransaction
        activate B
        B-->>A: Return BlockCheckpoint
        deactivate B
        
        A->>B: Get transactions from startBlock to endBlock
        activate B
        B-->>A: Return list of transactions
        deactivate B
        
        loop For each transaction
            A->>A: Process related transaction data
            A->>A: Extract events and attributes
            A->>A: Map message receivers
        end
        
        A->>B: Insert related transaction data
        A->>B: Update checkpoint = endBlock
    end
```

#### Configuration

```json
{
  "crawlTransaction": {
    "key": "numberOfBlockPerCall",
    "millisecondCrawl": 5000
  },
  "handleTransaction": {
    "key": "numberOfBlockPerCall",
    "txsPerCall": 100,
    "millisecondCrawl": 10000
  }
}
```

### 2. CoinTransferService

**Service Name**: `v1.coin-transfer`

**Purpose**: Extracts and processes coin transfer events from transactions.

#### Key Features

- **Transfer Detection**: Identifies coin transfer events in transactions
- **Amount Parsing**: Extracts amount and denomination from transfer events
- **Multi-Send Support**: Handles both single and multi-recipient transfers
- **Authz Integration**: Processes authorized transfer messages
- **Batch Processing**: Efficiently processes multiple transactions

#### Job Queue

| Job Name | Description | Frequency |
|----------|-------------|-----------|
| `HANDLE_COIN_TRANSFER` | Process coin transfer events | Configurable interval |

#### Data Flow

```mermaid
sequenceDiagram
    participant A as CoinTransferService
    participant B as Database
    participant C as Transaction Table
    participant D as Event Table
    participant E as CoinTransfer Table

    loop HANDLE_COIN_TRANSFER Job
        A->>B: Get BlockCheckpoint for HandleCoinTransfer
        activate B
        B-->>A: Return BlockCheckpoint
        deactivate B
        
        A->>C: Get transactions with messages from startBlock to endBlock
        activate C
        C-->>A: Return transactions with messages
        deactivate C
        
        A->>D: Get events for transaction range
        activate D
        D-->>A: Return events with attributes
        deactivate D
        
        loop For each transaction
            loop For each transfer event
                A->>A: Extract sender and recipient addresses
                A->>A: Parse amount and denomination
                A->>A: Handle multi-send scenarios
                A->>A: Create coin transfer record
            end
        end
        
        A->>E: Insert coin transfer records
        A->>B: Update checkpoint = endBlock
    end
```

#### Configuration

```json
{
  "handleCoinTransfer": {
    "key": "numberOfBlockPerCall",
    "millisecondCrawl": 15000
  }
}
```

### 3. HandleAuthzTxService

**Service Name**: `v1.handle-authz-tx`

**Purpose**: Processes authorized transaction messages and extracts nested messages.

#### Key Features

- **Authz Message Processing**: Handles authorized execution messages
- **Nested Message Extraction**: Extracts and processes nested messages
- **Parent-Child Relationships**: Maintains message hierarchy
- **Batch Processing**: Efficiently processes multiple authz messages

#### Job Queue

| Job Name | Description | Frequency |
|----------|-------------|-----------|
| `HANDLE_AUTHZ_TX` | Process authorized transaction messages | Configurable interval |

#### Data Flow

```mermaid
sequenceDiagram
    participant A as HandleAuthzTxService
    participant B as Database
    participant C as TransactionMessage Table
    participant D as Chain Registry

    loop HANDLE_AUTHZ_TX Job
        A->>B: Get BlockCheckpoint for HandleAuthzTx
        activate B
        B-->>A: Return BlockCheckpoint
        deactivate B
        
        alt not found BlockCheckpoint
            A->>A: Set checkpoint = 0
            A->>B: Insert BlockCheckpoint = 0
        end
        
        A->>A: endBlock = checkpoint + numberBlockPerCall
        
        A->>C: Get authz transaction messages from checkpoint to endBlock
        activate C
        C-->>A: Return list of authz messages
        deactivate C
        
        loop For each authz message
            A->>D: Decode nested messages
            activate D
            D-->>A: Return decoded messages
            deactivate D
            
            A->>A: Create child message records
            A->>A: Set parent-child relationships
        end
        
        A->>B: Insert decoded messages to DB
        A->>B: Update block checkpoint = endBlock
    end
```

#### Configuration

```json
{
  "handleAuthzTx": {
    "key": "numberOfBlockPerCall",
    "millisecondCrawl": 20000
  }
}
```

### 4. UploadTxRawLogToS3

**Service Name**: `v1.upload-tx-raw-log-to-s3`

**Purpose**: Uploads transaction raw logs to S3 storage for archival and analysis.

#### Key Features

- **S3 Integration**: Uploads transaction data to AWS S3
- **Organized Storage**: Structures data by chain, height, and transaction hash
- **Batch Upload**: Efficiently uploads multiple transactions
- **Metadata Tracking**: Updates transaction records with S3 links
- **Configurable Overwrite**: Controls whether to overwrite existing files

#### Job Queue

| Job Name | Description | Frequency |
|----------|-------------|-----------|
| `UPLOAD_TX_RAW_LOG_TO_S3` | Upload transaction raw logs to S3 | Configurable interval |

#### Data Flow

```mermaid
sequenceDiagram
    participant A as UploadTxRawLogToS3
    participant B as Database
    participant C as Transaction Table
    participant D as S3 Storage

    loop UPLOAD_TX_RAW_LOG_TO_S3 Job
        A->>B: Get BlockCheckpoint for UploadTxRawLogToS3
        activate B
        B-->>A: Return BlockCheckpoint
        deactivate B
        
        A->>C: Get transactions from startBlock to endBlock
        activate C
        C-->>A: Return list of transactions
        deactivate C
        
        loop For each transaction
            A->>D: Upload transaction data to S3
            activate D
            D-->>A: Return S3 upload result
            deactivate D
            
            A->>A: Generate S3 link metadata
        end
        
        A->>B: Update transaction records with S3 links
        A->>B: Update checkpoint = endBlock
    end
```

#### Configuration

```json
{
  "uploadTransactionRawLogToS3": {
    "key": "numberOfBlockPerCall",
    "overwriteS3IfFound": false,
    "returnIfFound": true,
    "millisecondCrawl": 30000
  }
}
```

## Database Schema

### Transaction Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `height` | INTEGER | Block height |
| `hash` | VARCHAR | Transaction hash |
| `codespace` | VARCHAR | Transaction codespace |
| `code` | INTEGER | Transaction result code |
| `gas_used` | VARCHAR | Gas used by transaction |
| `gas_wanted` | VARCHAR | Gas wanted by transaction |
| `gas_limit` | VARCHAR | Gas limit |
| `fee` | VARCHAR | Transaction fee |
| `memo` | TEXT | Transaction memo |
| `index` | INTEGER | Transaction index in block |
| `timestamp` | TIMESTAMP | Transaction timestamp |
| `data` | JSONB | Raw transaction data |
| `created_at` | TIMESTAMP | Record creation time |
| `updated_at` | TIMESTAMP | Record update time |

### TransactionMessage Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `tx_id` | INTEGER | Reference to transaction |
| `index` | INTEGER | Message index in transaction |
| `type` | VARCHAR | Message type |
| `sender` | VARCHAR | Message sender address |
| `content` | JSONB | Decoded message content |
| `parent_id` | INTEGER | Reference to parent message (for authz) |
| `created_at` | TIMESTAMP | Record creation time |
| `updated_at` | TIMESTAMP | Record update time |

### CoinTransfer Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `block_height` | INTEGER | Block height |
| `tx_id` | INTEGER | Reference to transaction |
| `tx_msg_id` | INTEGER | Reference to transaction message |
| `from` | VARCHAR | Sender address |
| `to` | VARCHAR | Recipient address |
| `amount` | VARCHAR | Transfer amount |
| `denom` | VARCHAR | Token denomination |
| `timestamp` | TIMESTAMP | Transfer timestamp |
| `created_at` | TIMESTAMP | Record creation time |
| `updated_at` | TIMESTAMP | Record update time |

### Event Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `tx_id` | INTEGER | Reference to transaction |
| `block_height` | INTEGER | Block height |
| `tx_msg_index` | INTEGER | Message index |
| `type` | VARCHAR | Event type |
| `source` | VARCHAR | Event source |
| `created_at` | TIMESTAMP | Record creation time |
| `updated_at` | TIMESTAMP | Record update time |

### EventAttribute Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `event_id` | INTEGER | Reference to event |
| `key` | VARCHAR | Attribute key |
| `value` | TEXT | Attribute value |
| `tx_id` | INTEGER | Reference to transaction |
| `created_at` | TIMESTAMP | Record creation time |
| `updated_at` | TIMESTAMP | Record update time |

## Message Type Registry

The service uses chain-specific registries to decode transaction messages. Each network has its own registry file:

### Aura Network Registry (`aura-network.json`)

```json
[
  "/cosmos.gov.v1beta1.MsgSubmitProposal",
  "/cosmos.upgrade.v1beta1.SoftwareUpgradeProposal",
  "/cosmos.distribution.v1beta1.CommunityPoolSpendProposal",
  "/cosmos.params.v1beta1.ParameterChangeProposal",
  "/ibc.core.client.v1.UpgradeProposal",
  "/cosmos.feegrant.v1beta1.BasicAllowance",
  "/cosmos.vesting.v1beta1.MsgCreatePeriodicVestingAccount",
  "/ibc.lightclients.tendermint.v1.Header",
  "/cosmos.slashing.v1beta1.MsgUnjail",
  "/aura.smartaccount.v1beta1.MsgActivateAccount",
  "/ethermint.evm.v1.MsgEthereumTx",
  "/evmos.erc20.v1.MsgConvertCoin"
]
```

### Supported Message Types

1. **Governance Messages**: Proposal submission, parameter changes
2. **Bank Messages**: Coin transfers, multi-send operations
3. **Staking Messages**: Delegation, unbonding, redelegation
4. **IBC Messages**: Cross-chain transfers, client updates
5. **Smart Contract Messages**: CosmWasm instantiate, execute
6. **Authz Messages**: Authorized execution
7. **Custom Messages**: Chain-specific message types

## Job Queue Dependencies

The transaction services have specific dependencies on other services:

1. **CrawlTxService** depends on:
   - `CRAWL_BLOCK` - For block checkpoint synchronization

2. **CoinTransferService** depends on:
   - `HANDLE_TRANSACTION` - For transaction processing completion

3. **HandleAuthzTxService** depends on:
   - `HANDLE_TRANSACTION` - For transaction message processing

4. **UploadTxRawLogToS3** depends on:
   - `HANDLE_TRANSACTION` - For transaction processing completion

## Error Handling

### Common Error Scenarios

1. **Unknown Message Types**: When message type is not in registry
   - Logged as error with message type
   - Transaction processing continues
   - Raw data preserved in database

2. **RPC/LCD Errors**: Network or service unavailability
   - Retry mechanism with exponential backoff
   - Job failure handling with retry limits

3. **Decoding Errors**: Invalid transaction data
   - Error logged with transaction hash
   - Raw data preserved for analysis
   - Processing continues with other transactions

4. **S3 Upload Errors**: Storage service issues
   - Individual transaction upload failures logged
   - Batch processing continues
   - Retry mechanism for failed uploads

### Error Recovery

- Jobs are configured with retry mechanisms
- Failed jobs are preserved for analysis (up to 3 failures)
- Checkpoint system ensures no data loss on restart
- Raw transaction data preserved even if decoding fails

## Performance Considerations

### Optimization Strategies

1. **Batch Processing**: Multiple transactions processed in single operations
2. **Checkpoint System**: Prevents reprocessing of already handled blocks
3. **Indexed Queries**: Database indexes on frequently queried columns
4. **Efficient Filtering**: Smart queries to only process relevant data
5. **Parallel Processing**: Concurrent processing of independent operations

### Scalability

- Services can be horizontally scaled
- Job queues support multiple workers
- Database partitioning for large datasets
- S3 storage for archival data

## Monitoring and Metrics

### Key Metrics to Monitor

1. **Transaction Processing Rate**: Number of transactions processed per minute
2. **Message Decoding Success Rate**: Percentage of successfully decoded messages
3. **Job Queue Health**: Queue length and processing times
4. **Error Rates**: Failed jobs and error types
5. **Database Performance**: Query execution times
6. **S3 Upload Performance**: Upload success rate and latency

### Logging

Services provide detailed logging for:
- Transaction discovery and processing
- Message decoding operations
- Event extraction and processing
- Error conditions and recovery actions
- Performance metrics and bottlenecks

## API Integration

### GraphQL Queries

```graphql
# Get transactions with messages
query GetTransactions($limit: Int!) {
  transaction(limit: $limit, order_by: {height: desc}) {
    id
    hash
    height
    code
    gas_used
    gas_wanted
    fee
    timestamp
    transaction_messages {
      id
      index
      type
      sender
      content
    }
    events {
      id
      type
      tx_msg_index
      event_attributes {
        key
        value
      }
    }
  }
}

# Get coin transfers
query GetCoinTransfers($fromAddress: String!) {
  coin_transfer(where: {from: {_eq: $fromAddress}}) {
    id
    block_height
    from
    to
    amount
    denom
    timestamp
    transaction {
      hash
      height
    }
  }
}

# Get transactions by message type
query GetTransactionsByType($messageType: String!) {
  transaction_message(where: {type: {_eq: $messageType}}) {
    id
    type
    sender
    content
    transaction {
      hash
      height
      timestamp
    }
  }
}
```

### REST API Endpoints

The transaction data is accessible through Hasura GraphQL API with the following permissions:
- **Public**: Read access to transaction data
- **Internal Service**: Read access with aggregation capabilities

## Configuration Examples

### Development Configuration

```json
{
  "crawlTransaction": {
    "key": 50,
    "millisecondCrawl": 10000
  },
  "handleTransaction": {
    "key": 50,
    "txsPerCall": 50,
    "millisecondCrawl": 15000
  },
  "handleCoinTransfer": {
    "key": 50,
    "millisecondCrawl": 20000
  },
  "handleAuthzTx": {
    "key": 50,
    "millisecondCrawl": 25000
  },
  "uploadTransactionRawLogToS3": {
    "key": 50,
    "overwriteS3IfFound": false,
    "returnIfFound": true,
    "millisecondCrawl": 30000
  }
}
```

### Production Configuration

```json
{
  "crawlTransaction": {
    "key": 100,
    "millisecondCrawl": 5000
  },
  "handleTransaction": {
    "key": 100,
    "txsPerCall": 100,
    "millisecondCrawl": 10000
  },
  "handleCoinTransfer": {
    "key": 100,
    "millisecondCrawl": 15000
  },
  "handleAuthzTx": {
    "key": 100,
    "millisecondCrawl": 20000
  },
  "uploadTransactionRawLogToS3": {
    "key": 100,
    "overwriteS3IfFound": false,
    "returnIfFound": true,
    "millisecondCrawl": 30000
  }
}
```

## Dependencies

### External Dependencies

- **RPC/LCD Client**: For blockchain data access
- **Cosmos SDK**: For transaction and message data structures
- **Database**: PostgreSQL for data storage
- **Job Queue**: Bull/BullMQ for task management
- **S3 Storage**: AWS S3 for raw log archival

### Internal Dependencies

- **BlockCheckpoint**: For synchronization
- **ChainRegistry**: For message type decoding
- **EventAttribute**: For event processing
- **Block**: For checkpoint synchronization

## Troubleshooting

### Common Issues

1. **Transactions Not Being Crawled**
   - Check if `CRAWL_BLOCK` job is running
   - Verify block checkpoint synchronization
   - Check RPC endpoint connectivity

2. **Message Decoding Failures**
   - Verify message type registry is up to date
   - Check for new message types in recent transactions
   - Review error logs for unknown message types

3. **Coin Transfers Not Processing**
   - Ensure `HANDLE_TRANSACTION` job is processing transactions
   - Check transaction message processing
   - Verify event attribute extraction

4. **S3 Upload Failures**
   - Check AWS credentials and permissions
   - Verify S3 bucket configuration
   - Check network connectivity to S3

### Debug Commands

```bash
# Check job queue status
curl -X GET "http://localhost:3000/api/queues"

# View job logs
curl -X GET "http://localhost:3000/api/jobs/{jobId}/logs"

# Check database connections
psql -h localhost -U username -d database -c "SELECT * FROM transaction ORDER BY created_at DESC LIMIT 5;"

# Check S3 upload status
aws s3 ls s3://bucket-name/rawlog/chain-name/chain-id/transaction/ --recursive
```

## Future Enhancements

1. **Real-time Processing**: WebSocket support for transaction updates
2. **Advanced Analytics**: Transaction pattern analysis
3. **Multi-chain Support**: Unified transaction tracking across chains
4. **Enhanced Monitoring**: Prometheus metrics integration
5. **Caching Layer**: Redis caching for frequently accessed data
6. **Streaming Processing**: Kafka integration for real-time data flow 