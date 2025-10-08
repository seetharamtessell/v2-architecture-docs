# Diagrams

## Overview
This directory contains all architectural diagrams for the V2 system.

## Organization

### system/
High-level system architecture diagrams
- `overall-architecture.png` - Complete system view
- `client-architecture.png` - Client components and interactions
- `server-architecture.png` - Server ecosystem and microservices

### flows/
Sequence and flow diagrams
- `request-sequence.png` - End-to-end request processing
- `sync-sequence.png` - AWS estate synchronization flow
- `agent-orchestration.png` - Agent collaboration workflow

### components/
Component-level diagrams
- `microservices-map.png` - Service topology and dependencies
- `data-flow.png` - Data movement across the system
- `integration-map.png` - External integration points

### source/
Editable diagram source files
- Store original files (draw.io, mermaid, etc.)
- Include instructions for editing

## Diagram Guidelines
1. **Keep diagrams simple**: Focus on key concepts, avoid clutter
2. **Use consistent notation**: Stick to standard diagram conventions
3. **Version control sources**: Always commit editable source files
4. **Export to PNG/SVG**: For easy viewing in documentation
5. **Update regularly**: Keep diagrams in sync with architecture changes

## Tools
- Draw.io / diagrams.net (recommended)
- Mermaid (for text-based diagrams)
- PlantUML
- Excalidraw

---
*Always maintain both source files and exported images.*