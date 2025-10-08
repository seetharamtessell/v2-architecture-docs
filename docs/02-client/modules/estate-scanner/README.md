# Estate Scanner Module

**Crate Name**: `cloudops-estate-scanner`
**Status**: To be designed 🔄

## Overview

The Estate Scanner discovers and indexes AWS resources across multiple accounts.

## Responsibilities

### 1. AWS API Integration
- Connect to AWS APIs
- Handle credentials from `~/.aws/credentials`
- Support multiple accounts/profiles
- Handle rate limiting, pagination, retries

### 2. Resource Discovery
- Scan EC2, RDS, S3, Lambda, VPC, ELB, etc.
- Transform to standard format
- Extract metadata (tags, state, permissions)
- Incremental vs full sync

## Architecture

See: [architecture.md](architecture.md) (TBD)

## API Reference

See: [api.md](api.md) (TBD)

## Scanning Strategies

See: [scanning-strategies.md](scanning-strategies.md) (TBD)

## AWS Service Support

See: [service-support.md](service-support.md) (TBD)