# Model Registry — Architecture Overview

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Clients & UI Layer                                 │
│                                                                                 │
│  ┌──────────────┐   ┌──────────────────┐   ┌──────────────────┐                │
│  │  Python       │   │  React UI        │   │  Catalog Python  │                │
│  │  Client       │   │  + Go BFF        │   │  Client          │                │
│  │ (clients/     │   │ (clients/ui/)    │   │ (catalog/clients/│                │
│  │  python/)     │   │                  │   │  python/)        │                │
│  └──────┬───────┘   └────────┬─────────┘   └────────┬─────────┘                │
│         │ HTTP/REST          │ HTTP/REST             │ HTTP/REST                │
└─────────┼────────────────────┼──────────────────────┼───────────────────────────┘
          │                    │                      │
          ▼                    ▼                      ▼
┌─────────────────────────────────────┐  ┌──────────────────────────────────────┐
│     Model Registry Server           │  │       Catalog Service                │
│     (cmd/proxy.go)                  │  │       (catalog/)                     │
│                                     │  │                                      │
│  ┌───────────────────────────────┐  │  │  ┌──────────────────────────────┐   │
│  │  OpenAPI Router & Handlers    │  │  │  │  OpenAPI Router & Handlers   │   │
│  │  (internal/server/openapi/)   │  │  │  │  (catalog/internal/server/)  │   │
│  └──────────────┬────────────────┘  │  │  └──────────────┬───────────────┘   │
│                 │                    │  │                 │                    │
│  ┌──────────────▼────────────────┐  │  │  ┌──────────────▼───────────────┐   │
│  │  Core Service                 │  │  │  │  Catalog Providers           │   │
│  │  (internal/core/)             │  │  │  │  (catalog/internal/catalog/) │   │
│  │  • RegisteredModel CRUD      │  │  │  │  • YAML Catalog              │   │
│  │  • ModelVersion CRUD         │  │  │  │  • Hugging Face Hub          │   │
│  │  • Artifact Management       │  │  │  │  • Leader Election (HA)      │   │
│  │  • InferenceService CRUD     │  │  │  └──────────────┬───────────────┘   │
│  │  • Experiment Tracking       │  │  │                 │                    │
│  └──────────────┬────────────────┘  │  │                 │ HTTP               │
│                 │                    │  │                 ▼                    │
│  ┌──────────────▼────────────────┐  │  │  ┌──────────────────────────────┐   │
│  │  DB Layer (GORM)              │  │  │  │  External Sources            │   │
│  │  (internal/db/)               │  │  │  │  • Hugging Face Hub API      │   │
│  │  • Repository Pattern         │  │  │  │  • Static YAML Files         │   │
│  │  • Filter Query Parser        │  │  │  └──────────────────────────────┘   │
│  │  • Pagination & Sorting       │  │  │                                      │
│  └──────────────┬────────────────┘  │  └──────────────────────────────────────┘
│                 │ SQL                │
│                 ▼                    │
│  ┌──────────────────────────────┐   │
│  │  MySQL  or  PostgreSQL       │   │
│  │  (with embedded migrations)  │   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Integration Layer                            │
│                                                                                 │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐  │
│  │  K8s Controller       │  │  CSI Storage         │  │  Async Upload Job   │  │
│  │  (cmd/controller/)    │  │  Initializer         │  │  (jobs/async-       │  │
│  │                       │  │  (cmd/csi/)          │  │   upload/)          │  │
│  │  • Watches CRDs       │  │                      │  │                     │  │
│  │  • Syncs model        │  │  • Parses custom URI │  │  • S3 / OCI / URI   │  │
│  │    metadata to MR     │  │    model-registry:// │  │  • Sigstore signing │  │
│  │  • Kubebuilder-based  │  │  • Downloads from    │  │  • ConfigMap-based  │  │
│  │                       │  │    S3/GCS/HTTP/OCI   │  │    metadata         │  │
│  └──────────┬───────────┘  │  • Injects into       │  └──────────┬──────────┘  │
│             │              │    KServe pods         │             │             │
│             │              └──────────┬─────────────┘             │             │
│             │ REST API               │ REST API                  │ REST API    │
│             └────────────┬───────────┘                           │             │
│                          ▼                                       ▼             │
│              ┌──────────────────────────────────────────────────────┐          │
│              │          Model Registry Server (REST API)            │          │
│              └──────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Data Model

