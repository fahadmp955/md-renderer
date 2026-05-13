# Football Market App Working Document

## 1. Application Overview

This application is a football prediction market platform with:

- A React + Vite frontend
- An Express server
- A tRPC API mounted at `/api/trpc`
- MySQL access via Drizzle ORM
- Privy-based authentication
- Scheduled sync/oracle jobs for football fixtures and player stats

At a high level:

1. The browser loads the React app from `client/src/main.tsx`.
2. The React app initializes `PrivyProvider`, `QueryClientProvider`, and the tRPC client.
3. The server starts from `server/_core/index.ts`.
4. The Express server mounts:
   - `POST /api/auth/*` auth routes
   - `GET /api/oracle/*` export/report routes
   - `/api/trpc` for all app procedures
5. On startup, the server also starts background schedulers for sync, fixture linking, market auto-lock, auto-resolve, and weekly maintenance.

## 2. What Happens On Start

### 2.1 Frontend start

Entrypoint: `client/src/main.tsx`

Libraries/services initialized on frontend start:

- `react` / `react-dom`
- `@tanstack/react-query` via `QueryClient`
- `@trpc/client` + `@trpc/react-query`
- `superjson` for tRPC serialization
- `@privy-io/react-auth` via `PrivyProvider`
- Global CSS from `client/src/index.css`

Frontend boot flow:

1. Reads `VITE_PRIVY_APP_ID`.
2. Creates a React Query client.
3. Creates a tRPC client using `httpBatchLink` pointed at `/api/trpc`.
4. Configures `fetch(..., { credentials: "include" })` so auth cookies are sent.
5. Wraps the app with:
   - `PrivyProvider`
   - `trpc.Provider`
   - `QueryClientProvider`
6. Renders `PrivySessionSync`, which exchanges the Privy access token for the app's own session cookie by calling `POST /api/auth/privy`.

### 2.2 Backend start

Entrypoint: `server/_core/index.ts`

Libraries/services initialized on backend start:

- `dotenv/config` for env loading
- `express`
- Node `http` server
- `@trpc/server` Express adapter
- Auth routes from `server/_core/routes/auth.routes.ts`
- tRPC router from `server/routers.ts`
- Oracle export routes from `server/oracleExport.ts`
- Vite dev middleware in development, static serving in production

Backend boot flow:

1. Loads `.env`.
2. Creates the Express app and HTTP server.
3. Adds JSON/urlencoded parsers with `50mb` limits.
4. Mounts:
   - `/api/auth`
   - `/api/oracle/...`
   - `/api/trpc`
5. Uses Vite middleware in development or static serving in production.
6. Starts listening on `PORT` (default `3000`).
7. After listen succeeds, it runs startup jobs:
   - `cleanupStaleSyncLogs()`
   - `ensureAdminRoles()`
   - `migrateMarketsToStatBased()`
8. Starts schedulers:
   - `startSyncScheduler()` in production only
   - `startAutoLockScheduler()`
   - `startAutoResolveScheduler()`
   - `startFixtureSyncScheduler()`
   - `startWeeklyLibraryScheduler()`

### 2.3 Authentication/session start

Relevant files:

- `client/src/components/PrivySessionSync.tsx`
- `server/_core/routes/auth.routes.ts`
- `client/src/_core/hooks/useAuth.ts`

Flow:

1. Privy authenticates the user in the frontend.
2. `PrivySessionSync` gets the Privy access token.
3. Frontend calls `POST /api/auth/privy` with `{ token: string }`.
4. Backend verifies the token using `@privy-io/server-auth`.
5. Backend upserts the user in MySQL.
6. Backend signs its own JWT and sets the session cookie.
7. Frontend then uses `trpc.auth.me` to fetch the authenticated user.

## 3. Environment Variables

### 3.1 Variables currently present in `.env`

| Variable | Used for |
| --- | --- |
| `DATABASE_URL` | MySQL connection string used by Drizzle and scripts |
| `FOOTBALL_API_KEY` | API-Football key used for stats sync, fixtures sync, oracle fetches, and scripts |
| `JWT_SECRET` | Signs/verifies the app's JWT session cookie |
| `NODE_ENV` | Switches dev vs production behavior and controls scheduler startup |
| `OWNER_OPEN_ID` | Auto-promotes the configured owner account to admin |
| `PRIVY_APP_ID` | Backend Privy token verification |
| `PRIVY_APP_SECRET` | Backend Privy token verification secret |
| `VITE_APP_ID` | Used in `client/src/const.ts` when building login URLs |
| `VITE_PRIVY_APP_ID` | Frontend Privy app id |
| `OAUTH_SERVER_URL` | Referenced by backend OAuth/sdk code |
| `PORT` | Backend listen port |
| `MYSQL_*` | Present for DB environment/reference; app code primarily uses `DATABASE_URL` |

