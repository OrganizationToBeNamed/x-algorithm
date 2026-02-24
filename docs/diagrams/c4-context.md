# C4 Context Diagram - X For You Feed System

**Navigation:** [Home](../index.md) · [Architecture](../ARCHITECTURE.md) · [Example Scenario](../EXAMPLE_SCENARIO.md) · [Pipeline Flow](pipeline-component-flow.md) · [Sequence](sequence-diagram.md) · [Phoenix ML](phoenix-ml-architecture.md) · [Thunder](thunder-architecture.md)

```mermaid
C4Context
    title X "For You" Feed - System Context (C4 Level 1)

    Person(user, "X User", "Views the For You timeline feed on X")

    System(homeMixer, "Home Mixer", "Orchestration layer that assembles the ranked For You feed via gRPC")

    System(thunder, "Thunder", "In-memory real-time post store for in-network content")
    System(phoenix, "Phoenix", "Grok-based ML system for retrieval & ranking")

    System_Ext(kafka, "Kafka", "Distributed event stream for tweet create/delete events")
    System_Ext(gizmoduck, "Gizmoduck", "User profile & metadata service")
    System_Ext(tes, "TES", "Tweet Entity Service - core post data")
    System_Ext(strato, "Strato", "Key-value store for user features & caching")
    System_Ext(socialgraph, "SocialGraph", "Social relationships (follow/block/mute)")
    System_Ext(vf, "Visibility Filtering", "Content safety & policy enforcement")

    Rel(user, homeMixer, "Requests For You feed", "gRPC")
    Rel(homeMixer, thunder, "Fetches in-network posts", "gRPC")
    Rel(homeMixer, phoenix, "Retrieves OON candidates & gets ranking scores", "gRPC")
    Rel(homeMixer, gizmoduck, "Fetches author profiles", "gRPC")
    Rel(homeMixer, tes, "Fetches post metadata", "gRPC")
    Rel(homeMixer, strato, "Reads/writes user features & cache", "gRPC")
    Rel(homeMixer, vf, "Checks content visibility", "gRPC")
    Rel(homeMixer, socialgraph, "Checks social relationships", "gRPC")
    Rel(kafka, thunder, "Tweet create/delete events", "Kafka")
```