```
┌─────────────────────────┐         ┌─────────────────────────┐
│   RegisteredModel       │         │  ServingEnvironment     │
│   • name, description   │         │  • name, description    │
│   • owner, state        │         │  • custom_properties    │
│   • custom_properties   │         └───────────┬─────────────┘
└───────────┬─────────────┘                     │
            │ 1:N                               │ 1:N
            ▼                                   ▼
┌─────────────────────────┐         ┌─────────────────────────┐
│   ModelVersion          │         │  InferenceService       │
│   • name, state         │◄────────│  • model_version_id     │
│   • author, description │ serves  │  • runtime, state       │
│   • custom_properties   │         │  • desired_state        │
└───────────┬─────────────┘         └───────────┬─────────────┘
            │ 1:N                               │ 1:N
            ▼                                   ▼
┌─────────────────────────┐         ┌─────────────────────────┐
│   ModelArtifact         │         │  ServeModel             │
│   • uri (s3/gcs/oci/..) │◄────────│  • model_artifact_id    │
│   • model_format_name   │  links  │  • last_known_state     │
│   • model_format_version│         └─────────────────────────┘
│   • storage_key/path    │
│   • state               │
└─────────────────────────┘

┌─────────────────────────┐
│   Experiment            │
│   • name, description   │
└───────────┬─────────────┘
            │ 1:N
            ▼
┌─────────────────────────┐
│   ExperimentRun         │
│   • metrics, parameters │
│   • datasets            │
└─────────────────────────┘
```

## Code Generation Flow

```
  api/openapi/src/*.yaml          Converter Interfaces         DB Schema
  (Source OpenAPI specs)          (internal/converter/)        (MySQL/PostgreSQL)
          │                              │                          │
          ▼                              ▼                          ▼
  ┌─────────────────┐          ┌──────────────────┐       ┌──────────────┐
  │ make api/openapi│          │ make gen/converter│       │ make gen/gorm│
  │ /model-registry │          │  (goverter)       │       │  (Docker)    │
  │ .yaml (merge)   │          └────────┬─────────┘       └──────┬───────┘
  └────────┬────────┘                   │                        │
           │                            ▼                        ▼
           ▼                  internal/converter/       internal/db/schema/
  api/openapi/                  generated/                *.gen.go
  model-registry.yaml
           │
     ┌─────┴──────┐
     ▼            ▼
  gen/openapi   gen/openapi-server
     │            │
     ▼            ▼
  pkg/openapi/  internal/server/
  (Go client)    openapi/
                (HTTP handlers)
```

## Deployment Topology (Kubernetes)

```
┌─────────────────── Kubernetes Cluster ───────────────────────────┐
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐ │
│  │ Model       │  │ Catalog     │  │ Database (MySQL/PG)      │ │
│  │ Registry    │  │ Service     │  │ StatefulSet              │ │
│  │ Deployment  │  │ Deployment  │  │                          │ │
│  │ :8080       │  │             │  │ :3306 / :5432            │ │
│  └──────┬──────┘  └─────────────┘  └──────────────────────────┘ │
│         │                                                        │
│  ┌──────┴──────────────────────────────────────────────────────┐ │
│  │ K8s Controller │ CSI Initializer │ Async Upload Jobs        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ KServe InferenceServices (model serving)                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  External: S3 / GCS / OCI Registry / Hugging Face Hub            │
└──────────────────────────────────────────────────────────────────┘
```