### 3.2 Variables referenced in code but missing from current `.env`

| Variable | Used for |
| --- | --- |
| `VITE_OAUTH_PORTAL_URL` | Used by `client/src/const.ts` for generated login URL |
| `BUILT_IN_FORGE_API_URL` | Used by storage/map/image-generation/data API helpers |
| `BUILT_IN_FORGE_API_KEY` | Used by storage/map/image-generation/data API helpers |
| `VITE_FRONTEND_FORGE_API_KEY` | Used by `client/src/components/Map.tsx` |
| `VITE_FRONTEND_FORGE_API_URL` | Used by `client/src/components/Map.tsx` |

## 4. API Architecture

The UI mainly talks to the backend through tRPC:

- Transport endpoint: `/api/trpc`
- Serialization: `superjson`
- Auth: cookie-based session

Non-tRPC routes:

- `POST /api/auth/privy`
- `POST /api/auth/logout`
- `GET /api/oracle/export.csv`
- `GET /api/oracle/export.pdf`
- `GET /api/oracle/market/:marketId/report.pdf`

## 5. Frontend Pages and URLs

### 5.1 Public/user pages

| URL | Page | Notes |
| --- | --- | --- |
| `/` | Home | Main trading page; internally shows Players, Markets, Roster tabs |
| `/?tab=players` | Home > Players tab | Default tab |
| `/?tab=markets` | Home > Markets tab | Can also use `expand` and `soon` query params |
| `/?tab=roster` | Home > Roster tab | Requires auth for actual data |
| `/leaderboard` | Leaderboard | Public rankings, plus logged-in user's rank |
| `/404` | NotFound | Fallback |

### 5.2 Admin pages

| URL | Page |
| --- | --- |
| `/admin` | AdminDashboard |
| `/admin/users` | AdminUsers |
| `/admin/limits` | AdminMarketLimits |
| `/admin/markets/new` | AdminCreateMarket |
| `/admin/markets/:id/edit` | AdminEditMarket |
| `/admin/players` | AdminPlayers |
| `/admin/settings` | AdminSettings |
| `/admin/audit` | AdminAudit |
| `/admin/sync-logs` | AdminSyncLogs |
| `/admin/sync` | AdminSync |
| `/admin/oracle` | AdminOracle |

## 6. Page-Wise Functionality and APIs

## 6.1 `/` Home shell

Shared UI features:

- Top ticker with clickable markets
- Wallet/auth menu
- Notification preference toggle
- Platform stats sidebar
- Tabbed navigation for Players / Markets / Roster

APIs used by shell components:

### `auth.me`

- Procedure: `trpc.auth.me`
- Request: none
- Response: `null | { id, openId, email, name, role, usdcBalance?, notificationsEnabled?, ... }`
- Used by: auth state, wallet display

### `markets.ticker`

- Procedure: `trpc.markets.ticker`
- Request: none
- Response: `Array<{ id, playerId, statType, marketBucket, overPrice, underPrice, totalVolume, homeTeam, awayTeam, kickoffAt, playerName, playerTeam, playerPosition, playerCountryCode }>`
- Used by: moving top ticker

### `roster.getNotificationPrefs`

- Procedure: `trpc.roster.getNotificationPrefs`
- Request: none
- Response: `{ notificationsEnabled: boolean }`

### `roster.setNotificationPrefs`

- Procedure: `trpc.roster.setNotificationPrefs`
- Request: `{ notificationsEnabled: boolean }`
- Response: `{ success: true }`

### `markets.platformStats`

- Procedure: `trpc.markets.platformStats`
- Request: none
- Response: `{ openMarkets, liveCount, totalPlayers, totalLines, leagueCounts: Array<{ league, playerCount }>, nextKickoffAt }`

## 6.2 `/` Home > Players tab

Main features:

