flowchart TB
  Client["Client (Web/Mobile)"]
  GraphQL["GraphQL Gateway / API"]
  subgraph Backend["Backend / Microservices"]
    A[Service A\n(Profile)]
    B[Service B\n(Billing)]
    C[Service C\n(Activity)]
    D[Service D\n(Inventory)]
  end

  Client -->|1 request (single query)| GraphQL
  GraphQL -->|RPC/HTTP/gRPC| A
  GraphQL -->|RPC/HTTP/gRPC| B
  GraphQL -->|RPC/HTTP/gRPC| C
  GraphQL -->|RPC/HTTP/gRPC| D
  A -->|response| GraphQL
  B -->|response| GraphQL
  C -->|response| GraphQL
  D -->|response| GraphQL
  GraphQL -->|aggregated response| Client
