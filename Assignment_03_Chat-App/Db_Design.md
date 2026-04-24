# Database Technology Choices:
-> Users & Profiles : PostgreSQL : ACID, complex queries, relational integrity
-> Conversations & Group Metadata : PostgreSQL : Relational membership queries
-> Messages : Apache Cassandra : High write throughput, time-series access pattern, linear horizontal scale
-> Media Metadata : PostgreSQL : Relational, low volume
-> Session / Presence / Typing : Redis : Sub-ms latency, TTL-based expiry, pub/sub
-> Message Fan-out Queue : Kafka : Durable, ordered, high-throughput event streaming
-> Full-text Search : Elasticsearch : Inverted index for message keyword search
-> Blob Storage : Amazon S3 / GCS : Media files, E2EE ciphertext blobs

# Data Models:

-> Message table for 1 on 1 chat : The primary key is message_id, which helps to decide message sequence. We cannot rely on created_at to decide the message sequence because two messages can be created at the same time.

message_id : bigint
message_from : bigint
message_to : bigint
content : text
created_at : timestamp

-> Message table for group chat : The composite primary key is (channel_id, message_id). Channel and group represent the same meaning here. channel_id is the partition key because all queries in a group chat operate in a channel.

channel_id : bigint
message_id : bigint
user_id : bigint
content : text
created_at : timestamp

-> To ascertain the order of messages, message_id must satisfy the following two requirements:
1. IDs must be unique.

2. IDs should be sortable by time, meaning new rows have higher IDs than old ones.

# Entity-Relationship Summary:
users
  |─── (1:N) ──→ user_devices
  |─── (M:N) ──→ conversations  (via conversation_members)
  |─── (1:N) ──→ user_blocks

conversations
  |─── (1:N) ──→ conversation_members
  |─── (1:N) ──→ messages  [Cassandra]

messages
  |─── (N:1) ──→ users (sender)
  |─── (N:1) ──→ media_metadata (optional)
  |─── (1:N) ──→ message_status [Cassandra]