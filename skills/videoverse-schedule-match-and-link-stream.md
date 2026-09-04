---
name: videoverse-schedule-match-and-link-stream
description: Build a tournament/team/player roster, create a match schedule from it, and link or unlink a stream to that fixture.
generated: '2026-09-04'
method: generated
source: openapi/videoverse-magnifi-partner-openapi.yml (derived from https://docs.prod.videoverse.dev/)
api: Magnifi Partner Integration API
operations:
  - addPlayer
  - addTeam
  - addTournament
  - createMatchSchedule
  - getAllMatchSchedules
  - getMatchScheduleByMatchScheduleId
  - updateMatchScheduleByMatchScheduleId
  - invokeStreamByMatchScheduleId
  - deLinkByMatchScheduleId
  - deleteMatchScheduleByMatchScheduleId
---

# Schedule a match and link a stream to it

Roster data is what makes player tagging work: it is reflected in the metadata of clips and
highlight clips. Build it bottom-up.

## 1. Players

`addPlayer` (`POST /v1/partner/player`) with `category`, `playerName` and free-form `metaData`
(e.g. `jerseyNumber`). Returns the player `id`. `getAllPlayers`, `getPlayerByPlayerId` and
`updatePlayerByPlayerId` round it out. There is no delete.

## 2. Teams

`addTeam` (`POST /v1/partner/team`) with `category`, `teamName`, `players` (an array of player ids)
and `metaData` (e.g. `shortName`). The response embeds the full player objects and returns both
`id` and `teamId`. Use `updateTeamByTeamId` to change the roster. There is no delete.

## 3. Tournaments

`addTournament` (`POST /v1/partner/tournament`) with `category`, `tournamentName`, `teams` (an array
of team ids) and `metaData`. Categories must line up: `MS005` / `S008` reject a tournament whose
category differs from the match or stream category, and `MS006` / `S007` reject a team that is not
part of the tournament.

## 4. The fixture

`createMatchSchedule` (`POST /v1/match-schedule`) with `name`, `matchStartTime`, `timezone`,
`teams` (each `{teamId, type: HOME|AWAY}`), `tournament` and `category`. Returns `msh_…` as `id`
plus a generated `ext_match_…` external id, with `streamId` and `streamUrl` still `null`.

`MS007` means that external match id already exists — resolve it with `getAllMatchSchedules`
(`startTime` / `endTime` window) rather than retrying blind. There is no idempotency key.

## 5. Link, unlink, delete

- `invokeStreamByMatchScheduleId` (`POST /v1/match-schedule/{matchScheduleId}/invoke-stream`)
  starts a stream for the fixture. `S009` guards one stream per `matchScheduleId`.
- `deLinkByMatchScheduleId` (`POST /v1/match-schedule/{matchScheduleId}/de-link`) is the documented
  reversal for that link. **The documentation states no window** — it does not say whether de-link
  works after the stream has started or completed. Treat that as unknown, not as unlimited.
- `deleteMatchScheduleByMatchScheduleId` (`DELETE /v1/match-schedule/{matchScheduleId}`) removes the
  fixture. Also no stated window.

Remember that the stream itself, once created, has no published delete.
