# Electric bus operations data request

From 2021-01-01 on.


## Scheduled Data
cline_sch.csv

 * week_start: schedules are updated weekly effective Saturday
 * VehicleScheduleID: identifier tied to schedule version and garage
 * RouteID: matches GTFS route_id
 * FromToVia: route variant
 * TripNumber: within route trip number
 * InternalTripNumber: matches GTFS trip_id
 * DutyNumber: duty or run number
 * IsPublic: is this a public trip (in-revenue) or not (deadhead, pull-out, pull-in)
 * TripTypeName: one of Regular, Deadhead, Pull-out or Pull-in
 * BlockNumber: block number
 * OperatingDays: days of week this schedule is operated, 12345 for weekdays, 6 for Saturday, 7 for Sunday. 
 * VehicleTypeName: vehicle assigned to the scheduled trip & block

## AVL Data
stop_crossing.csv

 * calendar_id: service date in format 1%Y%m%d, e.g., 120240501, with a 28 hour schedule service that operates past midnight keeps the same service date, but the time extends. For example, a stop departure at 1:30 am from a block starting on 1 May, 2024 would be assigned the 120240501 calendar_id, and the time 25:30.
 * Schedule: one of WK, SAT, SUN, HOL, RED
 * property_tag: vehicle number publicly visible on the bus, matches GTFS-realtime vehicle_label
 * source_host: vehicle number used internally, matches GTFS-realtime vehicle_id
 * BlockNumber: block
 * InternalTripNumber: matches GTFS trip_id
 * RouteID: matches GTFS route_id
 * RouteDirection: one of N, E, S, W
 * FromToVia: route variant
 * StopID: matches GTFS stop_id
 * ArrivalTime: arrival time in stop-zone, integer seconds since midnight
 * DepartureTime: departure time from stop-zone, integer seconds since midnight
 * ScheduledDepartureTime: scheduled departure time from stop-zone, integer seconds since midnight
 * DoorOpen: time of first door open at stop, integer seconds since midnight
 * DoorClose: time of last door close at stop, integer seconds since midnight
 * StopSequence: trip-stop-sequence
 * a.Cancelled: not-used

## APC Data
apc.csv

 * calendar_id: service date in format 1%Y%m%d, e.g., 120240501, with a 28 hour schedule service that operates past midnight keeps the same service date, but the time extends. For example, a stop departure at 1:30 am from a block starting on 1 May, 2024 would be assigned the 120240501 calendar_id, and the time 25:30.
 * Schedule: one of WK, SAT, SUN, HOL, RED
 * property_tag: vehicle number publicly visible on the bus, matches GTFS-realtime vehicle_label
 * source_host: vehicle number used internally, matches GTFS-realtime vehicle_id
 * BlockNumber: block
 * InternalTripNumber: matches GTFS trip_id
 * RouteID: matches GTFS route_id
 * RouteDirection: one of N, E, S, W
 * FromToVia: route variant
 * StopID: matches GTFS stop_id
 * ArrivalTime: arrival time in stop-zone, integer seconds since midnight
 * DepartureTime: departure time from stop-zone, integer seconds since midnight
 * Boardings: number of passengers boarding
 * Alightings: number of passengers alighting
 * DoorOpen: time of first door open at stop, integer seconds since midnight
 * DoorClose: time of last door close at stop, integer seconds since midnight
