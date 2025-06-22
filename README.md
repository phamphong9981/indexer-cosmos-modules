# Horoscope v2 - Cosmos Blockchain Indexer

Horoscope v2 là phiên bản tiếp theo của Horoscope, một dịch vụ lập chỉ mục toàn diện cho các blockchain dựa trên Cosmos. Hệ thống thu thập dữ liệu từ blockchain và lập chỉ mục vào PostgreSQL, cung cấp khả năng tìm kiếm và truy vấn dữ liệu hiệu quả thay vì truy vấn trực tiếp từ LCD hoặc RPC.

## 🌟 Tính năng chính

- **Lập chỉ mục toàn diện**: Hỗ trợ tất cả các module Cosmos SDK và các extension
- **Hỗ trợ EVM**: Lập chỉ mục đầy đủ cho các blockchain tương thích EVM như Evmos
- **CosmWasm Integration**: Hỗ trợ smart contract và các chuẩn token (CW20, CW721)
- **Real-time Processing**: Xử lý dữ liệu blockchain theo thời gian thực
- **GraphQL API**: API mạnh mẽ thông qua Hasura
- **Microservices Architecture**: Kiến trúc có thể mở rộng với Moleculer

## 🏗️ Kiến trúc tổng quan

```mermaid
graph TB
    subgraph "Blockchain Networks"
        A[Cosmos SDK Chains]
        B[EVM Compatible Chains]
        C[CosmWasm Chains]
    end
    
    subgraph "Horoscope v2 Core"
        D[Block Crawler]
        E[Transaction Processor]
        F[Event Handler]
        G[Smart Contract Indexer]
    end
    
    subgraph "Specialized Services"
        H[EVM Module]
        I[CosmWasm Module]
        J[IBC Module]
        K[Governance Module]
    end
    
    subgraph "Data Layer"
        L[(PostgreSQL)]
        M[Redis Queue]
    end
    
    subgraph "API Layer"
        N[Hasura GraphQL]
        O[REST API]
    end
    
    A --> D
    B --> D
    C --> D
    
    D --> E
    E --> F
    F --> G
    
    G --> H
    G --> I
    G --> J
    G --> K
    
    H --> L
    I --> L
    J --> L
    K --> L
    
    E --> M
    L --> N
    N --> O
```

## 🚀 Các dịch vụ được hỗ trợ

### Core Services
- [**crawl-block**](./docs/services/crawl-block/README.md): Thu thập block từ network và lưu vào DB
- [**crawl-transaction**](./docs/services/crawl-transaction/README.md): Xử lý giao dịch trong block và decode
- [**crawl-account**](./docs/services/crawl-account/README.md): Quản lý tài khoản và số dư
- [**handle-vote**](./docs/services/handle-vote/README.md): Xử lý tin nhắn vote trong governance

### Governance & Validation
- [**crawl-proposal**](./docs/services/crawl-proposal/README.md): Thu thập proposal và trạng thái
- [**crawl-validator**](./docs/services/crawl-validator/README.md): Quản lý validator và signing info

### Smart Contract & Token
- [**crawl-cosmwasm**](./docs/services/crawl-cosmwasm/README.md): Xử lý CosmWasm smart contract
- [**EVM Module**](./docs/services/evm/README.md): Hỗ trợ đầy đủ EVM (Evmos integration)
  - ERC-20 token indexing
  - ERC-721 NFT processing
  - Smart contract verification
  - Proxy contract support

### Specialized Modules
- [**feegrant**](./docs/services/feegrant/README.md): Xử lý fee grant module
- [**IBC**](./docs/services/ibc/README.md): Inter-Blockchain Communication
- [**crawl-genesis**](./docs/services/crawl-genesis/README.md): Xử lý state từ genesis

## 🌐 Mạng được hỗ trợ

Hiện tại hỗ trợ các network được xây dựng bằng Cosmos SDK v0.45.1 trở lên:

