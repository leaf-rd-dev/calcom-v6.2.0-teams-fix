# Cal.com v6.2.0 — flat Teams OAuth-ownership fix

This repository publishes a modification made to [Cal.com](https://github.com/calcom/cal.com) version `v6.2.0`, under the terms of the GNU Affero General Public License v3.0 (AGPLv3), which that version is licensed under.

## The bug

`POST /teams` (the flat, non-Organization Teams endpoint in `apps/api/v2`) never records which OAuth client created the team. Specifically, `TeamsController.createTeam()` passes only `(body, user.id)` through to `TeamsService.createTeam()`, which calls `teamsRepository.create(teamData)` without ever setting `createdByOAuthClientId` on the new `Team` row — for any caller, regardless of which credentials authenticated the request.

As a result, `TeamsMembershipsService.canUserBeAddedToTeam()` (used by `POST /teams/:teamId/memberships`) always finds `oAuthClientsRepository.getByTeamId(teamId)` to be `null` for any team created this way, and rejects every attempt to add a second OAuth-managed ("platform managed") user to that team with:

```
Can't add user to team - the user is platform managed user but team is not
because team probably was not created using OAuth credentials.
```

This makes it impossible for a flat team created by an OAuth-managed user to ever hold more than one OAuth-managed member — the team's creator — even though the `Team.createdByOAuthClientId` column exists in the schema for exactly this purpose.

## The fix

`TeamsService.createTeam()` now resolves the creating user's OAuth client via the already-existing `OAuthClientRepository.getByUserId()` and sets `createdByOAuthClientId` on the team before creating it. This is the same linkage the codebase already relies on elsewhere (e.g. the equivalent lookup for a *user's* OAuth client ownership) — it was simply never populated for teams created through this endpoint.

See [`teams-oauth-ownership-fix.patch`](./teams-oauth-ownership-fix.patch) for the full diff against `apps/api/v2/src/modules/teams/teams/services/teams.service.ts`.

## Applying the patch

```
git clone --depth 1 --branch v6.2.0 https://github.com/calcom/cal.com.git
cd cal.com
git apply /path/to/teams-oauth-ownership-fix.patch
```

## License

This patch is a modification of Cal.com, licensed under AGPLv3. It is published here, along with the exact base version it applies to (`calcom/cal.com` at tag `v6.2.0`), to comply with that license's source-availability terms. See [Cal.com's own LICENSE](https://github.com/calcom/cal.com/blob/main/LICENSE) for the full license text.
