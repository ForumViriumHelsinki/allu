---
title: Application Types
status: reference
verified_against: e3e24f8ed
verified_on: 2026-09-11
---

# Application Types

Allu manages eight application types (`ApplicationType`). All of them share the
`allu.application` table and differ by the type-specific model serialized into
its `extension` JSONB column, by their fee structure, and by their workflow.

This file is the shared overview. Types with enough depth to need their own
document get one in this directory — currently only
[excavation announcements](excavation-announcement.md).

```mermaid
classDiagram
    class Application {
        +Integer id
        +String applicationId
        +ApplicationType type
        +StatusType status
        +ZonedDateTime startTime
        +ZonedDateTime endTime
        +ApplicationExtension extension
    }

    class ExcavationAnnouncement {
        +Boolean pksCard
        +Boolean selfSupervision
        +ZonedDateTime winterTimeOperation
        +ZonedDateTime workFinished
    }

    class CableReport {
        +ZonedDateTime validityTime
        +Boolean cableSurveyRequired
        +List mapExtracts
    }

    class AreaRental {
        +Integer majorScaffold
        +Double trafficArrangementArea
        +Boolean workFinished
    }

    class PlacementContract {
        +String rationale
        +String section
        +ZonedDateTime contractDate
    }

    class Event {
        +EventNature nature
        +SurfaceHardness surfaceHardness
        +ZonedDateTime eventStartTime
        +ZonedDateTime eventEndTime
    }

    class ShortTermRental {
        +Boolean commercial
        +Boolean recurring
        +ZonedDateTime billableEndTime
    }

    class TrafficArrangement {
        +String trafficArrangements
        +TrafficArrangementImpedimentType impedimentType
    }

    class Note {
        +String description
        +Boolean recurring
    }

    Application <|-- ExcavationAnnouncement : extension
    Application <|-- CableReport : extension
    Application <|-- AreaRental : extension
    Application <|-- PlacementContract : extension
    Application <|-- Event : extension
    Application <|-- ShortTermRental : extension
    Application <|-- TrafficArrangement : extension
    Application <|-- Note : extension
```

## Summary matrix

| ApplicationType | Finnish name | Billable? | Pricing engine | Typical duration | Key specialty |
| --- | --- | --- | --- | --- | --- |
| **`EXCAVATION_ANNOUNCEMENT`** | Kaivuilmoitus | Yes | `ExcavationPricing` | Weeks to years | Two-phase completion (`winterTimeOperation`), 2-year warranty |
| **`CABLE_REPORT`** | Johtoselvitys | **No** (free) | None | 30–60 days | Prerequisite for digging, utility clearances, map sheet orders |
| **`AREA_RENTAL`** | Aluevuokraus | Yes | `AreaRentalPricing` | Days to months | Scaffolds, cranes, skips, containers, multi-period split invoicing |
| **`TEMPORARY_TRAFFIC_ARRANGEMENTS`** | Tilapäinen liikennejärjestely | Yes | Tariff/daily | Days to months | Street/lane closures, detours, bus stop changes, event traffic |
| **`PLACEMENT_CONTRACT`** | Sijoitussopimus | Fixed fee | `PlacementContractPricing` | Permanent | Two-phase contract creation: draft proposal → signed contract |
| **`SHORT_TERM_RENTAL`** | Lyhytaikainen maanvuokraus | Yes | `ShortTermRentalPricing` | Seasonal | Terraces, parklets, food trucks, banners; multi-year recurring |
| **`EVENT`** | Tapahtuma | Yes | `EventPricing` | Days | Surface hardness (asphalt vs grass), event days vs build days |
| **`NOTE`** | Muistiinpano | **No** | None | Ad hoc | Internal bookings: snow dumping, city bikes, election stands |

## Coverage

Only `EXCAVATION_ANNOUNCEMENT` is documented in depth so far, because that is
what the initial investigation needed. The remaining seven are covered by this
matrix alone. When adding one, give it its own file in this directory and link
it from the table above.
