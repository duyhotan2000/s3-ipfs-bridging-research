# S3-IPFS Bridging Research

Research about solutions for S3 bridging with IPFS platform.

## I. Solutions

I did some researches by the reference links in the assessments and also finding around the internet. There are some third parties services and also some open sources providing the solution for S3-IPFS bridging.

### Third Party Services (Commercial)
- **Filebase**: Filebase provides a unique IPFS pinning service to make decentralized storage as easy as Web 2.0 cloud storage
- **Storj**: Storj is the leading provider of enterprise-grade, globally distributed cloud object storage. It provides solution to store files by S3-compatible APIs to their own distributed network. Also provides IPFS pinning service (in beta) and open source plugin to integrate with.

### Open Sources
- **RTradeLtd/s3x**: Minio plugin to support IPFS file pinning, but it is now discontinued.

### My Researched Architecture and Solutions
1. **Pinata IPFS Pinning Solution**
   - Implement our own bridging service with Pinata IPFS pinning (our service <-> Pinata <-> IPFS)
   - Pinata provides HTTP APIs to store files and pinning files to IPFS nodes
   - Similar services to Pinata are available
   - **Pros**: Good solution with ready-to-use APIs
   - **Cons**: Reliance on Pinata, hard to scale in the future as we don't control everything

2. **Self-Contained Solution**
   - Implementation without relying on commercial third-party IPFS services
   - Detailed architecture described in the next section

## II. Architecture Design

A comprehensive architecture design for S3-compatible API service with IPFS as the backend storage, incorporating a message queue for optimized performance.

### 1. Client Applications
- Interact using standard S3 SDKs, CLI tools, or HTTP clients adhering to the S3 API
- Support operations: upload (PUT), download (GET), list (GET Bucket), delete (DELETE), metadata (HEAD)

### 2. Node.js S3-Compatible API Service

#### API Endpoints (Express.js)
- `/bucket/key` (PUT, GET, HEAD, DELETE)
- `/bucket/` or `/` (GET, PUT, DELETE - for bucket-like operations)

#### Components
- **S3 Request Handling Middleware**
  - Parses incoming S3-style HTTP requests
  - Extracts bucket, key, headers, body information

- **Authentication & Authorization Middleware** (Optional)
  - Verifies client credentials
  - Enforces access policies

- **Rate Limiting Middleware** (Optional)
  - Protects service from abuse
  - Limits request frequency

- **IPFS Interaction Layer**
  - Manages ipfs-http-client
  - Handles file operations (addFile, getFile, pinCID, unpinCID)
  - Manages file chunking/assembly
  - Determines CID generation strategy

- **Metadata Management Layer**
  - Manages S3-IPFS mappings
  - Handles bucket and object metadata
  - Provides CRUD operations

- **Message Queue Integration**
  - Uses client libraries (bull, amqplib)
  - Manages job publishing to queues

- **S3 Response Generation**
  - Formats XML/JSON responses
  - Handles headers and error responses

- **Configuration Management**
  - Loads service settings
  - Manages environment variables

- **Logging & Error Handling**
  - Records service activity
  - Provides S3-compatible error responses

### 3. Message Queue System
- **Queues**:
  - `pinning-queue`: Asynchronous CID pinning
  - `unpinning-queue`: Asynchronous CID unpinning
  - `metadata-processing-queue`: Background metadata processing

### 4. Worker Processes
- Dedicated Node.js processes for queue consumption
- **Types**:
  - Pinning Worker
  - Unpinning Worker
  - Metadata Processing Worker
- Scalable based on task volume

### 5. IPFS Node(s)
- Local IPFS daemon or remote service connection
- Stores file data addressed by CIDs
- Ensures data availability through pinning

### 6. Metadata Storage
- Database (PostgreSQL/MongoDB)
- Stores S3-IPFS mappings
- Manages metadata (filename, upload time, size, ETag, pin status)
- Optimized for bucket/key queries

## Data Flow Examples

### Upload Flow
1. Client PUT `/my-bucket/my-image.jpg` with file data
2. API Service receives request
3. IPFS Layer adds file, gets CID
4. Metadata Layer stores mapping
5. Job published to pinning-queue
6. Client receives success response
7. Worker pins CID on IPFS node

### Download Flow
1. Client GET `/my-bucket/my-image.jpg`
2. API Service receives request
3. Metadata Layer retrieves CID
4. IPFS Layer gets data
5. Service streams data to client

## Architecture Diagram

```
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
```

## Performance Optimization
- File chunking/assembly mechanism can be implemented
- Asynchronous processing through message queues
- Scalable worker processes

## III. References
- [Filebase SDK](https://github.com/filebase/filebase-sdk)
- [Pinata Cloud](https://pinata.cloud/)
- [RTradeLtd/s3x](https://github.com/RTradeLtd/s3x)
- [Gemini](https://gemini.google.com/)

### Research Prompts Used
- How to upload file to IPFS by Node.js
- How to upload file to IPFS by Node.js without third party service?
- Design an architecture of Node.js IPFS file storage service
- Design above application but using S3 compatible APIs
- Still using IPFS as backed storage but APIs are S3
- Do we need to use any message queue or job queue in above architecture to optimize performance?
- How to handle file chunking, assembly in above architecture?
