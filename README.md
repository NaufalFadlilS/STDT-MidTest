1. Teorema CAP dan BASE + Keterkaitan + Contoh
Teorema CAP

Teorema CAP (Consistency, Availability, Partition Tolerance) menyatakan bahwa pada sistem terdistribusi hanya dua dari tiga properti berikut yang dapat dijamin secara bersamaan:

• Consistency (C)

Semua node memiliki data yang sama pada waktu yang sama (strong consistency).

• Availability (A)

Setiap permintaan selalu mendapatkan respons, meskipun respons tidak selalu konsisten.

• Partition Tolerance (P)

Sistem tetap berfungsi meskipun terjadi gangguan atau putusnya jaringan antar node.

Karena partition selalu mungkin terjadi, maka sistem terdistribusi harus memilih:

CP → Konsisten, tapi jika jaringan terputus, sebagian layanan berhenti.

AP → Tetap tersedia, tapi data mungkin tidak konsisten sementara.

Teorema BASE

BASE adalah karakteristik sistem NoSQL dan kebalikan dari ACID.

Komponen BASE:
Komponen	Penjelasan
Basically Available	Sistem selalu memberi respons cepat
Soft State	Data tidak harus konsisten sepanjang waktu
Eventually Consistent	Konsistensi akan tercapai setelah beberapa waktu
Hubungan CAP dan BASE

BASE selaras dengan model AP dalam CAP:

Sistem tetap tersedia (Availability)

Sistem bertahan dari gangguan jaringan (Partition Tolerance)

Konsistensi diberikan belakangan (Eventually Consistent)

Contoh yang Saya Gunakan (Real Case)

Redis Cache

Ketika membaca data, Redis memberikan jawaban sangat cepat, meskipun data terkadang belum 100% sinkron dengan database utama. Artinya:

Redis mengutamakan Availability (A) + Partition Tolerance (P)

Konsistensi diperbaiki kemudian (eventual consistency)

Ini adalah contoh nyata hubungan CAP (AP) dan BASE.

2. Keterkaitan GraphQL dengan Komunikasi Antar Proses pada Sistem Terdistribusi
Hubungan GraphQL dengan IPC (Inter-Process Communication)

GraphQL bukan hanya query language untuk API, tetapi juga dapat digunakan sebagai protokol komunikasi antar layanan dalam sistem terdistribusi.

GraphQL memperbaiki komunikasi IPC dengan cara:
1. Contract-Based Communication

GraphQL memiliki schema → layanan lain dapat mengetahui struktur data secara pasti.
Ini memudahkan koordinasi antar microservices.

2. Mengurangi Chatty Communication

Tanpa GraphQL, proses sering melakukan banyak GET ke berbagai endpoint REST.
Dengan GraphQL → satu query dapat mengambil berbagai data sekaligus.

3. Typed Communication (Schema)

GraphQL menggunakan tipe data yang jelas, sehingga cocok untuk komunikasi antar proses yang memerlukan struktur data stabil.

4. Mendukung Query Fleksibel Antar Layanan

Setiap layanan hanya mengirim data yang diminta, bukan seluruh response (seperti REST).

Diagram Komunikasi GraphQL antar Proses
flowchart LR
    A([Client / Frontend]) -->|GraphQL Query| B([GraphQL Gateway<br>API Orchestrator])

    B --> C{Route Request?}

    C -->|Auth| D([Auth Service<br>Login & Token])
    C -->|User| E([User Service<br>Data User])
    C -->|Order| F([Order Service<br>Data Pesanan])
    C -->|Stock| G([Stock Service<br>Data Stok])

    classDef box fill:#ffffff,stroke:#e63946,stroke-width:2px,color:#000;
    classDef boxBlue fill:#ffffff,stroke:#1d3557,stroke-width:2px,color:#000;
    classDef decision fill:#fff,stroke:#6c5ce7,stroke-width:2px,color:#000;

    class A,B boxBlue;
    class D,E,F,G box;
    class C decision;
