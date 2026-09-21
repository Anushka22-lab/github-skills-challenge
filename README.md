# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

Hey there!

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!

## Task 1: Project Overview

This project is a small AIOps example for watching a `payment-service`. The service gives us
information such as response time, CPU usage, memory usage, and log messages.

The problem is to find unusual behaviour, such as slow payment requests, high resource usage, or
errors in the logs. AIOps checks this information, finds possible problems, and sends them through
the event workflow.

The workflow in this exercise is:

`Operational Data -> Anomaly Detection -> Event Generation -> Producer -> Topic -> Consumer -> AIOps Output`

The main files used are:

- `data/service_data.json` has the sample metrics and logs from the service.
- `src/anomaly_detector.py` checks each record and creates an event when it finds a problem.
- `src/event_producer.py` sends the anomaly events.
- `src/event_topic.py` is a simple in-memory topic that holds the events.
- `src/event_consumer.py` receives the events from the topic.
- `src/aiops_pipeline.py` joins these parts together and shows the final result.
- `tests/` has tests for anomaly detection, events, and the calculation example.

This is only a Python simulation, so it does not need a real Kafka or Airflow setup.

## Task 2: Operational Data Analysis

The sample data is in `data/service_data.json`. It contains 10 records for the
`payment-service`, covering one-minute intervals from 10:00 to 10:09 on 20 September 2026.

### Metrics

The metric fields are:

- `response_time_ms`: how long the request took, in milliseconds.
- `cpu_percent`: the percentage of CPU being used.
- `memory_percent`: the percentage of memory being used.

The `service` field identifies which service the record belongs to. It is not a metric itself.

### Log Information

The log fields are `log_level` and `message`. The level shows whether the record is normal or an
error, and the message gives more detail about what happened. In this data, normal records use
`INFO`, while the problem records use `ERROR`.

### Timestamps and Observations

The `timestamp` field records when each observation was made. The timestamps are in ISO format and
increase by one minute for each record, which makes it possible to see when the service changed
from normal behaviour to unusual behaviour and then recovered.

The records from 10:00 to 10:04 look normal. Response times stay between 120 and 142 ms, CPU stays
between 42% and 48%, memory stays between 51% and 55%, and the messages say that the payment was
processed successfully.

The record at 10:05 looks unusual. The response time rises to 610 ms and the log says `Payment
service timeout`. The record at 10:06 is also unusual and is more serious: the response time is
640 ms, CPU reaches 94%, memory reaches 91%, and the log says `Database connection timeout`.

The records from 10:07 to 10:09 look normal again. Their response times return to 138-150 ms,
CPU returns to 47-50%, memory returns to 55-57%, and the successful payment message comes back.

## Task 3: Anomaly Detection Results

I ran the provided detector against all 10 records. It processed the data successfully and found
two anomalous records:

- At `10:05`, the response time was `610 ms`, so it was flagged for **High response time**. The
	record also has an `ERROR` log with the message `Payment service timeout`.
- At `10:06`, the response time was `640 ms`, CPU was `94%`, and memory was `91%`. It was flagged
	for **High response time**, **High CPU utilization**, and **High memory utilization**. The log
	message was `Database connection timeout` and its level was `ERROR`.

The normal records were not flagged. This includes the records before the problem at 10:00-10:04
and the records after it at 10:07-10:09. Their metric values are below the detector thresholds.

In the first detector run, the error logs were visible in the source records but were not included
as reasons because the detector was checking for `WARNING` while the data uses `ERROR`. In the
original pipeline setup, the two events were also published to `service-events` while the consumer
listened to a different `anomaly-events` topic, so the first run reported 0 consumed events even
though 2 anomalies were detected.

The log-level check was corrected in Task 5, and the topic mismatch was corrected in Task 4. The
final report now includes both detected events and their error-log reason.

## Task 4: Event Flow Verification

I ran the workflow after connecting the consumer to the same topic used by the producer. The
execution processed 10 records, found 2 anomalies, and consumed 2 events.

The event flow works as follows:

1. The `AnomalyDetector` finds the unusual records at 10:05 and 10:06 and creates an anomaly
	event for each one.
2. The `EventProducer` receives each event and publishes it to the `service-events` topic.
3. The `EventTopic` holds the events in memory until they are read.
4. The `EventConsumer` reads both events from that same topic.
5. The pipeline passes the consumed events to the final AIOps output, which prints the service,
	timestamp, event type, and reasons for each anomaly.

The final output showed these two events:

- `2026-09-20T10:05:00` - high response time and an error log.
- `2026-09-20T10:06:00` - high response time, high CPU utilization, high memory utilization, and
	an error log.

This confirms that an anomaly becomes an event, travels through the producer and topic, is received
by the consumer, and reaches the downstream AIOps output.

## Task 5: Troubleshooting and Corrections

I found two problems while checking the workflow:

1. The consumer was listening to `anomaly-events`, but the producer was publishing to
	`service-events`. Because these were different in-memory topics, the consumer received no
	events. I corrected `src/aiops_pipeline.py` so the consumer uses the producer's topic. Running
	the pipeline again showed 2 detected events and 2 consumed events.
2. The anomaly detector checked for `WARNING` logs, but the supplied data contains `ERROR` logs.
	This meant the timeout records were detected from their metrics but did not include the log
	reason. I corrected `src/anomaly_detector.py` to check for `ERROR`. Running the pipeline again
	showed `Error log detected` in the reasons for both anomaly events.

Both corrections keep the existing detector, producer, topic, consumer, and pipeline structure.
The final run processed all 10 records and delivered both anomaly events to the AIOps output...

## Task 6: End-to-End Pipeline Execution

After making the corrections, I ran:

`python3 src/aiops_pipeline.py`

The final output confirmed the complete workflow:

- **Operational data processed:** 10 records were read from `service_data.json`.
- **Anomalies detected:** 2 records were identified as unusual.
- **Events generated:** each anomaly was turned into an `ANOMALY` event.
- **Events published:** the producer published both events to the `service-events` topic.
- **Events consumed:** the consumer received both events from the topic.
- **Events processed:** the pipeline printed both events in the final output.
- **AIOps result:** the output showed a payment-service timeout at 10:05 and a database connection
	timeout at 10:06, along with the high response time and resource-usage reasons.

This confirms the full path from operational data to the final AIOps output.

---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

