This project is a high-performance backend system built using Java and Spring Boot to ingest machine events, deduplicate and update event records,
and provide analytics endpoints for factories and production lines. Machines produce events containing eventId, eventTime, durationMs, and defectCount. 
The backend supports batch ingestion, data validation, deduplication, update rules, and statistical queries over a time window.

The system includes the following features:

Batch Event Ingestion (POST /events/batch):

Validates durationMs (0 to 6 hours)

Validates eventTime (not > 15 minutes in the future)

Deduplication by eventId and payloadHash

Update existing event only if newer receivedTime

Ignore receivedTime from client; server sets its own

Thread-safe using optimistic locking

Response includes counters (accepted, updated, deduped, rejected)

Machine Stats (GET /api/machine/stats?machineId=...&start=...&end=...):
Returns eventsCount, defectsCount, avgDefectRate, and status (Healthy or Warning). Defect count = -1 is ignored. Window is start inclusive and end exclusive.

Top Defect Lines (GET /api/stats/top-defects?factoryId=...&from=...&to=...&limit=10):
Returns a list of lines containing lineId, totalDefects, eventCount, and defectsPercent (defects per 100 events).

Architecture:
The project follows clean MVC layering.
Controllers: Accept HTTP requests
Services: Perform business logic
Repository: Database interactions
Entity: Maps to SQL tables
DTOs: Request and response models
Utils: Shared validation functions

Folder structure:
controller/ – EventController, MachineStatusController, TopDefectsController
service/ – EventISvc, MachineStatusISvc, TopDefectsISvc
repository/ – EventRepository
entity/ – EventEntity
dto/ – EventRequestDTO, ResponseGetOutputModel
util/ – ValidationUtils

Dedupe and Update Logic:
Duplicate event → same eventId + identical payloadHash → ignore
Update event → same eventId + different payloadHash + newer receivedTime → update
Older receivedTime → ignore

Thread-safety:
Handled using optimistic locking with @Version. Prevents race conditions. Repository operations ensure atomic updates. No shared mutable static state, so service is safe for concurrent batch processing.

Data Model:
Event table fields:
event_id, event_time, received_time, machine_id, factory_id, line_id, duration_ms, defect_count, payload_hash, version

Event table SQL:
CREATE TABLE mfidb.events (
event_id VARCHAR(50) PRIMARY KEY,
event_time VARCHAR(50),
received_time VARCHAR(50),
machine_id VARCHAR(50),
factory_id VARCHAR(50),
line_id VARCHAR(50),
duration_ms BIGINT,
defect_count INT,
payload_hash VARCHAR(255),
version INT DEFAULT 0
);

Recommended indexes:
CREATE INDEX idx_machine_time ON events(machine_id, event_time);
CREATE INDEX idx_factory_line_time ON events(factory_id, line_id, event_time);

Performance Strategy:
To process 1000 events under 1 second:

Fast in-memory validations

Atomic DB operations

Pre-indexed DB columns

Minimal object creation

Efficient loop processing

Optimistic locking instead of synchronized blocks

Test Cases Implemented (required 8):

Duplicate event deduped

Different payload + newer receivedTime updated

Older receivedTime ignored

Invalid duration rejected

Future event time rejected

defectCount = -1 ignored

Start inclusive / end exclusive validated

Thread safety verified with parallel ingestion tests

Setup Instructions:

Clone repository:
git clone https://github.com/kunalahire1/event-api-snow

cd event-api-snow

Create database:
CREATE DATABASE mfidb;

Update credentials in application.properties.

Run the application:
mvn spring-boot:run

BENCHMARK.md Summary:
Contains system specs, benchmark commands, measured time for ingestion of 1000 events, and performance optimization notes.

Edge Cases & Assumptions:

durationMs < 0 or > 6h → reject

eventTime > 15 min ahead → reject

defectCount = -1 → stored but ignored in stats

Empty batch allowed

Order of arrival irrelevant due to receivedTime logic

Default limit for top defects is configurable

Possible Improvements with More Time:

Add Kafka stream processing

Add Redis caching

Add Docker support

Add Prometheus/Grafana monitoring

Pagination for large stats queries

Contact:
For clarification or walkthrough, feel free to reach out.
