# Technical Proposal: Polar Web

## Lightning Network Visualization & Management Platform

---

**Document Version:** 1.1  
**Date:** January 2026  
**Status:** Draft for Review  
**Author:** Jemimah Nagasha, Solomon Eze
**Reviewers:** Jamaljsr

---

## Executive Summary

This proposal presents **Polar Web**, a web-based visualization and management interface for existing Lightning Network nodes. Unlike the existing Polar Desktop application which creates and manages local development nodes, Polar Web connects to **externally managed nodes** deployed in production environments (Kubernetes, bare metal, cloud).

### The Ask

We are requesting approval to proceed with an **16-24 week proof of concept** that will deliver:

1. A functional web application connecting to **Bitcoin Core**, LND, Core Lightning, and Eclair nodes
2. Real-time network topology visualization showing both Bitcoin and Lightning layers
3. Basic channel and payment operations
4. Kubernetes-native deployment

### Why Now?

- The original feature request ([GitHub Issue #535](https://github.com/jamaljsr/polar/issues/535)) has remained open since 2022
- Organizations running production Lightning infrastructure lack unified visualization tooling
- The Lightning Network ecosystem has matured, with multiple implementations now production-ready
- No existing open-source solution fills this gap

---

## Problem Statement

### Current State

**Polar Desktop** is a widely-used Electron application for Lightning Network development. It:
- Creates local networks using Docker Compose
- Manages the full lifecycle of nodes (create, start, stop, destroy)
- Provides excellent visualization of network topology
- Is limited to **single-user, local development** use cases

### The Gap

Organizations operating Lightning nodes in production face these challenges:

| Challenge | Impact |
|-----------|--------|
| **No unified dashboard** | Teams use separate CLI tools for each node |
| **No visual topology** | Network structure exists only in documentation |
| **Implementation silos** | LND, CLN, Eclair, and Bitcoin Core have different management interfaces |
| **No Bitcoin ↔ Lightning visibility** | Bitcoin layer health not visible alongside Lightning |
| **No team collaboration** | Each engineer connects individually to nodes |
| **Manual operations** | Channel management requires direct API/CLI access |

### User Story

> *"As a Lightning infrastructure engineer at a company running 10+ nodes across LND and Core Lightning, I need a single web interface where my team can visualize our network topology, monitor node health, and perform channel operations—without needing direct server access or implementation-specific CLI knowledge."*

---

## Proposed Solution

### Polar Web: A Connection-Based Model

Polar Web operates on a fundamentally different architecture than Polar Desktop:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│                    POLAR WEB ARCHITECTURAL PRINCIPLE                    │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │    "A window into your Lightning Network, not a factory for it" │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  POLAR WEB DOES:                    POLAR WEB DOES NOT:                │
│  ─────────────────                  ────────────────────               │
│  ✓ Connect to existing nodes        ✗ Create new nodes                 │
│  ✓ Connect to Bitcoin Core nodes    ✗ Manage node lifecycle            │
│  ✓ Visualize network topology       ✗ Provision infrastructure         │
│  ✓ Perform channel operations       ✗ Handle node restarts             │
│  ✓ Monitor node & chain health      ✗ Manage Docker/K8s resources      │
│  ✓ Support multiple implementations                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Differentiators

| Aspect | Polar Desktop | Polar Web |
|--------|---------------|-----------|
| **Deployment** | Local Electron app | Kubernetes-native web app |
| **Users** | Single developer | Teams with shared access |
| **Nodes** | Creates via Docker | Connects to existing |
| **Use Case** | Development/Testing | Development + Production visibility |
| **Node Location** | Local machine | Anywhere (K8s, cloud, on-prem) |

---

## Comparison with Alternatives

| Solution | Pros | Cons |
|----------|------|------|
| **RTL (Ride The Lightning)** | Mature, feature-rich | Single-node focus, no multi-impl support |
| **ThunderHub** | Good UX, LND focus | LND only, limited team features |
| **LNDHub** | Account management | Different use case (custodial) |
| **Polar Desktop** | Excellent visualization | Local dev only, Electron-based |
| **Polar Web (Proposed)** | Multi-impl, team-focused, K8s-native | New project, requires development |

---

## Technical Architecture

### High-Level System Design

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            POLAR WEB SYSTEM                                  │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                         FRONTEND (React)                               │  │
│  │                                                                        │  │
│  │   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐          │  │
│  │   │ Network Graph  │  │  Node Details  │  │ Import Wizard  │          │  │
│  │   │ (React Flow)   │  │    Panel       │  │    (Forms)     │          │  │
│  │   └────────────────┘  └────────────────┘  └────────────────┘          │  │
│  │                                                                        │  │
│  │                    ┌──────────────────────┐                            │  │
│  │                    │   Zustand Stores     │                            │  │
│  │                    │   (Client State)     │                            │  │
│  │                    └──────────┬───────────┘                            │  │
│  │                               │                                        │  │
│  └───────────────────────────────┼────────────────────────────────────────┘  │
│                                  │ HTTP/WebSocket                            │
│  ┌───────────────────────────────┼────────────────────────────────────────┐  │
│  │                        BACKEND (Go)                                    │  │
│  │                               │                                        │  │
│  │   ┌────────────────┐  ┌──────┴───────┐  ┌────────────────┐            │  │
│  │   │   REST API     │  │  WebSocket   │  │  Config Parser │            │  │
│  │   │   Handlers     │  │     Hub      │  │  (YAML/JSON)   │            │  │
│  │   └───────┬────────┘  └──────┬───────┘  └────────────────┘            │  │
│  │           │                  │                                         │  │
│  │   ┌───────┴──────────────────┴──────────────────────────────────────┐ │  │
│  │   │                   CONNECTION MANAGER                             │ │  │
│  │   │          (Pool of active node connections)                       │ │  │
│  │   └──────────────────────────┬───────────────────────────────────────┘ │  │
│  │                              │                                         │  │
│  │   ┌──────────────────────────┼───────────────────────────────────────┐ │  │
│  │   │                   ADAPTER LAYER                                   │ │  │
│  │   │ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────────┐           │ │  │
│  │   │ │   LND    │ │   CLN    │ │  Eclair  │ │   BITCOIN   │           │ │  │
│  │   │ │ Adapter  │ │ Adapter  │ │ Adapter  │ │   Adapter   │           │ │  │
│  │   │ │ (gRPC)   │ │ (gRPC)   │ │  (REST)  │ │  (RPC/REST) │           │ │  │
│  │   │ └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────┬──────┘           │ │  │
│  │   └──────┼────────────┼────────────┼──────────────┼──────────────────┘ │  │
│  │          │            │            │              │                    │  │
│  └──────────┼────────────┼────────────┼──────────────┼────────────────────┘  │
│             │            │            │              │                       │
│  ┌──────────┼────────────┼────────────┼──────────────┼────────────────────┐  │
│  │          ▼            ▼            ▼              │                    │  │
│  │ ┌────────────┐ ┌────────────┐ ┌────────────┐     │                    │  │
│  │ │  LND Node  │ │  CLN Node  │ │Eclair Node │     │ LIGHTNING LAYER    │  │
│  │ │ (External) │ │ (External) │ │ (External) │     │                    │  │
│  │ └──────┬─────┘ └──────┬─────┘ └──────┬─────┘     │                    │  │
│  │        │              │              │            │                    │  │
│  │        │   Each Lightning node       │            │                    │  │
│  │        │   connects to Bitcoin       │            │                    │  │
│  │        │              │              │            │                    │  │
│  │        └──────────────┼──────────────┘            │                    │  │
│  │                       ▼                           ▼                    │  │
│  │          ┌─────────────────────────────────────────────┐              │  │
│  │          │               BITCOIN CORE                  │              │  │
│  │          │           (RPC port 8332)                   │              │  │
│  │          │                                             │              │  │
│  │          │  • Block height & sync status               │ BITCOIN     │  │
│  │          │  • Mempool info                             │ LAYER       │  │
│  │          │  • On-chain wallet (if enabled)             │              │  │
│  │          │  • UTXO queries                             │              │  │
│  │          └─────────────────────────────────────────────┘              │  │
│  │                                                                       │  │
│  │                     EXTERNAL NODES (User Managed)                     │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Technology Stack

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Backend** | Go 1.22+ | Native gRPC support (LND uses gRPC), excellent K8s ecosystem, low memory footprint, single binary deployment |
| **Frontend** | React 18 + TypeScript | Type safety, component model matches Polar Desktop patterns, strong ecosystem |
| **State Management** | Zustand | Simpler than Redux, excellent TypeScript support, performant |
| **Visualization** | React Flow | Purpose-built for node-and-edge graphs, highly customizable |
| **Database** | SQLite | Embedded, zero-config, portable (upgradeable to PostgreSQL) |
| **Real-time** | WebSocket | Native browser support, bidirectional, lower overhead than polling |
| **Deployment** | Docker + Kubernetes | Target environment for production users |
| **Bitcoin RPC** | btcd/rpcclient or net/http | Standard JSON-RPC for Bitcoin Core communication |

### Core Design Pattern: Adapter Interface

The most critical architectural decision is the **Adapter Pattern** that abstracts implementation differences:

```go
// LightningAdapter defines the interface for ANY Lightning implementation
type LightningAdapter interface {
    // Connection lifecycle
    Connect(ctx context.Context) error
    Disconnect() error
    Ping(ctx context.Context) error
    
    // Node information
    GetInfo(ctx context.Context) (*NodeInfo, error)
    
    // Channel operations
    ListChannels(ctx context.Context) ([]*Channel, error)
    OpenChannel(ctx context.Context, req *OpenChannelRequest) (*ChannelPoint, error)
    CloseChannel(ctx context.Context, req *CloseChannelRequest) error
    
    // Payment operations
    CreateInvoice(ctx context.Context, req *CreateInvoiceRequest) (*Invoice, error)
    SendPayment(ctx context.Context, req *SendPaymentRequest) (*PaymentResult, error)
    
    // Real-time subscriptions
    SubscribeChannelEvents(ctx context.Context) (<-chan ChannelEvent, error)
    SubscribeInvoices(ctx context.Context) (<-chan *Invoice, error)
}

// BitcoinAdapter defines the interface for Bitcoin node communication
type BitcoinAdapter interface {
    // Connection lifecycle
    Connect(ctx context.Context) error
    Disconnect() error
    Ping(ctx context.Context) error
    
    // Chain information
    GetBlockchainInfo(ctx context.Context) (*BlockchainInfo, error)
    GetBlockCount(ctx context.Context) (int64, error)
    GetBestBlockHash(ctx context.Context) (string, error)
    
    // Mempool
    GetMempoolInfo(ctx context.Context) (*MempoolInfo, error)
    
    // Fee estimation
    EstimateSmartFee(ctx context.Context, confTarget int) (*FeeEstimate, error)
    
    // Network
    GetNetworkInfo(ctx context.Context) (*NetworkInfo, error)
    GetPeerInfo(ctx context.Context) ([]*PeerInfo, error)
}
```

This interface ensures:
- **Uniform API** regardless of underlying implementation
- **Isolated complexity** within each adapter
- **Easy extensibility** for new implementations
- **Testability** via mock adapters
- **Bitcoin visibility** alongside Lightning nodes

---

## Node Import Methods

A key feature is **flexible node import**, supporting multiple workflows:

### 1. YAML Configuration (GitOps)

```yaml
version: "1.0"
network:
  name: "Production Lightning"

# Bitcoin layer (required for chain visibility)
bitcoin_nodes:
  - name: "bitcoin-core-1"
    implementation: "bitcoind"
    connection:
      host: "bitcoind.bitcoin.svc"
      rpc_port: 8332
    auth:
      rpc_user: "polarweb"
      rpc_password_secret:
        name: "bitcoin-creds"
        key: "rpc_password"

# Lightning layer
nodes:
  - name: "routing-node-1"
    implementation: "lnd"
    bitcoin_node: "bitcoin-core-1"  # Link to Bitcoin node
    connection:
      host: "lnd-routing.lightning.svc"
      grpc_port: 10009
    auth:
      tls_cert_secret:
        name: "lnd-routing-creds"
        key: "tls.cert"
      macaroon_secret:
        name: "lnd-routing-creds"
        key: "admin.macaroon"
```

### 2. UI Wizard (Manual)

Step-by-step form:
1. Select implementation (LND/CLN/Eclair)
2. Enter connection details (host, port)
3. Upload credentials (TLS cert, macaroon)
4. Test connection
5. Import

### 3. Connection URI (Quick Import)

```
lndconnect://lnd.example.com:10009?cert=base64...&macaroon=base64...
```

### 4. Kubernetes Secret Reference

```yaml
auth:
  tls_cert_secret:
    name: "my-node-credentials"
    key: "tls.cert"
```

---

## Proof of Concept Scope

### In Scope (18-24 Week PoC)

| Feature | Priority | Description |
|---------|----------|-------------|
| **Bitcoin Adapter** | P0 | RPC adapter for Bitcoin Core nodes (chain health, sync status) |
| **LND Adapter** | P0 | Full gRPC adapter for LND nodes |
| **Node Import (YAML)** | P0 | Import nodes via configuration file |
| **Node Import (UI)** | P0 | Manual import wizard |
| **Network Topology Graph** | P0 | Interactive visualization with React Flow (Bitcoin + Lightning layers) |
| **Node Status Display** | P0 | Show online/offline, basic info, chain sync status |
| **Channel Listing** | P0 | Display channels with balances |
| **Real-time Updates** | P1 | WebSocket-based status updates |
| **CLN Adapter** | P1 | Core Lightning support |
| **Open Channel** | P1 | Open new channels via UI |
| **Close Channel** | P1 | Close channels (cooperative) |
| **Create Invoice** | P2 | Generate Lightning invoices |
| **Pay Invoice** | P2 | Send payments |
| **Eclair Adapter** | P2 | Eclair implementation support |
| **Docker Deployment** | P0 | Docker Compose for development |
| **Basic Auth** | P1 | Simple authentication |

### Out of Scope (Future Phases)

- Multi-tenancy / organization support
- Advanced RBAC
- Automated channel rebalancing
- Liquidity management suggestions
- Mobile responsive design
- Historical analytics / reporting
- Alerting and notifications
- Backup/restore functionality

---

## Proposed Implementation Phases

### Phase 0: Foundation & Learning (Weeks 1-4)

**Goal:** Project scaffolding + team Go ramp-up

- [ ] Go fundamentals for team
- [ ] Project scaffolding (Go backend, React frontend)
- [ ] Basic REST API with Chi router
- [ ] SQLite database schema
- [ ] Docker Compose setup
- [ ] CI/CD pipeline

**Exit Criteria:**
- `go build ./...` succeeds
- Docker Compose starts both services

### Phase 1: Bitcoin Core Integration (Weeks 5-7)

**Goal:** Working Bitcoin adapter as simpler first backend implementation

- [ ] Bitcoin RPC adapter implementation (simpler than gRPC, good learning project)
- [ ] `GetBlockchainInfo`, `GetBlockCount`, `GetNetworkInfo` methods
- [ ] Bitcoin node import via YAML
- [ ] Basic API endpoints for Bitcoin node status
- [ ] React component for Bitcoin node status card

**Exit Criteria:**
- Can connect to Bitcoin Core and retrieve blockchain info
- Bitcoin node status displays in UI

### Phase 2: LND Integration (Weeks 8-11)

**Goal:** Working LND gRPC adapter

- [ ] LND gRPC adapter implementation 
- [ ] TLS + macaroon authentication
- [ ] `GetInfo`, `ListChannels`, `ListPeers` methods
- [ ] LND node import via UI wizard
- [ ] Node status persistence in SQLite

**Exit Criteria:**
- Can connect to LND node and retrieve info
- Node persists in database
- Seamless authentication flow

### Phase 3: Visualization (Weeks 12-15)

**Goal:** Interactive network graph showing both layers

- [ ] React Flow integration
- [ ] Custom node components (Bitcoin + Lightning nodes)
- [ ] Custom edge components (channels with balances)
- [ ] Auto-layout algorithm (dagre)
- [ ] Node selection and detail panel
- [ ] Bitcoin ↔ Lightning relationship visualization

**Exit Criteria:**
- Graph displays Bitcoin node(s) + Lightning nodes
- Channels shown as edges with capacity
- Can see which Lightning nodes connect to which Bitcoin node

### Phase 4: Real-time & Additional Adapters (Weeks 16-19)

**Goal:** Live updates and multi-implementation support

- [ ] WebSocket hub implementation
- [ ] Channel event subscriptions
- [ ] CLN adapter implementation
- [ ] Eclair adapter implementation
- [ ] Status updates without refresh

**Exit Criteria:**
- Node going offline updates UI within 5 seconds
- LND, CLN, and Eclair nodes all work
- Mixed network displays correctly

### Phase 5: Operations & Polish (Weeks 20-24)

**Goal:** Channel operations, payments, production readiness

- [ ] Open channel modal
- [ ] Close channel functionality
- [ ] Create invoice flow
- [ ] Pay invoice flow
- [ ] Basic authentication (JWT)
- [ ] Connection URI import
- [ ] Helm chart
- [ ] Documentation

**Exit Criteria:**
- Can open/close channels via UI
- Can create and pay invoices
- Helm install works on fresh cluster
- Security review completed

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| **gRPC complexity** | Medium | High | Use official LND Go libraries; extensive testing; defer to Phase 2 after Go familiarity |
| **Implementation differences** | High | Medium | Strong adapter abstraction; focus on common operations first |
| **WebSocket scaling** | Low | Medium | Design for single replica initially; Redis pub/sub for scale |
| **Credential security** | Medium | Critical | Encryption at rest; never log credentials; security review |
| **LND/Bitcoin API changes** | Low | Medium | Pin to specific versions; monitor release notes |
| **Performance with many nodes** | Medium | Medium | Lazy loading; virtual scrolling; connection pooling |
| **Bitcoin RPC authentication** | Low | Low | Well-documented; standard HTTP basic auth + cookie auth |

---

## Security Considerations

### Credential Handling

1. **Storage**: All credentials (macaroons, TLS certs, passwords) encrypted at rest using AES-256-GCM
2. **Transmission**: TLS required for all node connections
3. **Logging**: Credentials NEVER appear in logs
4. **API**: Credentials NEVER returned in API responses

### Authentication

- JWT-based authentication for web UI
- API key support for programmatic access
- Session management with secure cookies

### Network Security

- Kubernetes NetworkPolicy for pod isolation
- mTLS for Lightning node connections
- Rate limiting on all API endpoints

---

## Success Target Metrics

### Proof of Concept Success Criteria

| Metric | Target |
|--------|--------|
| **Connect to Bitcoin Core** | < 10 seconds from config to connected |
| **Connect to LND** | < 30 seconds from config to connected |
| **Import 5 nodes** | < 2 minutes via YAML |
| **Graph render** | < 1 second for 20 nodes |
| **Status update latency** | < 2 seconds from event to UI |
| **Open channel** | Complete via UI in < 1 minute |
| **Docker image size** | < 50MB (backend) |
| **Test coverage** | > 70% |

### User Acceptance Criteria

1. Engineer can import existing Bitcoin Core node 
2. Engineer can import existing LND node 
3. Team can view network topology in browser (Bitcoin + Lightning layers)
4. Can identify node status (online/offline) and chain sync status at a glance
5. Can open a channel without using CLI
6. Works in Kubernetes environment

---

## Current Team Resources

### Team

| Role | Allocation | Current Skills | Development Plan |
|------|------------|----------------|------------------|
| Jemimah Nagasha | 1 FTE | Strong frontend (React/TS) | Go ramp-up weeks 1-3   |
| Solomon Eze  | 1 FTE | Strong frontend / mobile (React/TS) | Go ramp-up weeks 1-3 |
| Jamaljsr (tentative)  | Advisory | ALl round expertise | Product overview and architecture guidance |


### Infrastructure (Development)

- Development Kubernetes cluster or Docker Desktop
- Bitcoin Core node (regtest/testnet)
- Test LND/CLN/Eclair nodes (regtest)
- GitHub repository
- CI/CD (GitHub Actions)

### Timeline

**Total Duration:** 16-24 weeks  
**Start Date:** TBD (pending approval)  
**End Date:** Start + 24 weeks (with 18-week optimistic target)

| Milestone | Week | Deliverable |
|-----------|------|-------------|
| Phase 0 Complete | 3 | Scaffolding + team Go-ready |
| Phase 1 Complete | 7 | Bitcoin adapter working |
| Phase 2 Complete | 11 | LND adapter working |
| Phase 3 Complete | 15 | Visualization complete |
| Phase 4 Complete | 19 | All adapters + real-time |
| Phase 5 Complete | 24 | Full PoC ready |

---

## Open Questions for Review

1. **Authentication Provider**: Should we integrate with existing SSO/OIDC, or is standalone JWT sufficient for PoC?

2. **Multi-tenancy**: Is single-tenant deployment acceptable for PoC, with multi-tenancy as future enhancement?

3. **Observability**: What level of metrics/logging is required for PoC vs. production?

4. **External Dependencies**: Any concerns with the proposed technology stack (Go, React, SQLite)?

5. **Security Review**: At what phase should we conduct formal security review?

6. **Bitcoin Node Scope**: Should the Bitcoin adapter include wallet operations (generate address, send BTC), or only chain/mempool visibility?

7. **Timeline Flexibility**: Given team skill development needs, is there flexibility to extend beyond 24 weeks if necessary?

---