- League filter
- Search by player/team
- Team filter
- Position filter
- "Closing soon" filter
- Sort by rating or name
- Shows live badges when a draft oracle report exists

APIs:

### `players.withMarkets`

- Request: `{ league?: string }`
- Response: `Array<{ id, name, team, position, league, countryCode, rollingAverage, last10Scores, last10Stats?, imageUrl?, primaryStat?, secondaryStat?, markets: Array<open market rows> }>`

### `players.lastResolved`

- Request: none
- Response: `Record<number, Date | null>` keyed by `playerId`

### `oracle.liveMarkets`

- Request: none
- Response: `Array<{ marketId, actualStatValue, statType, marketBucket, playerName }>`

## 6.3 `/` Home > Markets tab

Main features:

- League-based market browsing
- Expand a player row to see stat markets
- View price history chart
- View order book depth
- Place market orders
- Place limit orders
- Set/cancel price alerts
- See live oracle value if admin already fetched stat

APIs:

### `markets.allWithPlayers`

- Request: `{ league?: string }`
- Response: `Array<market + player joined rows>`

### `oracle.liveMarkets`

- Request: none
- Response: same as above

### `markets.orderBookDepth`

- Request: `{ marketId: number }`
- Response: `{ overBids: Array<{ id, limitPrice, remainingQty, side }>, overAsks: Array<...>, underBids: Array<...>, underAsks: Array<...> }`

### `roster.listAlerts`

- Request: none
- Response: `Array<{ id, marketId, side, targetPriceCents, direction, status, expiresAt?, createdAt }>`

### `roster.setAlert`

- Request: `{ marketId, side: "over" | "under", targetPriceCents: number, direction: "above" | "below", expiresAt?: ISODateString }`
- Response: `{ success: true }`

### `roster.cancelAlert`

- Request: `{ alertId: number }`
- Response: `{ success: true }`

### `roster.placeMarketOrder`

- Request: `{ marketId, side: "over" | "under", limitPrice: number, amountUsd: number }`
- Response: `{ success: true, orderId, status, filledQty, avgFillPrice, takerFee, newBalance }`

### `markets.priceHistory`

- Request: `{ marketId: number }`
- Response: `Array<{ marketId, overPrice, underPrice, recordedAt }>`

### `oracle.reportByMarket`

- Request: `{ marketId: number }`
- Response: `null | { marketId, playerId, statType, marketBucket, actualStatValue, opponent, matchDate, source, resolvedAt, isDraft?, ... }`

### `roster.balance`

- Request: none
- Response: `number`

### `roster.placeLimitOrder`

- Request: `{ marketId, side: "over" | "under", limitPrice: number, amountUsd: number }`
- Response: `{ success: true, orderId, newBalance }`

## 6.4 `/` Home > Roster tab

Main features:

- Shows active positions
- Shows account value / allocated / available balance
- Open orders list and cancellation
- Price alert history and triggered alerts
- Fill history
- Settlement notifications
- Close position (sell back before lock window)
- Toggle settlement notifications
- Fee tier summary

APIs:

### `roster.positions`

- Request: none
- Response: `Array<{ position, playerName, playerTeam, playerPosition, playerCountryCode, playerRollingAverage, marketStatType, marketBucket, marketHomeTeam, marketAwayTeam, marketKickoffAt }>`

### `roster.balance`

- Request: none
- Response: `number`

### `roster.openOrders`

- Request: `{ marketId?: number }`
- Response: `Array<{ id, marketId, side, orderType, limitPrice, quantity, remainingQty, status, feeBps, createdAt }>`

### `roster.listAlerts`

- Request: none
- Response: active/cancelled alert rows for the user

### `roster.cancelAlert`

- Request: `{ alertId: number }`
- Response: `{ success: true }`

### `roster.fillHistory`

- Request: `{ limit: number }`
- Response: `Array<fill history rows joined with market/player info>`

### `roster.pendingAlertNotifications`

- Request: none
- Response: `Array<triggered alert notification rows>`

### `roster.markAlertsSeen`

- Request: `{ alertIds: number[] }`
- Response: `{ success: true }`

### `roster.pendingSettlementNotifications`

- Request: none
- Response: `Array<settled position rows awaiting toast acknowledgement>`

### `roster.markSettlementNotificationsSeen`

- Request: `{ positionIds: number[] }`
- Response: `{ success: true }`

### `roster.closePosition`

