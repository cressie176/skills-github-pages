---
title: "Introducing Filby: An Open Source Library for Managing Shared Reference Data"
date: 2024-12-29
tags:
- Temporal
- Versioned
- Reference Data
- PostgreSQL
---
Shared reference data can be a tricky aspect of software development, especially in distributed or microservice-based architectures. Enter **Filby**, an open-source library that simplifies the management of such data, ensuring consistency, reliability, and flexibility. Here, we'll explore how Filby works, its key benefits, and why it might be the solution you've been searching for.

#### The Problem: Managing Shared Reference Data

Most applications rely on reference data—information that changes infrequently but must remain consistent across the system. However, managing this data across distributed systems introduces challenges:

| Challenge     | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Consistency   | Whenever we duplicate our reference data, we increase the likelihood of inconsistency. Even if we have one authoritive source of truth, we may cache the reference data in multiple systems, resulting in temporary inconsisenty unless cache updates are sychronised. Given the reference data is slow moving, a short period of inconsistency may be acceptable.                                                                                                             |
| Load Times    | Some reference data sets may be too large to desirably load over a network connection for web and mobile applications. Therefore we should discourage accidentally including large data sets into a client bundle, or requesting large data sets over a network.                                                                                                                                                                                                               |
| Reliability   | Requesting data sets over a network may fail, especially when mobile. Bundling local copies of reference data into the application (providing they are not too large) will aleviate this, but increase the potential for stale data.                                                                                                                                                                                                                                           |
| Stale Data    | Even though reference data is slow moving, it will still change occasionally. Therefore we need a strategy for refreshing reference data.                                                                                                                                                                                                                                                                                                                                      |
| Temporality   | When reference data changes, the previous values may still be required for historic comparisons. Therefore all reference data should have an effective date. Effective dates can also be used to synchronise updates by including future records when the values are known in advance. This comes at the cost of increased size, and there may still be some inconsistency due to clock drift and cache expiry times.                                                          |
| Evolution     | Both reference data, and our understanding of the application domain evolves over time. We will at some point need to make backwards incompatible changes to our reference data, and will need to do so without breaking client applications. This suggests a versioning and validation mechanism. The issue of temporality compounds the challenge of evolution, since we may need to retrospecively add data to historic records. In some cases this data will not be known. |
| Local Testing | Applications may be tested locally, and therefore any solution sould work well on a local development machine.                                                                                                                                                                                                                                                                                                                                                                        |

#### Meet Filby: Version Control for Reference Data

Think of Filby as **version control for reference data**. Just as source control systems like Git track changes to source code, Filby manages reference data changes. Like checking out a commit, your applications can use Filby to retrieve reference data for a given change set id. They can also inspect the changelog to find which change set was in effect at a given point in time, and subscribe to reference data update notifications.

#### Key Features and Benefits

Filby offers a structured approach to managing shared, temporal reference data, delivering several advantages:

1. **Safe, Predictable Updates**: Deploy new reference data ahead of time and activate it when needed.
2. **Consistency Across Systems**: The change set mechanism ensures consistency, even with distributed systems.
3. **Historic and Future Data Access**: Retrieve past data or schedule future changes.
4. **Customisable Projections**: Tailor views of reference data for specific clients or systems.
5. **Version Control**: Support backward-incompatible changes with versioned projections.
6. **Caching Made Easy**: Change sets should never change making them highly cacheable.
7. **Local Development Support**: Test applications locally using HTTP mocking libraries.

#### Filby’s Core Concepts

To understand how Filby works, let’s break down its main components:

- **Projections**: Versioned views of reference data, tailored to specific use cases.
- **Entities**: The individual pieces of reference data.
- **Data Frames**: Snapshots of entities at specific points in time, grouped by change sets.
- **Change Sets**: Logical bundles of updates with a common effective date.
- **Notifications**: Update events for maintaining synced data across systems.
- **Hooks**: Custom event handlers triggered by data changes.

<pre>
┌───────────────┐
│               │
│               │ announces changes via
│  Projection   │────────────────────────┐
│               │                        │
│               │                        │
└───────────────┘                        │
        │ depends on                     │
        │                                │
        │                                │
        │                                │
       ╱│╲                              ╱│╲
┌───────────────┐                ┌───────────────┐                 ┌──────────────┐
│               │                │               │                 │              │
│               │                │               │╲ delivered via  │              │
│    Entity     │                │ Notification  │─────────────────│     Hook     │
│               │                │               │╱                │              │
│               │                │               │                 │              │
└───────────────┘                └───────────────┘                 └──────────────┘
        │ aggregates                    ╲│╱ is raised by
        │                                │
        │                                │
        │                                │
       ╱│╲                               │
┌───────────────┐                ┌───────────────┐
│               │                │               │
│               │╲ is grouped by │               │
│  Data Frame   │────────────────│  Change Set   │
│               │╱               │               │
│               │                │               │
└───────────────┘                └───────────────┘
</pre>

#### A Real-World Use Case

Imagine a system managing holiday park data. With Filby:

1. Define entities like “Park” and “Season” in JSON or YAML files.
2. Create a changelog for the Park and Season data updates in JSON, YAML, CSV or SQL.
3. Tailor projections to specific needs, such as a Mobile App requiring a compact view of park details.
4. Use Filby’s API to retrieve park data at any point in time.
5. Add "hooks" to notify other systems that new data and/or projects are available.

<pre>
               Change
               Hook       Invalidate Cache
           ┌─────────┐   ┌──────────────┐
           │         │   │              │
           │         ▼   │              ▼
┌────┬───────────┬──────────┐       ┌────────┐   GET /changelog/parks/v1        ┌──────────┐
│    │           │          │◀──────│        │◀─────────────────────────────────│          │
│    │           │ RESTful  │       │        │                                  │  Mobile  │
│ DB │   Filby   │   API    │       │  CDN   │                                  │   App    │
│    │           │          │       │        │   GET /projection/parks/v1?id=29 │          │
│    │           │          │◀──────│        │◀─────────────────────────────────│          │
└────┴───────────┴──────────┘       └────────┘                                  └──────────┘
           ▲
           │
           │
┌─────────────────────────────────────────────┐
│                                             │
│               Reference Data                │
│                 Change Sets                 │
│                                             │
└─────────────────────────────────────────────┘
     ▲           ▲           ▲           ▲
     │           │           │           │
     │           │           │           │
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│         │ │         │ │         │ │         │
│   CSV   │ │  YAML   │ │  JSON   │ │   SQL   │
│         │ │         │ │         │ │         │
└─────────┘ └─────────┘ └─────────┘ └─────────┘
</pre>
In the above scenario the first of the two API calls, `/api/changelog` lists the changes sets id and effective dates for the specified projection. The second API call `/api/projection/v1/parks` requestes the park data at a specific point in time. The requests are routed via a CDN for caching. When the data behind the projections a hook causes the cache to be invalidated.
However, it's completely up to you how the projections are accessed - you could build a RESTful API to expose them over HTTP as above, bundle them in a client side JavaScript module or export them as a set of [Apache AVRO](https://avro.apache.org/) files to S3.

### Conclusion

Filby transforms how you manage shared, temporal reference data, combining the best practices of source control with runtime flexibility, no matter how the data is consumed. It isn't a turn key system though, you need to model and provide the data, then write the code calls the Filby API to retrieve the reference data at a point in time and expose it to the outside world.

Ready to explore Filby? [Start here](https://github.com/acuminous/filby).
