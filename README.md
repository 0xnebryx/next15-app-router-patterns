# next15-app-router-patterns

> Patterns for Next.js 15 App Router specifically for Web3 dashboards — server components, streaming, real-time data, error boundaries.

The App Router is great but Web3 use cases hit edges the docs don't cover well. These are patterns from shipping production crypto dashboards.

---

## Server vs client component boundaries

The rule:
- **Server component** by default. Anything that fetches public data or static content.
- **Client component** only when you need browser APIs, hooks, event handlers, or wallet/signing context.

For a Web3 dashboard:

```
app/
├── layout.tsx                 [server] — shell, fonts, providers
├── page.tsx                   [server] — landing
├── dashboard/
│   ├── page.tsx               [server] — fetches public data, renders layout
│   ├── balance-panel.tsx      [client] — wallet-connected, signs transactions
│   ├── market-prices.tsx      [server] — fetches from RPC, no interactivity
│   └── trade-form.tsx         [client] — form, validation, signing
└── providers.tsx              [client] — wraps WalletProvider, QueryClient, etc.
```

The mistake people make: `'use client'` at the top of `layout.tsx` because they want a global theme provider. This makes the entire app client-rendered. Instead, isolate the providers in a `<Providers>` client component and mount it inside the server layout.

```tsx
// app/layout.tsx (server)
import { Providers } from './providers';
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html><body><Providers>{children}</Providers></body></html>
  );
}

// app/providers.tsx
'use client';
import { WalletAdapterProvider } from '...';
export function Providers({ children }) {
  return <WalletAdapterProvider>{children}</WalletAdapterProvider>;
}
```

## Server-side RPC reads (the cache friend)

For read-only on-chain state (token metadata, mint info, public balances), call from a server component:

```tsx
// app/token/[mint]/page.tsx
import { Connection } from '@solana/web3.js';

export default async function TokenPage({ params }: { params: { mint: string } }) {
  const conn = new Connection(process.env.RPC_URL!);
  const info = await conn.getParsedAccountInfo(new PublicKey(params.mint));
  return <TokenView info={info} />;
}
```

This RPC call runs at the edge / on your server. Next.js automatically dedupes and caches it per request. With `unstable_cache`, you also get cross-request caching:

```tsx
import { unstable_cache } from 'next/cache';
const getMint = unstable_cache(
  async (mint: string) => connection.getParsedAccountInfo(new PublicKey(mint)),
  ['mint'],
  { revalidate: 60, tags: [`mint-${mint}`] }
);
```

## Streaming UI with `<Suspense>`

For dashboards with heavy data, stream sections as they're ready:

```tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <>
      <Header />                                  {/* renders instantly */}
      <Suspense fallback={<BalanceSkeleton />}>
        <BalancePanel />                          {/* loads async, streams in */}
      </Suspense>
      <Suspense fallback={<HistorySkeleton />}>
        <TransactionHistory />                    {/* parallel, streams in */}
      </Suspense>
    </>
  );
}
```

Each `<Suspense>` boundary renders its fallback first, then swaps in the real content when ready. Parallel boundaries fetch in parallel.

## Client-side real-time data

For data that updates frequently (prices, balances), don't poll from server components. Use a client component with TanStack Query:

```tsx
'use client';
import { useQuery } from '@tanstack/react-query';

export function PriceTicker({ mint }: { mint: string }) {
  const { data } = useQuery({
    queryKey: ['price', mint],
    queryFn: () => fetch(`/api/price/${mint}`).then(r => r.json()),
    refetchInterval: 5000,
    staleTime: 4000,
  });
  return <span>{data?.price ?? '...'}</span>;
}
```

Set `refetchInterval > staleTime` to avoid double-fetching.

## Server actions for signed mutations

For "transaction needs signing" mutations, do the off-chain pre-flight on the server, return the unsigned TX to the client, client signs and submits:

```tsx
// app/actions/build-swap.ts
'use server';
export async function buildSwap(input: SwapInput) {
  // Validate input server-side
  if (!input.amount || input.amount < 0) throw new Error('Invalid amount');
  // Quote via Jupiter
  const quote = await jupiterQuote(input);
  // Build but don't sign
  const tx = await jupiterBuildTx(quote);
  return { tx: tx.serialize().toString('base64'), quote };
}

// On client:
const { tx } = await buildSwap(input);
const deserialized = VersionedTransaction.deserialize(Buffer.from(tx, 'base64'));
const signed = await wallet.signTransaction(deserialized);
const sig = await connection.sendRawTransaction(signed.serialize());
```

Benefits: input validation happens server-side (can't be bypassed), quoting/building uses server-side env (RPC URL, Jupiter config), client only does what only it can — sign.

## Error boundaries that matter

Next.js auto-renders `error.tsx` for caught errors in a segment. For Web3, surface useful info:

```tsx
// app/dashboard/error.tsx
'use client';
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  const userFacing = humanizeWeb3Error(error);
  return (
    <div className="p-6">
      <h2>{userFacing.title}</h2>
      <p>{userFacing.message}</p>
      {userFacing.canRetry && <button onClick={reset}>Try again</button>}
    </div>
  );
}

function humanizeWeb3Error(e: Error) {
  if (e.message.includes('BlockhashNotFound')) return { title: 'Network is busy', message: 'Please try again in a moment.', canRetry: true };
  if (e.message.includes('User rejected')) return { title: 'Cancelled', message: 'You cancelled the transaction.', canRetry: false };
  if (e.message.includes('insufficient lamports')) return { title: 'Not enough SOL', message: 'You need more SOL for transaction fees.', canRetry: false };
  return { title: 'Something went wrong', message: e.message, canRetry: true };
}
```

## Footguns

- **Don't fetch from RPC in `layout.tsx`.** It runs on every navigation. Move to `page.tsx` or specific segments.
- **Don't pass server-fetched data through context.** Server components can't supply context to client components directly. Pass as props through a client boundary.
- **`'use client'` is sticky downward.** A client component's children are client unless explicitly marked otherwise.
- **Route segments cache by default.** `revalidate` controls how long. For real-time financial data set `revalidate: 0` or use client fetching.
- **Server actions can't return non-serializable values.** Bigint and `Connection` instances can't cross the wire — convert to strings/JSON.

## License

MIT