- Request: `{ positionId: number }`
- Response: `{ success: true, exitValue: number, exitPrice: number }`

### `roster.toggleSettlementNotify`

- Request: `{ positionId: number, notify: boolean }`
- Response: `{ success: true }`

### `roster.cancelOrder`

- Request: `{ orderId: number }`
- Response: `{ success: true, newBalance: number }`

### `roster.positionPriceHistory`

- Request: `{ marketId: number }`
- Response: `Array<{ marketId, overPrice, underPrice, recordedAt }>`

### `fees.myTier`

- Request: none
- Response: `{ thirtyDayVolume, takerBps, makerBps, tierIndex, tierCount, nextTierAt, nextTierTakerBps, nextTierMakerBps }`

### `fees.schedule`

- Request: none
- Response: `Array<{ minVolume, takerBps, makerBps, takerPct, makerPct }>`

## 6.5 `/leaderboard`

Main features:

- Public rankings
- Sort by P&L / win rate / wagered / settled
- Filter by time window
- Search by trader name
- Tier 1+ filter
- Infinite pagination/load more
- Shows logged-in user's true rank even if off-page

APIs:

### `leaderboard.rankings`

- Request: `{ limit?: number, offset?: number, timeFilter?: "week" | "month" | "all" }`
- Response: `Array<{ userId, username, pnl, winRate, totalWagered?, totalCost?, settledCount, tierIndex?, ... }>`

### `leaderboard.myRank`

- Request: none
- Response: `null | { userId, username, rank, pnl, winRate, totalWagered, settledCount, tierIndex?, ... }`

## 6.6 `/admin`

Main features:

- Overall platform metrics
- Recent trades
- DB health counts
- API quota/last sync visibility
- Navigation into sync/oracle/admin tools

APIs:

### `admin.dashboardMetrics`

- Request: none
- Response: `{ markets: { open, resolved, ... }, positions: { active, settled }, totalVolume, totalFeesCollected, users, recentTrades, ... }`

### `admin.dbRowCounts`

- Request: none
- Response: table counts per major DB entity

### `footballSync.apiStatus`

- Request: none
- Response: `{ plan, requestsToday, requestsLimit, active, error? }`

### `footballSync.lastSync`

- Request: none
- Response: `null | { id, type, status, startedAt, completedAt, playersUpdated, marketsUpdated, apiRequestsUsed, details }`

## 6.7 `/admin/users`

Main features:

- Search/sort users
- View balances, role, positions, open orders, created date, last sign-in
- Promote user to admin
- Demote admin to user

APIs:

### `admin.listUsers`

- Request: none
- Response: `Array<{ id, name, email, role, usdcBalance, positionCount, openOrderCount, createdAt, lastSignedIn }>`

### `admin.promoteUser`

- Request: `{ userId: number }`
- Response: `{ success: true }`

### `admin.demoteUser`

- Request: `{ userId: number }`
- Response: `{ success: true }`

## 6.8 `/admin/limits`

Main features:

- View markets and their current max position limits
- Filter by status
- Set/remove limit per market

APIs:

### `admin.marketsWithLimits`

- Request: `{ status?: "all" | "open" | "closed" | "settled" | "resolved" }`
- Response: `Array<{ marketId, threshold, status, totalVolume, maxPositionUsd, playerName, playerTeam }>`

### `admin.setMarketLimit`

- Request: `{ marketId: number, limitUsd: number | null }`
- Response: `{ success: true }`

## 6.9 `/admin/markets/new`

Main features:

- Choose player
- Pull upcoming fixtures by league
- Inspect existing markets for player
- Create a full stat-market set for the player

APIs:

### `players.list`

- Request: `{ league?: string }`
- Response: `Array<active player rows>`

### `statMarkets.upcomingFixtures`

- Request: `{ league: string }`
- Response: `Array<{ apiFixtureId, leagueId, homeTeam, awayTeam, kickoffAt, status, round }>`

### `statMarkets.forPlayer`

- Request: `{ playerId: number }`
- Response: `Array<open stat market rows for player>`

### `statMarkets.createForPlayer`

- Request: `{ playerId, fixtureId?, kickoffAt?, homeTeam?, awayTeam?, matchDate?, maxPositionUsd?, bucketPrices?: Record<string, number> }`
- Response: `{ success: true, marketsCreated: number, marketIds: number[] }`

