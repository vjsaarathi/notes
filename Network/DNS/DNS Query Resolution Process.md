---
tags:
  - dns
  - query-resolution
  - recursive-resolution
  - authoritative-servers
---

## Full Resolution Path (Cache Miss Scenario)

When resolving www.example.com with no cached data:

```mermaid
sequenceDiagram
    participant Client
    participant LocalCache
    participant Resolver
    participant RootNS as Root Nameserver
    participant TLD as .com TLD Server
    participant Auth as example.com Auth Server
    
    Client->>LocalCache: Check cache
    LocalCache-->>Client: Miss
    
    Client->>Resolver: Query www.example.com
    Resolver->>RootNS: Who handles .com?
    RootNS-->>Resolver: .com nameservers list
    
    Resolver->>TLD: Who handles example.com?
    TLD-->>Resolver: example.com nameservers
    
    Resolver->>Auth: IP for www.example.com?
    Auth-->>Resolver: 93.184.216.34
    
    Resolver-->>Client: 93.184.216.34
    Client->>LocalCache: Cache result
```

## Cache-Optimized Resolution (Common Case)

Most queries skip the full hierarchy due to caching:

```mermaid
graph TD
    A[DNS Query] --> B{Local Cache Hit?}
    B -->|Yes| C[Return Cached IP]
    B -->|No| D{Router Cache Hit?}
    D -->|Yes| E[Return from Router]
    D -->|No| F{ISP Resolver Cache Hit?}
    F -->|Yes| G[Return from ISP Cache]
    F -->|No| H[Full Recursive Resolution]
```

## Resolution Types

### Recursive Resolution
- Client sends single query to resolver
- Resolver handles entire resolution process
- Client receives final answer
- **Used by**: End-user devices

### Iterative Resolution
- Client handles resolution step-by-step
- Server provides referrals to next nameserver
- Client follows referral chain manually
- **Used by**: DNS servers communicating with each other

## Caching Layers

### Layer 1: Application Cache
- **Location**: Browser, OS resolver
- **TTL**: Respects DNS record TTL
- **Scope**: Single device
- **Performance**: Sub-millisecond response

### Layer 2: Local Network Cache
- **Location**: Router/gateway device
- **Scope**: All devices on local network
- **Benefit**: Shared cache across household/office

### Layer 3: ISP Resolver Cache
- **Location**: ISP's DNS infrastructure
- **Scope**: All ISP customers
- **Benefit**: Massive shared cache for popular domains

### Layer 4: Authoritative Server Cache
- **Location**: DNS servers themselves
- **Purpose**: Cache data about other domains they've queried

## Popular Domain Acceleration

```mermaid
graph LR
    A[First Query to google.com] --> B[Cache Miss - Full Resolution]
    B --> C[Cached at Multiple Layers]
    D[Subsequent Queries] --> E[Cache Hit - Instant Response]
    
    F[Cache Expiration] --> G[Background Refresh]
    G --> H[Updated Cache]
```

**Effect**: Popular domains become faster to resolve due to widespread caching

## TTL Impact on Resolution

Different record types, different caching strategies:
- **A records**: Often 300-3600 seconds TTL
- **NS records**: Often 24-48 hours TTL  
- **Root hints**: Rarely change, cached for days

**Trade-off**: Longer TTL = better performance, slower propagation of changes

---
**Related**: [[DNS Caching Architecture]] | [[DNS Message Structure]] | [[Authoritative vs Recursive Servers]]