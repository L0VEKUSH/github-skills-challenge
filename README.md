# AIOps Service Monitoring Simulation

## Scenario

This project simulates a lightweight AIOps workflow for a payment-processing service. The service exposes metrics such as response time, CPU usage, and memory usage while also emitting log events that describe operational health. The operational problem being addressed is that the service occasionally becomes slow and unstable under load, which can lead to timeouts and resource saturation. The goal of the AIOps workflow is to identify these abnormal conditions early, convert them into structured events, pass them through a simple producer/topic/consumer pipeline, and surface the final operational issue for downstream response.

## Operational Data

The synthetic data is stored in `data/service_data.json`. Each record represents one observation from the monitored service and contains:

- `timestamp`: the time the observation was captured
- `service`: the service name
- `response_time_ms`: the application latency metric
- `cpu_percent`: the CPU utilization metric
- `memory_percent`: the memory utilization metric
- `log_level`: the log severity level
- `message`: the accompanying log message

### Metrics vs logs

The metric fields are:

- `response_time_ms`
- `cpu_percent`
- `memory_percent`

The log fields are:

- `log_level`
- `message`

Timestamps are used as the time dimension for each observation. In the dataset, readings are recorded on 10 distinct timestamps at roughly one-minute intervals from `2026-09-20T10:00:00` to `2026-09-20T10:09:00`.

## Data Observations

### Normal behaviour

The first, second, third, fourth, fifth, seventh, eighth, ninth, and tenth observations are consistent with normal behaviour for the service. These readings stay below the key thresholds and have informational log messages such as:

- `Payment request processed successfully`

The service appears healthy during those intervals, with response times around 120-150 ms and low CPU/memory values.

### Unusual behaviour

The unusual readings are concentrated at the following timestamps:

- `2026-09-20T10:05:00`
- `2026-09-20T10:06:00`

At `10:05:00`, the service shows elevated response time (`610 ms`) and an `ERROR` log message, `Payment service timeout`.

At `10:06:00`, the service becomes much more degraded, with:

- `response_time_ms = 640`
- `cpu_percent = 94`
- `memory_percent = 91`
- `log_level = ERROR`
- `message = Database connection timeout`

These conditions clearly deviate from normal operating behaviour and represent the anomalous patterns the AIOps workflow is designed to capture.

## Anomaly Detection Findings

The implementation in `src/anomaly_detector.py` evaluates a record as anomalous when any of the following are true:

- response time exceeds the threshold of 500 ms
- CPU usage exceeds 80%
- memory usage exceeds 80%
- the log level is `ERROR`, `WARNING`, or `CRITICAL`

Using the provided data, the detector identifies two anomalies:

1. `2026-09-20T10:05:00` — High response time, Error log detected
2. `2026-09-20T10:06:00` — High response time, High CPU utilization, High memory utilization, Error log detected

This result is consistent with the underlying operational data and distinguishes the normal observations from the faulty ones. The detector returns a readable anomaly object with the service name, timestamp, anomaly type, and list of reasons.

### Limitation / possible improvement

The detection logic is intentionally simple and threshold-based. It is effective for obvious spikes, but it may miss slow gradual degradation or generate false positives when thresholds are not tuned to a service's baseline. A useful improvement would be to compare against a rolling baseline or statistically derived thresholds rather than fixed percentages.

## Event-Processing Flow

The event flow mirrors the AIOps workflow described in the challenge:

1. Operational data is read and processed.
2. `AnomalyDetector.detect()` identifies abnormal records.
3. Each anomaly is turned into an event payload.
4. `EventProducer` publishes the event to the `anomaly-events` topic.
5. `EventConsumer` consumes the event from that topic.
6. The downstream pipeline reports the detected issue.

The key components are:

- `EventTopic`: the in-memory event stream
- `EventProducer`: publishes anomalies to the topic
- `EventConsumer`: receives messages from the topic
- message/event: the structured anomaly object containing `type`, `service`, `timestamp`, and `reasons`

## Final Workflow Result

Running the complete pipeline with `python src/aiops_pipeline.py` produced:

- `Records processed: 10`
- `Anomalies detected: 2`
- `Events consumed: 2`

The final output identifies the degraded payment service and explains the reasons for the anomaly, including slow response times and timeout-related error logs.

## Issues Identified and Corrected

The workflow originally contained several problems that prevented it from working reliably:

1. Import path issue for pytest: the project root was not configured so `from src...` imports did not resolve in the validation environment.
   - Fix: added a `pytest.ini` file with `pythonpath = .` and a package initializer in `src/__init__.py`.

2. Incorrect anomaly detection logic: the issue was treating only `WARNING` as an error signal, leaving `ERROR` and other high-severity log events unflagged.
   - Fix: detect `ERROR`, `WARNING`, and `CRITICAL` as relevant log-driven anomaly indicators.

3. Broken event flow: the producer and consumer were connected to different topics, preventing the event from being observed by the downstream consumer.
   - Fix: both components use the same `anomaly-events` topic.

4. Direct script execution compatibility: component imports worked only in one execution context.
   - Fix: added defensive imports to support both package-style and direct script execution.

## Reproduction Steps

To reproduce the workflow locally:

1. Open the repository in the project environment.
2. Run the validation suite:
   - `pytest -q`
3. Execute the AIOps pipeline:
   - `python src/aiops_pipeline.py`
4. Review the reported anomalies and consumed events.

The validation checks confirm that the operational data can be processed, anomalies are detected, event generation works, the consumer sees the message, and the end-to-end AIOps flow completes successfully.

## Verification

The repository validation currently passes:

- `pytest -q` => `8 passed in 0.02s`

This verifies that the operational data can be processed successfully, anomalies are detected as expected, and the simulated event pipeline works end-to-end within the provided architecture.

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