## 6.10 `/admin/markets/:id/edit`

Main features:

- Edit threshold
- Edit opening price
- Edit max position limit

APIs:

### `admin.getMarket`

- Request: `{ marketId: number }`
- Response: `market row joined with player info`

### `admin.updateMarket`

- Request: `{ marketId, threshold?, overPriceCents?, maxPositionUsd? }`
- Response: `{ success: true }`

## 6.11 `/admin/players`

Main features:

- View active players and market counts
- View/import inactive players library
- Bulk import players by league from API-Football
- Activate inactive players and optionally create default markets
- Edit player fields
- Toggle player active/inactive
- Toggle individual markets open/locked
- Bulk activate/deactivate whole league
- Upload player image
- Audit log by player/market

APIs:

### `admin.listPlayers`

- Request: none
- Response: `Array<player rows + marketCount + openMarketCount>`

### `admin.getPlayerWithMarkets`

- Request: `{ playerId: number }`
- Response: `{ player, markets: Array<market rows> }`

### `admin.updatePlayer`

- Request: `{ playerId, name?, team?, position?, league?, countryCode?, imageUrl?, rollingAverage?, primaryStat?, secondaryStat?, isActive? }`
- Response: `{ success: true }`

### `admin.toggleMarket`

- Request: `{ marketId: number }`
- Response: `{ success: true, status: "open" | "locked" }`

### `admin.getEntityAuditLog`

- Request: `{ entityType: "player" | "market" | "setting", entityId: number | null }`
- Response: `Array<audit log rows>`

### `admin.createMarket`

- Request: `{ playerId, threshold, overPriceCents, maxPositionUsd?, statType?, marketBucket? }`
- Response: `{ success: true, marketId }`

### `admin.updateMarketFull`

- Request: `{ marketId, statType?, marketBucket?, overPriceCents?, kickoffAt?, homeTeam?, awayTeam?, maxPositionUsd? }`
- Response: `{ success: true }`

### `playerLibrary.list`

- Request: `{ search?, league?: "laliga" | "premier_league" | "bundesliga" | "serie_a" | "all", limit?, offset? }`
- Response: inactive players page result

### `playerLibrary.getImportProgress`

- Request: `{ league: "laliga" | "premier_league" | "bundesliga" | "serie_a" }`
- Response: `null | { status: "fetching" | "importing" | "done", teamsTotal, teamsProcessed, playersInserted, playersSkipped }`

### `playerLibrary.importLeague`

- Request: `{ league: "laliga" | "premier_league" | "bundesliga" | "serie_a" }`
- Response: `{ success: true, inserted, skipped, errors, teamsProcessed }`

### `playerLibrary.activate`

- Request: `{ playerId, createMarkets?: boolean, overrides?: { primaryStat?, rollingAverage?, imageUrl? } }`
- Response: `{ success: true, player, marketsCreated }`

### `admin.togglePlayer`

- Request: `{ playerId: number }`
- Response: `{ success: true, isActive: boolean }`

### `admin.bulkToggleLeague`

- Request: `{ league: string, activate: boolean }`
- Response: `{ success: true, affected: number }`

### `admin.uploadPlayerImage`

- Request: `{ playerId: number, dataUri: string }`
- Response: `{ url: string }`

## 6.12 `/admin/settings`

Main features:

- Update sell-back spread
- Update taker fee / maker rebate
- Update auto-lock minutes
- Update sync log retention
- Manually trigger auto-lock / auto-resolve
- View settings audit history
- Notify owner on setting changes

APIs:

### `admin.getSettings`

- Request: none
- Response: `Record<string, string>`

### `admin.setSetting`

- Request: `{ key: string, value: string }`
- Response: `{ success: true }`

### `system.notifyOwner`

- Request: `{ title: string, content: string }`
- Response: `{ success: boolean }`

### `admin.getAutoLockStatus`

- Request: none
- Response: `{ autoLockMinutes: number, lastRun: { locked: number, lockedAt: Date } | null }`

### `admin.triggerAutoLock`

- Request: none
- Response: `{ locked: number, lockedAt: Date }`

### `admin.getAutoResolveStatus`

- Request: none
- Response: `{ lastRun: { resolved, voided, errors, ranAt } | null }`

### `admin.triggerAutoResolve`

- Request: none
- Response: `{ resolved, voided, errors, ranAt }`

