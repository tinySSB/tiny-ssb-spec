# tinySSB Specification - Context So Far

## Project Overview

We are developing a comprehensive technical specification for tinySSB, a version of Secure Scuttlebutt (SSB) designed to work over constrained networks like LoRa. The specification aims to document the protocol in sufficient detail for independent implementations.

## Specification Structure

Based on the outline in `spec-arc.md`, the specification is structured into three main sections:

1. **Introduction**
   - Goals and motivation
   - Core design decisions
   - Important concepts
   - Scope and limitations

2. **Replication**
   - GOSET protocol for feed coordination
   - WANT vector mechanism
   - Replication process flow

3. **Feeds**
   - Feed structure and format
   - Message/packet format
   - DMX header mechanism
   - Main chain and side chain packets

## Current Progress

### Completed

- **Replication Section** (`replication-spec.md`): This section has been fully drafted and includes:
  - Goals and overview of the replication process
  - Detailed explanation of the GOSET protocol
  - GOSET synchronization process with diagrams
  - WANT vector protocol and packet format
  - Overview of the complete replication process
  - Limitations and considerations

- **Feeds Section** (`feeds-spec.md`): This section has been fully drafted and includes:
  - Feed structure and concepts
  - DMX header mechanism explanation
  - Main chain packet format
  - Type 0 messages (fixed size)
  - Type 1 messages (variable size)
  - Side chain packet format
  - CHNK vector protocol for side chain replication

### Source Materials

The specification is being developed based on three primary source files:

1. `spec-arc.md`: Outlines the overall structure and arc of the specification
2. `basic-spec.md`: Contains initial content on design decisions and feed structure
3. `wire-spec.md`: Details the wire protocol specification

### Formatting Decisions

- The specification is being written in Markdown format
- ASCII diagrams are used for packet byte layouts
- Mermaid diagrams are used for process flows
- Code examples are provided as pseudo-code only

## Next Steps

The following sections still need to be completed:

1. **Introduction Section**
   - Expand on the design decisions from basic-spec.md
   - Clearly define all important concepts and terminology
   - Outline the scope and limitations

2. **Integration**
   - Ensure consistent terminology across all sections
   - Add cross-references between related concepts
   - Create a comprehensive glossary

## Implementation References

Reference implementations are available at:
- `/home/mix24/projects/TINY-SSB/esp32-firmware/loramesh`
- `/home/mix24/projects/TINY-SSB/esp32-firmware/tinySSBlib`

These could be consulted for additional details or clarification if needed.

## Considerations and Decisions

1. **Diagrams**: ASCII diagrams are used for packet byte layouts, while Mermaid diagrams are used for process flows.

2. **CHNK Protocol**: The CHNK protocol was intentionally omitted from the replication section as it relates to side chains, which will be introduced in the Feeds section.

3. **Packet Exchange**: The detailed packet exchange process was kept minimal in the replication section, focusing instead on the coordination protocols (GOSET and WANT).

4. **Code Examples**: Only pseudo-code is included, with no specific implementation details.

5. **Test Vectors**: The specification may include test vectors in the future, but this is not a current priority.
