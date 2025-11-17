# STDT-MidTest
graph TD
    A[Aplikasi Klien (Web/Mobile)] -- "1. Kueri GraphQL Tunggal" --> GQL(GraphQL Gateway)

    subgraph "Backend Microservices (IPC)"
        GQL -- "2. Panggilan IPC (REST)" --> S1(UserService)
        GQL -- "2. Panggilan IPC (gRPC)" --> S2(OrderService)
        GQL -- "2. Panggilan IPC (REST)" --> S3(ProductService)
    end

    subgraph "Respons IPC"
        S1 -- "3. Respons data User" --> GQL
        S2 -- "3. Respons data Order" --> GQL
        S3 -- "3. Respons data Produk" --> GQL
    end

    GQL -- "4. Respons JSON Gabungan" --> A

    sequenceDiagram
    participant Klien as Aplikasi Klien
    participant GQL as GraphQL Gateway
    participant S1 as UserService (REST)
    participant S2 as OrderService (gRPC)
    participant S3 as ProductService (REST)

    Klien ->> GQL: 1. Kueri (data user, order, & produk)
    
    activate GQL
    Note over GQL: Gateway menerjemahkan kueri
    
    GQL ->> S1: 2a. (IPC) Minta data user
    GQL ->> S2: 2b. (IPC) Minta data order
    GQL ->> S3: 2c. (IPC) Minta data produk

    activate S1
    S1 -->> GQL: 3a. Respons data user
    deactivate S1
    
    activate S2
    S2 -->> GQL: 3b. Respons data order
    deactivate S2

    activate S3
    S3 -->> GQL: 3c. Respons data produk
    deactivate S3

    Note over GQL: Gateway menggabungkan semua respons
    GQL -->> Klien: 4. Respons JSON Gabungan
    deactivate GQL