### `fees.schedule`

- Request: none
- Response: fee tier schedule

### `admin.getAuditLog`

- Request: none
- Response: `Array<settings audit log rows>`

## 6.13 `/admin/audit`

Main features:

- Search/filter audit logs across players, markets, settings
- Pagination
- Export audit logs

APIs:

### `admin.getAllAuditLogs`

- Request: `{ entityType?: "player" | "market" | "setting" | "all", search?: string, limit?: number, offset?: number }`
- Response: `{ rows: Array<audit rows>, total: number }`

### `admin.exportAuditLog`

- Request: `{ entityType?: "player" | "market" | "setting" | "all", search?: string }`
- Response: `{ rows: Array<audit rows> }`

## 6.14 `/admin/sync-logs`

Main features:

- View recent sync log history
- Filter by sync type

APIs:

### `admin.getSyncLogs`

- Request: `{ type?: "fixture_sync" | "player_stats" | "all", limit?: number }`
- Response: `{ rows: Array<sync log rows> }`

## 6.15 `/admin/sync`

Main features:

- See API quota status
- See last sync state
- Preview player-to-API mappings
- Trigger player stats sync
- Trigger fixture sync
- Trigger stat history refresh script
- Wipe test data
- Inspect recent sync log entries

APIs:

### `footballSync.apiStatus`

- Request: none
- Response: `{ plan, requestsToday, requestsLimit, active, error? }`

### `footballSync.recentLogs`

- Request: none
- Response: `Array<recent sync log rows>`

### `footballSync.lastSync`

- Request: none
- Response: last successful `player_stats` sync log or `null`

### `footballSync.previewSync`

- Request: none
- Response: `{ totalPlayers, mappedPlayers, unmappedPlayers, players: Array<{ id, name, currentAvg, apiId, hasMapped }> }`

### `footballSync.triggerSync`

- Request: none
- Response: `{ success: true, logId, message }`

### `footballSync.triggerStatHistoryRefresh`

- Request: none
- Response: `{ success: true, logId, message }`

### `footballSync.triggerFixtureSync`

- Request: none
- Response: `{ success: true, logId, message }`

### `admin.wipeTestData`

- Request: none
- Response: `{ wiped: true }`

## 6.16 `/admin/oracle`

Main features:

- View pending markets
- Auto-fetch stat value from API-Football
- Preview resolution
- Resolve market
- Void market
- Reopen market
- View oracle history
- Download CSV/PDF exports
- Download per-market PDF report

APIs:

### `oracle.pendingMarkets`

- Request: none
- Response: `Array<{ market, player }>`

### `statOracle.preview`

- Request: `{ marketId: number, actualStatValue: number }`
- Response: preview summary for stat-bucket settlement

### `statOracle.fetchStat`

- Request: `{ marketId: number, playerName: string, statType: string }`
- Response: `{ statValue: number, opponent: string, matchDate: string, playerNotInSquad?: boolean, ... }`

### `statOracle.resolve`

- Request: `{ marketId: number, source?: "api" | "manual", manualStatValue?: number }`
- Response: `{ success: true, outcome?, actualStatValue?, positionsSettled, totalPayout, winners, losers, voided?, refunded? }`

### `statOracle.void`

- Request: `{ marketId: number, reason: string }`
- Response: `{ success: true, positionsVoided?, totalRefunded?, ... }`

### `oracle.reopenMarket`

- Request: `{ marketId: number }`
- Response: `{ success: true, marketId, previousStatus }`

### `oracle.history`

- Request: `{ limit?: number, offset?: number }`
- Response: `Array<oracle history rows with player/market/report data>`

### Export/report routes

- `GET /api/oracle/export.csv`
- `GET /api/oracle/export.pdf`
- `GET /api/oracle/market/:marketId/report.pdf`

These require an admin cookie-backed session.

## 7. Background Jobs, Schedulers, and Scripts

## 7.1 Runtime schedulers started by the server

Defined mainly in `server/syncScheduler.ts`.

### Daily player stats sync

- Starter: `startSyncScheduler()`
- Frequency: first run at next `06:00 UTC`, then every 24 hours
- Production only: yes
- What it does:
  - Groups tracked players by team
  - Calls API-Football for each team's last finished fixture
  - Calls fixture player stats endpoint
  - Updates `players.last10Stats` and rolling averages
  - Writes sync logs
