# Service Level Objectives (SLO)

## Service Overview
This document defines the Service Level Objectives for the Agama web application.

## User Journeys

### User Journey 1: View Agama Main Page
**Description:** User accesses the main Agama page to view content.

**Service Level Indicators (SLIs):**
- **Availability:** Percentage of successful HTTP requests (status codes 2xx and 3xx)
  - Target: 95% of requests succeed
  - Measurement: `(count of 2xx/3xx responses) / (total requests) * 100`

- **Latency:** Response time for the main page
  - Target: 95% of requests complete in < 500ms
  - Measurement: p95 response time from Nginx logs

**SLO Thresholds:**
- Availability SLO: >= 95%
- Latency SLO: p95 <= 500ms

### User Journey 2: Submit Data to Agama Application
**Description:** User submits data through the Agama web interface.

**Service Level Indicators (SLIs):**
- **Availability:** Percentage of successful POST/PUT requests
  - Target: 90% of write requests succeed
  - Measurement: `(count of successful POST/PUT with 2xx/3xx) / (total POST/PUT requests) * 100`

- **Latency:** Response time for data submission
  - Target: 95% of requests complete in < 1000ms
  - Measurement: p95 response time for POST/PUT requests from Nginx logs

**SLO Thresholds:**
- Availability SLO: >= 90%
- Latency SLO: p95 <= 1000ms

## Monitoring and Alerting

SLIs are tracked through:
- Nginx access logs parsed by Promtail and stored in Loki
- Metrics visualized in Grafana dashboards
- Prometheus for metric aggregation and alerting

## Review Period

SLOs are reviewed monthly and adjusted based on:
- Actual system performance
- User feedback
- Business requirements changes
