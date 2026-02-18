# Collibra Service Provider Documentation

## Overview
The `Collibra Service Provider` is a Profisee Script Service Provider that upserts Collibra governance objects for each input record:

- Domain (created if missing)
- Code Set asset (created if missing)
- Code Value asset (created if missing, updated if existing)
- Code Value -> Code Set relation (created if missing)
- Code Value attributes (Description, Location) via upsert

Service name: `Upsert Assets`  
Service key: `UpsertAssets`

## How To Configure
Configure the service in Profisee Connect with the following inputs.

### Service-scope inputs
- `baseUrl` (String, required): Collibra base URL, for example `https://your-collibra-instance`
- `username` (String, required): Collibra username
- `password` (Credential, required): Collibra password (read in script via `input.password.configuration.value`)
- `connection` (ParameterSet): UI grouping for Base URL / Username / Password

### Data-scope inputs (per record)
- `communityName` (String, required): Target community name (must already exist)
- `domainName` (String, required): Target domain name
- `codeSetName` (String, required): Code Set asset name
- `codeSetId` (String, required by configuration): Value written to Collibra `Id` attribute on a newly created Code Set
- `source` (String, required by configuration): Value written to Collibra `Source` attribute on a newly created Code Set
- `codeValueName` (String, required): Code Value asset name
- `codeValueDisplayName` (String, optional): Code Value display name (used on create and update)
- `description` (String, required by configuration): Upserted to Code Value Description attribute
- `location` (String, required by configuration): Upserted to Code Value Location attribute

## How It Works (Execution Flow)
For each record, the script:

1. Builds a Basic Auth header from `username:password`.
2. Resolves attribute type IDs by name:
- `Source`
- `Id`
3. Resolves Community by exact name. If not found, the record fails.
4. Resolves Domain by exact name within the community.
5. If Domain is missing, creates it using a discovered domain `typeId`.
6. Resolves Code Set by name in the domain.
7. If Code Set is missing, creates it and sets:
- `Source` attribute (if provided)
- `Id` attribute (if provided)
8. Upserts Code Value in the domain:
- If missing: creates asset.
- If found: updates asset via `PATCH /rest/2.0/assets/{assetId}` (name/displayName).
9. Ensures the Code Value -> Code Set relation exists.
10. Upserts Code Value attributes:
- Description
- Location
11. Returns output record values.

## Outputs
Configured outputs:
- `codeValueName` (String): Code Value name
- `codeValueId` (String): Code Value GUID
- `codeUrl` (String): Built as `{baseUrl}/domain/{domainId}`
- `log` (String): Debug log output is configured, but current script output object does not populate `log` per record.

## API Endpoints Used
The script uses Collibra REST 2.0 endpoints including:

- `GET /rest/2.0/communities`
- `GET /rest/2.0/domains`
- `POST /rest/2.0/domains`
- `GET /rest/2.0/assets`
- `POST /rest/2.0/assets`
- `PATCH /rest/2.0/assets/{assetId}`
- `GET /rest/2.0/relations`
- `POST /rest/2.0/relations`
- `GET /rest/2.0/attributes`
- `POST /rest/2.0/attributes`
- `PATCH /rest/2.0/attributes/{attributeId}`
- `GET /rest/2.0/attributeTypes/name/{name}`

## Operational Notes
- Matching is exact-name based for community/domain/asset lookups.
- `maximumRecordsPerRequest` is `100`.
- Worker configuration is single worker (`workerCount = 1`).
- On hard failure, service returns `isSuccess: false` with aggregated log text in `error`.
- Domain and Code Set caching is used within one batch to reduce duplicate lookups.