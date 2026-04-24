# Functional requirements:

1.1 User Management
-> Users can register with phone number or email + password.
-> Users can log in / log out across multiple devices simultaneously.
-> Users can update their profile (name, avatar, status/bio).
-> Users can search for other users by name or phone number.
-> Users can block / unblock other users.

1.2 Messaging
-> Users can send one-to-one (1:1) text messages in real time.
-> Users can send media messages: images, videos, audio clips, documents.
-> Users can send voice messages (recorded audio).
-> Messages support emoji reactions.
-> Users can reply to, forward, and delete messages (delete for self / delete for everyone).
-> Users can see message status: Sent ✓ → Delivered ✓✓ → Read ✓✓ (blue).

1.3 Group Chats
-> Users can create group conversations (up to 1,024 members).
-> Group admins can add/remove members, change group name/photo, and promote other admins.
-> Users can leave a group at any time.
-> Groups support all message types and reactions.

1.4 Presence & Notifications
-> System shows online / last-seen status (user-configurable visibility).
-> Typing indicators ("Alice is typing…") in 1:1 and group chats.
-> Push notifications delivered via APNs (iOS), FCM (Android), and Web Push when the app is backgrounded.
-> @mentions in group chats trigger targeted notifications.

1.5 Message History & Search
-> Full message history is persisted and retrievable with infinite scroll (pagination).
-> Users can search messages within a conversation by keyword.
-> Messages are stored end-to-end encrypted (E2EE); server stores only ciphertext.

1.6 Calls (Stretch Goal)
-> Voice calls and video calls between two users (WebRTC).
-> Group voice calls up to 8 participants.


# Non-Functional Requirements:

2.1 Scale
-> MetricTargetRegistered users1 billionDaily Active Users (DAU)500 millionConcurrent connections100 millionMessages per day100 billionPeak messages per second~5 million msg/sMedia storage growth~10 PB / year

2.2 Performance
-> Message delivery latency: < 100 ms p99 on the same continent.
-> Push notification delivery: < 2 s p99 globally.
-> API response time: < 200 ms p99 for non-media endpoints.
-> Media upload: < 5 s for images under 10 MB on a 4G connection.

2.3 Availability
-> System availability: 99.99% uptime (< 53 min downtime / year).
-> No single point of failure — all critical components are replicated across ≥ 3 availability zones.
-> Graceful degradation: message queueing survives transient WebSocket server restarts.

2.4 Durability
-> Messages must never be lost once the server acknowledges receipt.
-> At-least-once delivery with idempotency (deduplication on client and server).
-> Media stored with 11 nines of durability (object storage with cross-region replication).

2.5 Security
-> End-to-End Encryption (Signal Protocol / Double Ratchet).
-> TLS 1.3 for all data in transit.
-> Rate limiting per user and per IP to prevent spam and DDoS.

2.6 Consistency
-> Message ordering within a conversation is causal (preserving happens-before relationships).
-> Read receipts and delivery status are eventually consistent (acceptable lag: < 5 s).
-> User profile updates are strongly consistent within 1 s.

2.7 Maintainability & Observability
-> Centralized structured logging and distributed tracing (OpenTelemetry).
-> Metrics & alerting (Prometheus + Grafana), SLO-based alerting.
-> Feature flags for gradual rollouts.
-> CI/CD pipelines with automated integration and load tests.