# Module: Geolocation (AWS Location Service)

Adds reverse geocoding and location lookup via AWS Location Service.

---

## Server Additions

### GeolocationService

Core operations:
- Reverse geocode coordinates (latitude/longitude) to an address or place name
- Forward geocode an address string to coordinates
- Batch reverse geocode multiple coordinate pairs

### NuGet Package

```xml
<PackageReference Include="AWSSDK.LocationService" />
```

---

## Infrastructure Additions

### GeolocationStack (new stack)

- Place Index: `{projectname}-place-index-{stage}`
  - Data source: `Esri` (default, good coverage) or `Here` (alternative)
  - Intended use: `SingleUse` (per-request geocoding) or `Storage` (if caching results)
- Expose: `PlaceIndex`, `PlaceIndexName`

Key decisions:
- **Data provider**: Esri (default, free tier generous) vs Here (alternative provider). Each has different coverage strengths
- **Intended use**: `SingleUse` if results are displayed transiently; `Storage` if results are persisted in your database. This affects pricing and terms of service
- **Caching**: Consider caching geocode results in DynamoDB/Postgres to reduce API calls and latency

### CDK Entry Point Update

Add `GeolocationStack` as an independent stack. Pass `PlaceIndexName` to `ApiStack` via props.

### ApiStack Update

- Add `PLACE_INDEX_NAME` Lambda environment variable
- Grant Location Service permissions to the Lambda execution role:
  - `geo:SearchPlaceIndexForPosition` (reverse geocoding)
  - `geo:SearchPlaceIndexForText` (forward geocoding)

---

## File Additions

```
Server/src/Server/
  Services/
    IGeolocationService.cs
    GeolocationService.cs

Infrastructure/
  Stacks/
    GeolocationStack.cs
```
