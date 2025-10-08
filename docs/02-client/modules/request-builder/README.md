# Request Builder Module

**Crate Name**: `cloudops-request-builder`
**Status**: To be designed 🔄

## Overview

The Request Builder enriches user prompts with context and communicates with the server.

## Responsibilities

### 1. Context Enrichment
- Take user prompt
- Search estate for mentioned resources
- Add full resource metadata
- Add relevant chat history

### 2. Server Communication
- Build request payload
- Send to server (HTTP client)
- Parse server response
- Handle retries and errors

## Architecture

See: [architecture.md](architecture.md) (TBD)

## API Reference

See: [api.md](api.md) (TBD)

## Request Flow

See: [request-flow.md](request-flow.md) (TBD)

## Server Protocol

See: [server-protocol.md](server-protocol.md) (TBD)