- External APIs called:
  - `GET /status`
  - `GET /fixtures?...status=FT&last=1`
  - `GET /fixtures/players?fixture=...`

### Fixture sync scheduler

- Starter: `startFixtureSyncScheduler()`
- Frequency: immediately on startup, then every 30 minutes
- What it does:
  - Fetches upcoming fixtures for all 4 supported leagues
  - Upserts fixtures
  - Auto-links fixtures to open markets
- External API called:
  - `GET /fixtures?league=...&season=...&from=...&to=...`

### Auto-lock scheduler

- Starter: `startAutoLockScheduler()`
- Frequency: immediately on startup, then every 5 minutes
- What it does:
  - Locks markets near kickoff using platform settings

### Auto-resolve scheduler

- Starter: `startAutoResolveScheduler()`
- Frequency: runtime scheduler; used to resolve finished markets automatically
- What it does:
  - Detects locked markets whose fixtures finished
  - Resolves or voids them based on stat/oracle outcomes

### Weekly library scheduler

- Starter: `startWeeklyLibraryScheduler()`
- Frequency: weekly
- What it does:
  - Weekly maintenance around the player library/import flow

## 7.2 Standalone scripts in `/scripts`

These are not automatically invoked by `package.json` scripts, but some are triggered manually or from admin flows.

### `scripts/fetch-stat-history.mjs`

- Purpose:
  - Fetch the last 10 completed fixtures per team
  - Fetch player stats per fixture
  - Build rich `last10Stats` history in DB
- Trigger:
  - Manually with `node scripts/fetch-stat-history.mjs`
  - Indirectly from `footballSync.triggerStatHistoryRefresh`
- Frequency:
  - On demand
- External APIs:
  - `GET /fixtures?...status=FT&last=10`
  - `GET /fixtures/players?fixture=...`

### `scripts/sync-fixtures.mjs`

- Purpose:
  - One-off/manual fixture fetch and market linking
- Trigger:
  - Manual
- Frequency:
  - On demand
- External APIs:
  - `GET /fixtures?league=...&season=...&from=...&to=...`

### `scripts/seed-stat-markets.mjs`

- Purpose:
  - Seed stat-based markets for all active players
- Trigger:
  - Manual
- Frequency:
  - One-time or maintenance use
- External APIs:
  - None

### `scripts/seed-new-leagues.mjs`

- Purpose:
  - Seed Premier League, Bundesliga, and Serie A players and markets
- Trigger:
  - Manual
- Frequency:
  - One-time data bootstrap
- External APIs:
  - None

### `scripts/seed-history.mjs`

- Purpose:
  - Seed `playerMatchHistory`
  - Seed `marketPriceHistory`
- Trigger:
  - Manual
- Frequency:
  - One-time/dev bootstrap
- External APIs:
  - None

## 8. External APIs and Integrations

### API-Football

Used by:

- `server/footballApi.ts`
- `server/statApi.ts`
- `server/syncScheduler.ts`
- `scripts/fetch-stat-history.mjs`
- `scripts/sync-fixtures.mjs`

Used for:

- API quota/status
- Upcoming fixtures
- Last finished fixtures
- Fixture player stats
- Oracle stat fetches
- League squad imports

### Privy

Used by:

- `@privy-io/react-auth` on frontend
- `@privy-io/server-auth` on backend

Used for:

- User authentication
- Token verification
- User identity bootstrap

### MySQL / Drizzle ORM

Used for:

- All application persistence
- Players
- Markets
- Positions
- Orders/order book
- Oracle reports
- Price alerts
- Audit logs
- Sync logs
- Fixtures
- Platform settings

### Forge/internal proxy integrations

Referenced by:

- storage helpers
- map helpers
- image generation
- voice transcription/data API helpers

These integrations are code-referenced but not central to the main trading flow documented above, and the required env vars are not present in the current `.env`.

## 9. Notes / Caveats

- The UI uses logical tRPC procedures, but all of them are transported over `/api/trpc`.
- Several response shapes are DB-backed and include extra timestamps/ids not listed here; the shapes above focus on the fields the UI actually uses.
- The current `.env` contains secrets. This document intentionally describes variable purpose only and should not duplicate actual secret values.
- `client/src/const.ts` still references OAuth portal env vars, but the active auth flow is Privy-driven.
