---
layout: post
title: "Tracking KL Bus Punctuality with GTFS and BigQuery"
date: 2026-05-27
description: "How I built a pipeline to measure Rapid KL bus lateness using real-time GPS data, Airflow, dbt, and Looker Studio."
---

Public transit reliability is hard to observe from the outside. You can feel it as a rider, but quantifying it — which routes are worst, at what times, how consistently — requires data. This project builds a pipeline to do that for Rapid KL and MRT feeder buses in Kuala Lumpur.

The repo is [on GitHub](https://github.com/nicolenair/kl-bus-reliability-tracker).

## Architecture

```
GTFS Realtime API (30s) ──► GCS (raw JSON) ──► BigQuery ──► dbt ──► Looker Studio
GTFS Static API (daily) ──► BigQuery ──────────────────────►
```

Airflow handles orchestration. Terraform provisions the GCP infrastructure. The pipeline runs on two rhythms: position data extracted every 30 seconds, static schedule data (routes, stops, timetables) truncate-loaded daily.

## The core problem: estimating arrivals

The GTFS Realtime API gives you bus coordinates every 30 seconds. It doesn't give you arrival events. So the pipeline estimates when a bus arrived at each stop by finding the point in time when its GPS coordinates were closest to the stop's lat/long.

This works reasonably well in practice but introduces some noise — a bus could be nearest to a stop while stopped in traffic just before it, for instance. I've scoped the analysis to acknowledge this rather than try to engineer around it.

## Key decisions

### Time window: 5 AM–11 PM only

Overnight routes span calendar day boundaries in ways that break naive date-based arrival assignment. A bus that was "scheduled for 11:45 PM" but arrives at "12:05 AM" the next day looks absurdly late under a simple date join. Rather than handling this edge case carefully, I excluded the overnight window entirely. The 5 AM–11 PM window covers the service hours that matter most anyway.

### First and last stops excluded

On circular routes, the first and last stops are often the same location, which creates duplicate arrival signals and skews punctuality metrics at those endpoints. Excluding them keeps the analysis cleaner.

### Partitioning and clustering

The output mart table is partitioned by `actual_arrival_time` and clustered by `route_id`. Most dashboard queries filter on date ranges and specific routes, so this structure keeps scan costs low as the table grows.

## What the dashboard shows

Looker Studio sits on top of the BigQuery mart and surfaces three main views: average delay by route, delay patterns by time of day, and a geographic heatmap of late arrivals. The goal is to make it easy for someone looking at operations to spot which routes are structurally late versus occasionally late.

## What I'd improve

- **Arrival estimation accuracy** — the nearest-coordinate approach could be tightened with a bearing check (is the bus approaching vs. departing the stop?) to reduce false positives
- **Overnight handling** — properly accounting for routes that cross midnight would expand analysis coverage
- **Alert layer** — flagging routes that exceed a lateness threshold over a rolling window, rather than just reporting after the fact
