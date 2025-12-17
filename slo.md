# Service Level Objectives (SLO)

## Service Overview
This document defines the Service Level Objectives for the Agama web application.

## User Journeys

### User Journey 1: View Agama Main Page
**Description:** User accesses the main Agama page to view content.

**SLI Type:** Availability

**SLI Specification:**
- Event: HTTP request to Agama main page
- Valid events: All requests through HAProxy (excluding health checks)
- Success criterion: Response status code 2xx or 3xx
- Recording: HAProxy logs collected by Promtail into Loki

**SLI Implementation:**
- Formula: `count(2xx + 3xx) / count(2xx + 3xx + 5xx) * 100`
- LogQL: `100 * sum(count_over_time({job="haproxy"} |~ "^[23]"[15m])) / sum(count_over_time({job="haproxy"} |~ "^[235]"[15m]))`

**SLO:** >= 95% over 15-minute rolling window

---

**SLI Type:** Latency

**SLI Specification:**
- Event: HTTP request to Agama main page
- Valid events: Successful requests (2xx/3xx responses)
- Success criterion: Response time below threshold
- Recording: HAProxy logs collected by Promtail into Loki

**SLI Implementation:**
- Formula: Average response time for successful requests
- LogQL: `avg_over_time({job="haproxy"} | regexp "^[2-3][0-9]{2} (?P<latency>[0-9.]+)" | unwrap latency [15m])`

**SLO:** average <= 100ms over 15-minute rolling window

### User Journey 2: Submit Data to Agama Application
**Description:** User submits data through the Agama web interface.

**SLI Type:** Availability

**SLI Specification:**
- Event: HTTP POST/PUT request to Agama application
- Valid events: All write requests through HAProxy
- Success criterion: Response status code 2xx or 3xx
- Recording: HAProxy logs collected by Promtail into Loki

**SLI Implementation:**
- Formula: `count(2xx + 3xx) / count(2xx + 3xx + 5xx) * 100` for POST/PUT requests

**SLO:** >= 90% over 15-minute rolling window

---

**SLI Type:** Latency

**SLI Specification:**
- Event: HTTP POST/PUT request to Agama application
- Valid events: Successful write requests (2xx/3xx responses)
- Success criterion: Response time below threshold
- Recording: HAProxy logs collected by Promtail into Loki

**SLI Implementation:**
- Formula: Average response time for successful write requests

**SLO:** average <= 100ms over 15-minute rolling window

## Monitoring and Alerting

SLIs are tracked through:
- HAProxy logs parsed by Promtail and stored in Loki
- Metrics visualized in Grafana Main dashboard
- 15-minute rolling window measurements
- Visual thresholds: Green (>= 98%), Yellow (60-98%), Red (< 60% time remaining)

## Review Period

SLOs are reviewed monthly and adjusted based on:
- Actual system performance
- User feedback
- Business requirements changes
