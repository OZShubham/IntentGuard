# TradeGuard Mesh

> **AI investigation for letter-of-credit document discrepancies.**

TradeGuard Mesh is a GCP-native, evidence-driven multi-agent prototype for trade-finance document examination.

It helps a trade officer investigate a letter-of-credit presentation by extracting document data, verifying discrepancies deterministically, retrieving synthetic rules, calling a remote shipment-verification agent, and preparing an approval-ready resolution plan.

> **AI investigates. Humans decide.**

## Why TradeGuard?

Trade-finance document examination involves comparing information across letters of credit, invoices, packing lists, bills of lading, insurance certificates, and shipment records.

TradeGuard does not make payment or legal decisions. Instead, it produces an auditable investigation showing:

- What discrepancy was found
- Which documents and fields support it
- Which deterministic comparison established the fact
- Which synthetic rule or LC requirement applies
- What should happen next
- What requires human approval

## Prototype scope

The initial demo investigates four synthetic discrepancies:

1. Shipment date after the LC deadline
2. Invoice amount exceeding the LC amount
3. Invoice and packing-list quantity mismatch
4. Insurance coverage below the LC requirement

## Architecture

```text
Documents → Document AI → Extraction + Normalization
                               ↓
                    Deterministic Verification
                               ↓
                    Evidence-backed Findings
                               ↓
             MCP Rules / Shipment / Case Management
                               ↓
          A2A Remote Shipment Verification Agent
                               ↓
                  Resolution Plan + Human Approval
```

## Tech stack

- Google Cloud
- Gemini and Vertex AI
- Google Agent Development Kit (ADK)
- Document AI
- Vertex AI Search
- MCP for governed tools
- A2A for remote agent collaboration
- Cloud Run, Firestore, BigQuery, and Cloud Storage

## Status

🚧 **Hackathon prototype in active development**

All data used in this project is synthetic.

## Run locally

Setup instructions will be added as the prototype is implemented.

```bash
git clone [https://github.com/TODO/TradeGuard-Mesh.git](https://github.com/TODO/TradeGuard-Mesh.git)
cd TradeGuard-Mesh
```

## Disclaimer

TradeGuard Mesh is an investigation-support prototype. It does not provide legal advice, release or refuse payment, accept waivers, send external messages, or replace qualified trade-finance professionals.

