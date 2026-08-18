# Huawei Cloud – Denodo Integration

A summary of integration methods between Denodo and the Huawei Cloud Big Data Platform.

> **Note:** This documentation was written in early 2026. Some details may have changed since publication.

---

## Background

Denodo is a data virtualization platform that provides a unified access layer over heterogeneous data sources, both on-premises and cloud-based, across a wide range of vendors. Because Denodo virtualizes rather than persists data, it holds no storage of its own. Connectivity is therefore limited to the JDBC protocol.

---

## Problem Statement

Denodo exposes its views over JDBC, which in principle allows Huawei Cloud Big Data services to extract data from it. This document records which integration paths were validated, which were not viable, and the recommended approach for ingesting Denodo view data into the Huawei Cloud Big Data Platform.

---

## Solution Overview

Two integration approaches were evaluated:

| Method | Status | Reference |
|---|---|---|
| Data Lake Insight (DLI) | Validated | [Integration guide](dli-integration.md) |
| Data Warehouse Service (DWS) | Not supported — no JDBC driver available | [Notes](dws-integration.md) |

### Recommended Sequence

1. [Data preparation](data-preparation.md)
2. [Denodo server configuration](Denodo-server.md)
3. [DLI integration](dli-integration.md)
4. [DWS integration — not viable](dws-integration.md)
