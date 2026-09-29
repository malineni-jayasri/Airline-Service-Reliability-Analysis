# Airline Service Reliability — Technical README

This document explains the implementation behind the [business analysis](README.md).

**Stack:** Microsoft Fabric Lakehouse · PySpark · Delta tables · Direct Lake semantic model · Power BI · DAX

The workflow was run through notebooks. Scheduled orchestration is not part of the documented implementation.

## 1. Source and Architecture

The source is the BTS [Reporting Carrier On-Time Performance dataset](https://www.transtats.bts.gov/TableInfo.asp?QO_fu146_anzr=b0-gvzr&gnoyr_VQ=FGJ).

| Item | Project configuration |
|---|---|
| Reporting period | June 2025–June 2026 |
| Input | 13 monthly CSV files |
| Source grain | One record per reported flight |
| Selected source fields | 52 business fields |
| Total flight records | 7,654,891 |
| Airline coverage | 14 operating airlines |
| Airport dimensions | 358 origin airports and 358 destination airports |
| Fabric workspace | `Airline Service Reliability` |
| Lakehouse | `Airline_Reliability_Lakehouse` |
| Landing folder | `Files/landing/bts_on_time` |
| Semantic model | `Airline_Service_Reliability_Model` |

| Layer | Table | Purpose |
|---|---|---|
| Bronze | `bronze.flight_on_time_raw` | Retain ingested records with source lineage |
| Silver | `silver.flight_on_time_clean` | Standardize types and create analytical eligibility fields |
| Gold | `gold.route_hour_monthly` | Store monthly route-and-hour performance aggregates |
| Gold | `gold.cancellation_monthly` | Store cancellation counts by cause and reporting dimensions |

Fabric notebooks used:

- `01_Bronze_Ingestion_Validation`
- `02_Silver_Profiling_Transformation`
- `03_Gold_Dimensions`

An earlier SQL Server prototype loaded and validated one monthly batch before the full workflow moved to Fabric. The results in these READMEs use the Fabric implementation.

## 2. Bronze Ingestion and Profiling

Bronze contains **7,654,891 rows and 54 columns**: 52 source fields plus `_SOURCE_FILE` and `_INGESTED_AT_UTC` for lineage. Monthly record counts were reconciled before transformation.

Profiling examined missing values, numeric validity, clock values, duplicate records, and the relationship between flight status and missing operational fields.

- No duplicates were found; duplicate removal is not claimed.
- All 33 selected numeric fields had zero invalid nonblank numeric values.
- The seven operational clock fields had zero invalid values under the clock validation rules, including special handling for `2400`.
- Missing arrival fields were investigated by cancellation and diversion status before cleaning.

Blank values were counted separately from invalid values. A value can be valid when present while legitimately missing for a particular flight status.

## 3. Silver Transformations

| Transformation | Treatment |
|---|---|
| Text | Trim surrounding whitespace and convert empty strings to null |
| Flight date | Parse `FL_DATE` into date field `FlightDate` |
| Identifiers, flags, and groups | Cast selected fields to integers |
| Measurements | Cast delay, elapsed-time, and distance fields to doubles |
| Operational clock values | Add integer minute-of-day fields; map `2400` to `0` |
| Early arrivals and departures | Preserve negative delay values |
| Route | Combine origin and destination codes |
| Scheduled departure hour | Derive from scheduled departure minutes |
| Flight status | Classify as Cancelled, Diverted, Incomplete, or Completed |
| Arrival eligibility | Require a non-cancelled, non-diverted flight with arrival time and delay present |
| Quality flag | Flag the incomplete arrival as `MissingArrivalData` |

Minute-of-day fields were added for:

`CRS_DEP_TIME`, `DEP_TIME`, `WHEELS_OFF`, `WHEELS_ON`, `CRS_ARR_TIME`, `ARR_TIME`, and `FIRST_DEP_TIME`.

For an ordinary HHMM value, the conversion is `hour × 60 + minute`. These are clock positions, not elapsed durations. Mapping `2400` to midnight does not preserve next-day context for duration calculations.

Conditional nulls in diversion, cancellation, and delay-cause fields were retained. Replacing all such nulls with zero would erase the distinction between missing or inapplicable information and a reported zero.

One non-cancelled, non-diverted flight lacked sufficient arrival information. It remains in scheduled-flight totals but is excluded from arrival-performance calculations.

## 4. Gold Tables and Semantic Model

### Performance fact

`gold.route_hour_monthly` contains **548,014 rows and 26 columns**.

**Grain:** month × operating airline × origin airport × destination airport × scheduled departure hour.

| Column group | Fields |
|---|---|
| Grouping fields — 7 | `Month`, `OP_UNIQUE_CARRIER`, `ORIGIN_AIRPORT_ID`, `ORIGIN`, `DEST_AIRPORT_ID`, `DEST`, `ScheduledDepartureHour` |
| Flight counts — 7 | `ScheduledFlights`, `ArrivalEligibleFlights`, `OnTimeArrivals`, `DelayedArrivals`, `CancelledFlights`, `DivertedFlights`, `IncompleteArrivalFlights` |
| Additional delay measures — 2 | `DelayedArrivalMinutes`, `FlightsWithLateAircraftDelay` |
| Cause minutes — 5 | `CarrierDelayMinutes`, `WeatherDelayMinutes`, `NASDelayMinutes`, `SecurityDelayMinutes`, `LateAircraftDelayMinutes` |
| Reported-value counts — 5 | `CarrierReportedFlights`, `WeatherReportedFlights`, `NASReportedFlights`, `SecurityReportedFlights`, `LateAircraftReportedFlights` |

A reported-value count includes nonmissing zero values. It is different from counting flights with a positive delay attributed to that cause. For example, `LateAircraftReportedFlights` and `FlightsWithLateAircraftDelay` answer different questions.

### Cancellation fact

`gold.cancellation_monthly` aggregates cancellations by **month, airline, origin airport, destination airport, scheduled departure hour, and cancellation cause**, with `CancelledFlights` as the count.

| BTS code | Cause label |
|---|---|
| A | Air carrier |
| B | Weather |
| C | National Aviation System |
| D | Security |
| Unmapped or missing | Unknown, retained for quality review |

The summed cancellation count is **137,060 across 13 months**. This is the number of cancelled flights, not the number of aggregate rows.

### Relationships

Each relationship is active, one-to-many, with a single filter direction from the dimension to the fact. Apply these five mappings to **both** Gold facts.

| Dimension column — one side | Fact column — many side |
|---|---|
| `dim_month[Month]` | `Month` |
| `dim_airline[AirlineCode]` | `OP_UNIQUE_CARRIER` |
| `dim_origin_airport[AirportID]` | `ORIGIN_AIRPORT_ID` |
| `dim_destination_airport[AirportID]` | `DEST_AIRPORT_ID` |
| `dim_hour[ScheduledDepartureHour]` | `ScheduledDepartureHour` |

Origin and destination use separate airport dimensions. The month dimension covers the 13 reporting months; the hour dimension supplies hour labels and departure-period categories. Reuse the existing departure-period definitions consistently across notebook and report comparisons.

The facts have no direct relationship. Shared-dimension slicers filter both facts. A cancellation-cause selection on the cancellation fact does not automatically filter the performance fact or its measures.

## 5. KPI Definitions and DAX

| KPI | Definition |
|---|---|
| Arrival OTP | On-time arrivals / arrival-eligible flights |
| On-time arrival | Eligible arrival less than 15 minutes late |
| Delayed arrival | Eligible arrival at least 15 minutes late |
| Cancellation rate | Cancelled flights / scheduled flights |
| Diversion rate | Diverted flights / scheduled flights |
| Late-aircraft minutes per 100 flights | Late-aircraft minutes / scheduled flights × 100 |
| Reported delay-cause share | Minutes for one cause / minutes across all five reported causes in the same context |
| Cancellation-cause share | Cancellations for one cause / cancellations across all causes under the same other filters |

Rates are calculated from **summed counts**, not by averaging percentages from Gold rows.

Create each expression below as a separate measure:

```dax
Total Flights =
SUM(route_hour_monthly[ScheduledFlights])

Arrival Eligible Flights =
SUM(route_hour_monthly[ArrivalEligibleFlights])

On-Time Arrivals =
SUM(route_hour_monthly[OnTimeArrivals])

Arrival OTP % =
DIVIDE([On-Time Arrivals], [Arrival Eligible Flights])

Cancelled Flights =
SUM(route_hour_monthly[CancelledFlights])

Cancellation Rate % =
DIVIDE([Cancelled Flights], [Total Flights])

Diverted Flights =
SUM(route_hour_monthly[DivertedFlights])

Diversion Rate % =
DIVIDE([Diverted Flights], [Total Flights])

Late Aircraft Delay Minutes =
SUM(route_hour_monthly[LateAircraftDelayMinutes])

Late Aircraft Minutes per 100 Flights =
DIVIDE([Late Aircraft Delay Minutes] * 100, [Total Flights])

Cancellations by Cause =
SUM(cancellation_monthly[CancelledFlights])
```

Format the ratio measures as percentages; do not also multiply those measures by 100. The minutes-per-100 measure is a numeric value, not a percentage.

Use `Cancellations by Cause` for cancellation-cause visuals. The existing `Cancelled Flights` measure reads the performance fact and will not respond to a cause filter on the cancellation fact. A cause-specific cancellation rate would require a cause-aware numerator and the intended scheduled-flight denominator.

## 6. Validation Results

All **19 additive measures** matched between Silver and Gold. The saved Gold performance table was checked again, confirming **548,014 rows and 26 columns**.

| Measure | Reconciled total |
|---|---:|
| ScheduledFlights | 7,654,891 |
| ArrivalEligibleFlights | 7,495,755 |
| OnTimeArrivals | 5,780,303 |
| DelayedArrivals | 1,715,452 |
| CancelledFlights | 137,060 |
| DivertedFlights | 22,075 |
| IncompleteArrivalFlights | 1 |
| DelayedArrivalMinutes | 125,493,023 |
| FlightsWithLateAircraftDelay | 891,070 |
| CarrierDelayMinutes | 41,461,767 |
| CarrierReportedFlights | 1,715,452 |
| WeatherDelayMinutes | 7,899,215 |
| WeatherReportedFlights | 1,715,452 |
| NASDelayMinutes | 26,092,496 |
| NASReportedFlights | 1,715,452 |
| SecurityDelayMinutes | 179,697 |
| SecurityReportedFlights | 1,715,452 |
| LateAircraftDelayMinutes | 49,841,177 |
| LateAircraftReportedFlights | 1,715,452 |

Additional reconciliations:

- On-time arrivals + delayed arrivals = arrival-eligible flights.
- Eligible arrivals + cancellations + diversions + incomplete arrivals = scheduled flights.
- The cancellation fact sums to 137,060 cancellations across 13 months.
- January’s cause counts reconcile: 22,463 weather + 2,246 carrier + 879 NAS + 47 security = 25,635 cancellations.

Unfiltered report checks: **77.11% arrival OTP**, **1.79% cancellation rate**, **0.29% diversion rate**, and **651.10 late-aircraft minutes per 100 flights**.

<details>
<summary>Monthly record-count baseline</summary>

| Month | Flight records |
|---|---:|
| June 2025 | 611,575 |
| July 2025 | 631,428 |
| August 2025 | 602,378 |
| September 2025 | 562,439 |
| October 2025 | 605,844 |
| November 2025 | 570,550 |
| December 2025 | 582,304 |
| January 2026 | 544,003 |
| February 2026 | 515,037 |
| March 2026 | 612,102 |
| April 2026 | 597,919 |
| May 2026 | 611,735 |
| June 2026 | 607,577 |
| **Total** | **7,654,891** |

</details>

## 7. Report Context and Interpretation Checks

The report contains Executive Overview, Delay Drivers, Routes & Departure Times, and Cancellation Analysis pages. Use shared-dimension fields for month, airline, airport, and departure-time slicers.

| Finding | Required context |
|---|---|
| July arrival OTP: 71.11% | All airlines and airports; July 2025 |
| Late aircraft: 40.79% of reported cause minutes | All airlines and airports; July 2025 |
| DEN late-aircraft minutes: 417,588 | Origin DEN; all airlines; July 2025 |
| Morning 85.71% vs evening 23.46% OTP | Southwest; DEN–SLC; July 2025 |
| Cancellation rate: 4.71% | All airlines and airports; January 2026 |
| Weather: 87.63% of cancellations | All airlines and airports; January 2026 |

The Southwest route comparison contains 77 scheduled morning flights and 83 scheduled evening flights. Two evening flights were cancelled, so the scheduled-flight count is not the arrival-OTP denominator.

A chart ranking cancellation counts by departure period does not rank cancellation probability. Airport cancellation rates also require scheduled and cancelled flight counts before setting operational priorities; that volume review remains open.

## 8. Reproduction Workflow and Limits

These two README files document the project. They do not include notebook exports, source CSV files, or a deployable report package.

To reproduce the implementation with those assets:

1. Land the 13 source files and reconcile their row counts.
2. Run Bronze ingestion and profiling; retain lineage and profiling outputs.
3. Run Silver transformations and verify eligibility, statuses, and data types.
4. Build both Gold facts and shared dimensions; reconcile aggregate totals to Silver and verify the saved tables.
5. Create the Direct Lake model, relationships, and measures described above.
6. Build the four report pages and validate totals, filter interactions, and the scoped findings.

Important limits:

- Minute-of-day values alone cannot calculate elapsed time across midnight or time zones.
- Reported cause minutes and delayed-arrival minutes are distinct measures; do not assume their totals must match.
- Monthly aggregates cannot reconstruct an aircraft’s flight sequence. Further operational investigation requires flight-level records and appropriate sequencing data.
- Reported categories do not establish the complete causal chain. See the BTS [guide to delay and cancellation reporting](https://www.bts.gov/topics/airlines-and-airports/understanding-reporting-causes-flight-delays-and-cancellations).
- Thirteen months provide limited evidence of recurring seasonality. Passenger counts, disruption costs, and rebooking outcomes are outside this extract.
- Recommendations are hypotheses for review or testing. Operational adoption and measured improvements are not claimed.

Return to the [Business README](README.md) for the project story, findings, and stakeholder recommendations.
