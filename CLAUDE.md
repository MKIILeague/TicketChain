# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

TicketWave is a blockchain-powered NFT ticketing platform. The repo contains three independent codebases that each have their own dependencies and tooling:

- **Root** — the Next.js 14 web app (Pages Router, JavaScript, no TypeScript).
- **`smartcontract/`** — the `ConcertTicketsNFT` Solidity contract (ERC721) + Hardhat.
- **`graph/ticket/`** — The Graph subgraph that indexes contract events.

## Commands

Web app (run from repo root):

```bash
npm install
npm run dev      # dev server at http://localhost:3000
npm run build    # production build
npm run start    # serve production build
npm run lint     # next lint (eslint-config-next)
```

Subgraph (run from `graph/ticket/`):

```bash
npm run codegen  # regenerate types from schema.graphql + ABI
npm run build
npm run deploy   # deploy to The Graph Studio (subgraph name: ticket)
npm run test     # matchstick-as tests; run a focused file: graph test <name>
```

Smart contract (run from `smartcontract/`): uses Hardhat. Note the README's `npx hardhat run scripts/deploy.js --network optimism` does not match `hardhat.config.js`, which only configures zkSync networks — there is no `optimism` network or `scripts/` dir checked in. Treat contract deployment as out-of-band.

## Architecture

The defining characteristic of this app: **concert/event metadata is NOT on-chain.** Understand this split before touching anything ticketing-related.

- **`data/data.json`** is the source of truth for all event display data (title, images, date, venue, price, description). The smart contract only stores `concertId → { totalCapacity, ticketsSold }` and `tokenId → { concertId, hasEntered, purchaseDate }`. The two are joined by `concertId`/`id` in the UI.
- **`data/students.json`** is an off-chain allowlist used to gate certain purchases/certificate minting by email.
- A "certificate" is just a concert whose `title` contains the word "certificate" (checked via `title.toLowerCase().includes("certificate")`). There is no separate type — the same NFT mint flow is reused.

### Data flow

1. **Writes** (purchase, scan-for-entry) go through `thirdweb` from the browser to the contract. See `utils/client.js` for the live `contract` instance — chain `11155420` (**Optimism Sepolia**, despite the README mentioning Optimism mainnet) at `0x5039e2bF006967F8049933c7DF6c7Ca0b49AeBeB`. Contract calls use thirdweb's string method signatures (e.g. `prepareContractCall({ contract, method: "function purchaseTickets(...)" })`), not the ABI.
2. **Per-event reads** (e.g. remaining tickets) use `useReadContract` directly against the contract.
3. **User's owned tickets** are NOT read from the contract. `pages/profile.js` queries The Graph subgraph (`https://api.studio.thegraph.com/query/90400/ticket/version/latest`) for `ticketMinteds` by buyer address, then enriches each result with metadata from `data.json`.

### Frontend conventions

- Pages Router under `pages/`. `pages/_app.js` wraps everything in `QueryProvider` (React Query) → `ThirdwebProvider` → `NextUIProvider`, and renders `Navbar`/`Footer` on every page **except** `/landingpage`.
- Event detail pages are dynamic: `pages/[slug].js`. Slugs are `<title-kebab>-<id>` and the numeric `id` is recovered with `slug.split("-").pop()`. Slug generation lives in `pages/events.js` (`getEventSlug`).
- Import alias `@/*` maps to the repo root (`jsconfig.json`).
- UI mixes three libraries: **NextUI** (primary), **MUI**, and **shadcn/ui** (config in `components.json`, style "new-york", components in `components/ui/`, util `lib/utils.js` `cn()`). Match whatever the surrounding file already uses.
- Wallet/account: `useActiveAccount()` from `thirdweb/react`; `account.address` is the connected wallet. Guard rendering on `!address` for wallet-gated pages (see `pages/verify.js`).
- PWA via `next-pwa` (`next.config.mjs`) — disabled in development, writes service worker to `public/`.

### Secrets

All runtime config is `NEXT_PUBLIC_*` env vars (Firebase config in `utils/firebase.js`, thirdweb client id in `utils/client.js`). There is no `.env` checked in; these must be set in the environment / Vercel.

## Gotchas

- **`utils/constants.js` is stale/unused** — it exports a different `contractAddress` (`0xe2a3d5...`) and a partial ABI. The live contract is in `utils/client.js`. Don't wire new code to `constants.js`.
- **`other/` holds dead variants** (`[id].js.manyticket`, `[id].js.one section`) of the detail page — not part of the build. Ignore unless explicitly asked.
- After changing the contract's events or `schema.graphql`, the subgraph must be regenerated (`npm run codegen`) and redeployed, and the query URL/version in `pages/profile.js` updated, or the profile page silently shows no tickets.
- The Graph subgraph network is `optimism-sepolia` and must stay in sync with the contract address/chain in `utils/client.js`.
