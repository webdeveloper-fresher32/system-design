# 📦 Module 7: Storage Systems — Object, Block & Distributed File Systems

> **Core Philosophy:** *Databases are engineered for structured query records; unstructured binary blobs (images, videos, backups) belong in horizontally scalable, virtually infinite Object Storage accessed via pre-signed direct URLs.*

---

## 📌 Table of Contents
1. [The Problem: Binary Blobs in Relational Databases](#1-the-problem-binary-blobs-in-relational-databases)
2. [Storage Types: Object vs Block vs File Storage](#2-storage-types-object-vs-block-vs-file-storage)
3. [AWS S3 & Cloud Object Storage Architecture Internals](#3-aws-s3--cloud-object-storage-architecture-internals)
4. [The Pre-Signed URL Pattern: Eliminating Server Bottlenecks](#4-the-pre-signed-url-pattern-eliminating-server-bottlenecks)
5. [Multi-Part Chunked Uploads & Resumable File Transfers](#5-multi-part-chunked-uploads--resumable-file-transfers)
6. [Deduplication at Scale: Content-Defined Chunking (CDC)](#6-deduplication-at-scale-content-defined-chunking-cdc)
7. [Storage Lifecycle Policies & Tiering (Standard to Glacier)](#7-storage-lifecycle-policies--tiering-standard-to-glacier)
8. [Java / Spring Boot AWS S3 Pre-Signed Upload & Event Ingestion](#8-java--spring-boot-aws-s3-pre-signed-upload--event-ingestion)
9. [Side-by-Side Storage Comparison Table](#9-side-by-side-storage-comparison-table)
10. [Interview Rapid Q&A Checklist](#10-interview-rapid-qa-checklist)

---

## 1. The Problem: Binary Blobs in Relational Databases

### Anti-Pattern: Storing 100MB Videos as BLOBs in PostgreSQL / MySQL
- Database buffer pool RAM is filled with raw image bytes instead of index pages.
- Database backups (`pg_dump`) balloon from gigabytes to petabytes.
- Replicating massive binary writes saturates inter-database network bandwidth.
- **Golden Rule:** Store media in **Object Storage (S3)**; store only the resulting immutable metadata URL (`https://cdn.example.com/media/uuid.jpg`) in the database!

---

## 2. Storage Types: Object vs Block vs File Storage

```
┌─────────────────┬───────────────────────────────┬───────────────────────────────┐
│ Storage Type    │ Abstraction                   │ Protocols & Characteristics   │
├─────────────────┼───────────────────────────────┼───────────────────────────────┤
│ Block Storage   │ Raw, unformatted volume disk  │ iSCSI, Fibre Channel, NVMe    │
│ (AWS EBS)       │ (formatted as ext4, NTFS)     │ Low latency (<1ms), random I/O│
├─────────────────┼───────────────────────────────┼───────────────────────────────┤
│ File Storage    │ Hierarchical directory tree   │ NFS, SMB, POSIX compliant     │
│ (AWS EFS / NFS) │ with folders and permissions  │ Shared multi-instance mounts  │
├─────────────────┼───────────────────────────────┼───────────────────────────────┤
│ Object Storage  │ Flat namespace bucket with    │ REST / HTTP (GET, PUT, DELETE)│
│ (AWS S3)        │ unique key-value object blobs │ Infinite scale, 11 9s durability│
└─────────────────┴───────────────────────────────┴───────────────────────────────┘
```

---

## 3. AWS S3 & Cloud Object Storage Architecture Internals

- **Flat Namespace:** Objects live in a flat bucket namespace with a string key (e.g. `users/42/profile.png`). Folders are purely visual abstractions represented by forward slashes (`/`).
- **Durability (11 9s - 99.999999999%):** AWS S3 replicates every uploaded byte across at least 3 geographically separated Availability Zones (AZs) using erasure coding.
- **Strong Consistency:** S3 provides immediate read-after-write consistency for PUT and DELETE requests of new and existing objects.

---

## 4. The Pre-Signed URL Pattern: Eliminating Server Bottlenecks

```
WRONG ARCHITECTURE (Server Chokepoint):
[ Client ] ── 500 MB Video ──▶ [ Spring Boot App ] ── 500 MB ──▶ [ AWS S3 Bucket ]
* App server exhausts heap memory, locks worker thread for 2 minutes, maxes out NIC!

PRODUCTION ARCHITECTURE (Direct Pre-Signed Upload):
[ Client ] ── 1. Request Upload URL ────────▶ [ Spring Boot API ]
[ Client ] ◀─ 2. Cryptographic Pre-Signed URL ─ [ S3 Client creates signed URL with IAM Secret ]
    │
    ▼ 3. Direct HTTP PUT of binary bytes
[ AWS S3 Bucket ]
    │ 4. S3 Event Notification triggers async pipeline
    ▼
[ AWS SQS / Kafka ] ──▶ [ Transcoding Worker Fleet ]
```

---

## 5. Multi-Part Chunked Uploads & Resumable File Transfers

For files exceeding $100	ext{ MB}$:
1. **Initiate:** Client calls S3 `CreateMultipartUpload` to obtain an `UploadId`.
2. **Chunking:** Client slices file into 5MB–20MB chunks locally.
3. **Parallel Upload:** Chunks upload in parallel over multiple HTTP connections.
4. **Resumability:** If chunk 7 fails due to a network drop, only chunk 7 is retried!
5. **Complete:** Client calls `CompleteMultipartUpload` passing the ordered list of chunk ETags; S3 stitches the file together.

---

## 6. Deduplication at Scale: Content-Defined Chunking (CDC)

How Google Drive and Dropbox save $40\%$ of cloud storage costs:
- Files are divided into chunks based on content hash (**Rabin Fingerprinting** or **SHA-256**).
- Before uploading chunk $C$, client checks metadata index: *"Does chunk with hash `e3b0c44...` already exist in S3?"*
- If **YES**, skip the upload entirely and create a database pointer to the existing chunk!

---

## 7. Java / Spring Boot AWS S3 Pre-Signed Upload Implementation

```java
@Service
@RequiredArgsConstructor
public class MediaUploadService {

    private final S3Presigner s3Presigner;

    public record PreSignedUploadResponse(String uploadUrl, String fileKey, Instant expiresAt) {}

    public PreSignedUploadResponse generateUploadUrl(String originalFileName, String contentType) {
        String fileKey = "media/" + UUID.randomUUID() + "-" + sanitize(originalFileName);

        PutObjectRequest putRequest = PutObjectRequest.builder()
            .bucket("production-media-storage")
            .key(fileKey)
            .contentType(contentType)
            .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
            .signatureDuration(Duration.ofMinutes(15)) // Valid for 15 mins only
            .putObjectRequest(putRequest)
            .build();

        PresignedPutObjectRequest presigned = s3Presigner.presignPutObject(presignRequest);

        return new PreSignedUploadResponse(
            presigned.url().toString(),
            fileKey,
            presigned.expiration()
        );
    }

    private String sanitize(String name) {
        return name.replaceAll("[^a-zA-Z0-9.-]", "_");
    }
}
```

---

## 8. Side-by-Side Storage Comparison Table

| Storage Class | Durability | Access Latency | Cost / GB / Month | Retrieval Fee |
| :--- | :--- | :--- | :--- | :--- |
| **S3 Standard** | 99.999999999% | Milliseconds | ~$0.023 | None |
| **S3 Infrequent Access (IA)**| 99.999999999% | Milliseconds | ~$0.0125 | Per GB retrieved |
| **S3 Glacier Flexible** | 99.999999999% | Minutes to Hours | ~$0.0036 | Per GB retrieved |
| **S3 Glacier Deep Archive**| 99.999999999% | 12 to 48 Hours | ~$0.00099 | Per GB retrieved |

---

## 9. Interview Rapid Q&A Checklist
- *How do you prevent malicious users from uploading arbitrary files via pre-signed URLs?* (Restrict the HTTP `Content-Type` header, enforce maximum `Content-Length` conditions in the IAM policy, and run an asynchronous virus-scanning Lambda on upload completion).
- *What is Origin Shielding in CDN?* (An intermediate cache tier between the CDN edge PoPs and the S3 origin that collapses multiple edge cache misses into a single request, protecting S3 from thundering herds).
