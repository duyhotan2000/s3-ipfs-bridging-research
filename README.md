# s3-ipfs-bridging-research
Research about solutions for S3 bridging with IPFS platform

I. Solutions:
I did some researches by the reference links in the assesments and also finding around the internet. There are some third parties services and also some open sources providing the solution for for S3-IPFS bridging.
- Third party services (commercial):
    - Filebase: Filebase provides a unique IPFS pinning service to make decentralized storage as easy as Web 2.0 cloud storage
    - Storj: Storj is the leading provider of enterprise-grade, globally distributed cloud object storage. It provides solution to store files by S3-compatible APIs to their own distributed network. Also provides IPFS pinning service (in beta) and open source plugin to integrate with. 
- Open sources:
    - RTradeLtd/s3x: minio plugin to support IPFS file pinning, but it is now discontinued.
- My researched architecture and solutions:
    - Implement our own bridging service with Pinita IPFS pinning (our service <-> Pinata <-> IPFS). Pinata provides HTTP APIs to store files and pinning files to IPFS nodes. There are also some other services are similar to Pinata. This solution is quite good, but we will relying Pinita and hard to scale in the future as we do not control everything.
    - Implement solution that's not relying any commercial third-party IPFS services (like Pinata). I will describe about this solution in the next part.
II. My researched architecture:
Alright, here's the final architecture design for the S3-compatible API service with IPFS as the backend storage, incorporating a message queue for optimized performance:

1. Client Applications

Interact with the service using standard S3 SDKs, CLI tools (configured with the service's custom endpoint), or HTTP clients adhering to the S3 API.
Perform operations like uploading (PUT), downloading (GET), listing (GET Bucket), deleting (DELETE), and retrieving metadata (HEAD).
2. Node.js S3-Compatible API Service

API Endpoints (Express.js):
/bucket/key (PUT, GET, HEAD, DELETE)
/bucket/ or / (GET, PUT, DELETE - for bucket-like operations)
S3 Request Handling Middleware: Parses incoming S3-style HTTP requests, extracts relevant information (bucket, key, headers, body).
Authentication & Authorization Middleware (Optional): Verifies client credentials and enforces access policies for buckets and objects.
Rate Limiting Middleware (Optional): Protects the service from abuse by limiting the number of requests.
IPFS Interaction Layer:
Initializes and manages the ipfs-http-client.
Provides functions to addFile, getFile, pinCID, unpinCID.
Handles file chunking/assembly (if needed for large files).
Determines CID generation strategy.
Metadata Management Layer:
Interacts with the Metadata Database to store and retrieve mappings between S3 "buckets," "keys," and IPFS CIDs, along with other metadata.
Provides functions for creating/deleting "buckets" (logical), creating/retrieving/updating/deleting object metadata.
Message Queue Integration:
Client library for the chosen message queue (e.g., bull, amqplib).
Functions to publish jobs to relevant queues (e.g., pinning-queue, metadata-processing-queue).
S3 Response Generation: Formats responses in the XML or JSON structure expected by S3 clients, including headers (ETag, Content-Length, etc.) and error responses.
Configuration Management: Loads settings for IPFS node, database, message queue, and API.
Logging & Error Handling: Records service activity and handles errors gracefully, providing S3-compatible error responses.
3. Message Queue (e.g., Redis with Bull, RabbitMQ)

Queues:
pinning-queue: For asynchronous pinning of newly uploaded CIDs to IPFS. Messages contain the CID to be pinned.
unpinning-queue: For asynchronous unpinning of CIDs when objects are "deleted." Messages contain the CID to be unpinned.
metadata-processing-queue (Optional): For background processing or indexing of metadata. Messages contain object metadata.
Other queues as needed for future asynchronous tasks.
4. Worker Processes

Separate Node.js processes dedicated to consuming messages from the queues.
Pinning Worker: Connects to the pinning-queue, retrieves CIDs, and uses the IPFS client to pin the corresponding data. Handles retries and logging of pinning status.
Unpinning Worker: Connects to the unpinning-queue, retrieves CIDs, and uses the IPFS client to unpin the corresponding data.
Metadata Processing Worker (Optional): Connects to the metadata-processing-queue and performs any required background processing on the metadata.
Can be scaled independently based on the volume of queued tasks.
5. IPFS Node(s)

Can be a local IPFS daemon or a connection to a remote IPFS service (Infura, Pinata, etc.).
Stores the actual file data addressed by CIDs.
Pinning ensures data availability.
6. Metadata Storage (Database - e.g., PostgreSQL, MongoDB)

Stores the crucial mapping between S3 "buckets" and "keys" and their corresponding IPFS CIDs.
Contains other metadata like original filename, upload time, size, ETag, and pin status.
Optimized for querying based on "bucket" and "key."
Data Flow (Example: Upload)

Client PUT /my-bucket/my-image.jpg with file data.
API Service receives the request.
IPFS Interaction Layer adds the file to IPFS, gets Qm...CID.
Metadata Management Layer stores metadata: bucket: my-bucket, key: my-image.jpg, cid: Qm...CID, etc.
API Service publishes a job to the pinning-queue with Qm...CID.
API Service responds to the client with an S3-compatible success (including ETag).
Pinning Worker consumes the job from pinning-queue and pins Qm...CID on the IPFS node.
Data Flow (Example: Download)

Client GET /my-bucket/my-image.jpg.
API Service receives the request.
Metadata Management Layer retrieves the CID associated with my-bucket and my-image.jpg (e.g., Qm...CID).
IPFS Interaction Layer retrieves the data from IPFS using Qm...CID.
API Service streams the data back to the client with S3-compatible headers.
Architecture Diagram:

+-------------------------+        +---------------------------------+        +---------------+
| S3-Compatible Clients   | <----> | Node.js S3 API Service        |   <----> | Message Queue |
+-------------------------+        |                               |          +---------------+
                                     |                                        ^         ^
                                     |                                        |         |
                                     v                                        |         |
                           +-----------------------+                  +-----------------+
                           | Metadata Storage (DB) | <----------------> | Worker Processes|
                           +-----------------------+                  +-----------------+
                                     ^
                                     |
                                     | (CID Mapping)
                                     v
                           +---------------------+
                           | IPFS Node(s)        |
                           +---------------------+
This final architecture design incorporates a message queue to handle asynchronous tasks like pinning, improving the responsiveness and scalability of your S3-compatible IPFS-backed storage service. Remember to choose the specific technologies for the message queue and database based on your project's requirements and infrastructure.

To optimize upload/download file performance, we can apply chunking/assembly mechanism onto this solution.

III. References:
- https://github.com/filebase/filebase-sdk
- https://pinata.cloud/
- https://github.com/RTradeLtd/s3x
- https://gemini.google.com/
- Gemini prompts:
    - How to upload file to ipfs by nodejs
    - How to upload file to ipfs by nodejs without third party service?
    - Design an architecture of nodejs ipfs file storage service
    - Design above application but using s3 compatible apis
    - Still using ipfs as backed storage but apis are s3
    - Do we need to use any message queue or job queue in above architecture to optimize performance?
    - How to handle file chunking, assembly in above architecture?