### Mainnet
- **Aura Network**: Layer-1 blockchain với NFT focus
- **Evmos**: EVM-compatible Cosmos chain
- **Sei Network**: High-performance DeFi chain

### Testnet
- **Aura Testnet (Euphoria)**
- **Evmos Testnet**
- **Sei Testnet**

> **Tìm Horoscope v1?** Repository Horoscope v1 đã được archive tại [`Horoscope v1`](https://github.com/aura-nw/Horoscope).

## 🛠️ Công nghệ sử dụng

- **Backend**: Node.js với TypeScript
- **Framework**: [Moleculer.js](https://moleculer.services/) microservices
- **Database**: PostgreSQL với partitioning
- **Queue**: Bull/BullMQ với Redis
- **API**: Hasura GraphQL Engine
- **Blockchain Interaction**: CosmJS, Viem, AuraJS

## 📊 Schema Database

Xem chi tiết schema database [tại đây](./docs/database_schema.md)

### Bảng chính
- **Blockchain Core**: block, transaction, event, message
- **Account Management**: account, account_balance, account_statistics
- **Governance**: proposal, vote, validator
- **Smart Contract**: smart_contract, contract_code
- **EVM Support**: evm_transaction, evm_event, evm_internal_transaction, erc20_contract, erc721_token
- **IBC**: ibc_channel, ibc_connection, ibc_client
- **Feegrant**: feegrant, feegrant_history
- **Cosmwasm**: smart_contract, smart_contract_events, cw20(cw721)_contracts, cw20(cw721)_activities

## 🚀 Cài đặt và Chạy

### Yêu cầu hệ thống
- Node.js 18+
- PostgreSQL 14+
- Redis 6+
- Docker & Docker Compose

### 1. Cài đặt dependencies

Horoscope sử dụng private package [aurajs](https://github.com/aura-nw/aurajs). Tạo Personal Access Token có quyền đọc package:

```bash
# Tạo .npmrc file
@aura-nw:registry=https://npm.pkg.github.com/aura-nw
//npm.pkg.github.com/:_authToken=YOUR_GITHUB_TOKEN
```

### 2. Khởi động infrastructure

```bash
# Khởi động PostgreSQL, Redis, Hasura
docker-compose up -d

# Cài đặt dependencies
npm install
```

### 3. Cấu hình

```bash
# Tạo file environment
cp .env.sample .env

# Cấu hình network
cp network.json.sample network.json

# Cấu hình chain
cp config.json.sample config.json
```

### 4. Migration Database

```bash
# Chạy migration
npm run migrate

# Seed data (optional)
npm run seed
```

### 5. Khởi động services

```bash
# Development mode với hot-reload
npm run dev

# Production mode
npm run build
npm run start
```

## ⚙️ Cấu hình

### Environment Variables (.env)
```bash
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/horoscope

# Redis
REDIS_URL=redis://localhost:6379

# Blockchain RPC
COSMOS_RPC_URL=https://rpc.cosmos.network
EVM_RPC_URL=https://eth.public-rpc.com

# Hasura
HASURA_GRAPHQL_ENDPOINT=http://localhost:8080/v1/graphql
```

### Network Configuration (network.json)
```json
{
  "aura": {
    "chainId": "xstaxy-1",
    "lcd": "https://lcd.aura.network",
    "rpc": "https://rpc.aura.network",
    "database": "aura_mainnet"
  },
  "evmos": {
    "chainId": "evmos_9001-2",
    "lcd": "https://rest.bd.evmos.org:1317",
    "rpc": "https://tendermint.bd.evmos.org:26657",
    "evmRpc": "https://eth.bd.evmos.org:8545",
    "database": "evmos_mainnet"
  }
}
```

### Chain Configuration (config.json)
```json
{
  "chainId": "xstaxy-1",
  "crawlBlock": {
    "blocksPerCall": 100,
    "millisecondCrawl": 5000
  },
  "evm": {
    "enabled": true,
    "crawlEvmBlock": {
      "millisecondCrawl": 3000
    }
  }
}
```

## 🔧 Hasura Setup

### Cài đặt Hasura CLI
```bash
# MacOS
brew install hasura-cli

# Linux
curl -L https://github.com/hasura/graphql-engine/raw/stable/cli/get.sh | bash

# Windows
# Download từ GitHub releases
```

### Migration Metadata
```bash
# Khởi tạo Hasura directory
hasura init hasura

# Cấu hình environment
cp .env.hasura.sample hasura/.env

# Export metadata hiện tại
cd hasura
hasura metadata export

# Apply metadata
hasura metadata apply

# Console
hasura console
```

## 📈 Monitoring & Logging

### Health Check Endpoints
- **Service Health**: `GET /health`
- **Database Status**: `GET /health/db`
- **Queue Status**: `GET /health/queue`

### Metrics
- Block processing rate
- Transaction throughput
- Queue depth
- Error rates

### Logging Levels
- **ERROR**: Critical errors
- **WARN**: Warning conditions
- **INFO**: General information
- **DEBUG**: Detailed debug info

## 🧪 Testing

```bash
# Unit tests
npm run test

# Integration tests
npm run test:integration

# E2E tests
npm run test:e2e

# Coverage
npm run test:coverage
```

## 📚 API Documentation

### GraphQL API
Truy cập Hasura Console tại: `http://localhost:8080`

### Sample Queries

#### Lấy thông tin block mới nhất
```graphql
query LatestBlocks {
  block(limit: 10, order_by: {height: desc}) {
    height
    hash
    time
    tx_count
    proposer_address
  }
}
```

#### Lấy giao dịch ERC-20
```graphql
query ERC20Transfers {
  erc20_activity(limit: 20, order_by: {height: desc}) {
    height
    tx_hash
    from
    to
    amount
    erc20_contract {
      name
      symbol
      decimals
    }
  }
}
```

## 🔒 Security

### Best Practices
- Environment variables cho sensitive data
- Database connection pooling
- Rate limiting trên API endpoints
- Input validation và sanitization
- Regular security updates

### Access Control
- Service-level authentication
- Database role-based access
- API key management
- Network security groups

## 🚀 Deployment

### Docker Deployment
```bash
# Build image
docker build -t horoscope-v2 .

# Run container
docker run -d \
  --name horoscope-v2 \
  -p 3000:3000 \
  -e DATABASE_URL=postgresql://... \
  horoscope-v2
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: horoscope-v2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: horoscope-v2
  template:
    metadata:
      labels:
        app: horoscope-v2
    spec:
      containers:
      - name: horoscope-v2
        image: horoscope-v2:latest
        ports:
        - containerPort: 3000
```

## 🤝 Contributing

### Development Workflow
1. Fork repository
2. Tạo feature branch
3. Implement changes
4. Add tests
5. Submit pull request

### Code Standards
- TypeScript strict mode
- ESLint + Prettier
- Conventional commits
- 100% test coverage cho core services

## 📋 Scripts

- `npm run dev`: Development mode với hot-reload
- `npm run build`: Build production
- `npm run start`: Start production mode
- `npm run lint`: Run ESLint
- `npm run test`: Run tests
- `npm run migrate`: Database migration
- `npm run seed`: Seed database

## 🗺️ Roadmap

### Q1 2024
- [ ] ERC-1155 support
- [ ] Advanced analytics
- [ ] Performance optimizations

### Q2 2024
- [ ] Multi-chain aggregation
- [ ] Real-time subscriptions
- [ ] Advanced querying features

### Q3 2024
- [ ] Machine learning insights
- [ ] Cross-chain analytics
- [ ] Mobile SDK

## 📄 License

MIT License - xem [LICENSE](LICENSE) để biết thêm chi tiết.

---
