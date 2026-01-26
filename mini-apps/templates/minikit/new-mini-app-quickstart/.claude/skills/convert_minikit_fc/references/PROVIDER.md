# Provider Migration

**You must CREATE these provider components — they are NOT exported from `@farcaster/miniapp-sdk`.**

> **CRITICAL**: After creating the provider files, you MUST update `layout.tsx` to use them. The provider only works when it wraps the entire app. Creating the files without updating the layout does nothing.

## Contents

- [Complete Provider Transformation](#complete-provider-transformation)
- [Step 1: Create MiniAppProvider](#step-1-create-miniappprovider)
- [Step 2: Create Wagmi Provider](#step-2-create-wagmi-provider)
- [Step 3: Combine Providers](#step-3-combine-providers)
- [Step 4: Use in Layout](#step-4-use-in-layout)
- [Using the Context](#using-the-context)
- [Remove Old Providers](#remove-old-providers)

---

## Complete Provider Transformation

Remove `OnchainKitProvider` and `MiniKitProvider` entirely. Replace with custom MiniAppProvider and WagmiProvider components that you create.

**BEFORE (MiniKit with OnchainKit):**
```typescript
import { OnchainKitProvider } from '@coinbase/onchainkit';

export function RootProvider({ children }) {
  return (
    <OnchainKitProvider apiKey={...} chain={base}>
      {children}
    </OnchainKitProvider>
  );
}
```

**AFTER (Farcaster SDK):**
```typescript
import { MiniAppProvider } from '@/components/providers/MiniAppProvider';
import WagmiProvider from '@/components/providers/WagmiProvider';

export function RootProvider({ children }) {
  return (
    <MiniAppProvider>
      <WagmiProvider>
        {children}
      </WagmiProvider>
    </MiniAppProvider>
  );
}
```

**OnchainKitProvider is completely removed. Do NOT wrap it or keep it.**

---

## Step 1: Create MiniAppProvider

`src/components/providers/MiniAppProvider.tsx`:

```typescript
'use client'

import { createContext, useContext, useEffect, useState, type ReactNode } from 'react';
import sdk from '@farcaster/miniapp-sdk';

interface MiniAppContextValue {
  context: Awaited<typeof sdk.context> | null;
  isReady: boolean;
}

export const MiniAppContext = createContext<MiniAppContextValue | null>(null);

export function useMiniApp() {
  const context = useContext(MiniAppContext);
  if (!context) {
    throw new Error('useMiniApp must be used within MiniAppProvider');
  }
  return context;
}

export function MiniAppProvider({ children }: { children: ReactNode }) {
  const [context, setContext] = useState<Awaited<typeof sdk.context> | null>(null);
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    const init = async () => {
      const isInApp = await sdk.isInMiniApp();
      if (isInApp) {
        const ctx = await sdk.context;
        setContext(ctx);
        await sdk.actions.ready();
        setIsReady(true);
      }
    };
    init();
  }, []);

  return (
    <MiniAppContext.Provider value={{ context, isReady }}>
      {children}
    </MiniAppContext.Provider>
  );
}
```

> **Note**: Use `Awaited<typeof sdk.context>` to get the context type. The SDK uses a default export.

---

## Step 2: Create Wagmi Provider

`src/components/providers/WagmiProvider.tsx`:

```typescript
'use client'

import { createConfig, http, WagmiProvider as WagmiBase } from 'wagmi';
import { base } from 'wagmi/chains';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { farcasterMiniApp } from '@farcaster/miniapp-wagmi-connector';
import { ReactNode, useState } from 'react';

const config = createConfig({
  chains: [base],
  transports: { [base.id]: http() },
  connectors: [farcasterMiniApp()],
});

export default function WagmiProvider({ children }: { children: ReactNode }) {
  const [queryClient] = useState(() => new QueryClient());

  return (
    <WagmiBase config={config}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </WagmiBase>
  );
}
```

---

## Step 3: Combine Providers

`src/app/providers.tsx`:

```typescript
'use client'

import { ReactNode } from 'react';
import { MiniAppProvider } from '@/components/providers/MiniAppProvider';
import WagmiProvider from '@/components/providers/WagmiProvider';

export function Providers({ children }: { children: ReactNode }) {
  return (
    <MiniAppProvider>
      <WagmiProvider>
        {children}
      </WagmiProvider>
    </MiniAppProvider>
  );
}
```

---

## Step 4: Wrap Layout (REQUIRED)

> **This step is REQUIRED. Without it, the providers do nothing.**

**The provider MUST wrap the entire app in the root layout.**

`src/app/layout.tsx`:

```typescript
import { Providers } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

> **Important**: All pages and components must be descendants of `<Providers>` to access the mini app context.

---

## Using the Context

```typescript
import { useMiniApp } from '@/components/providers/MiniAppProvider';

function MyComponent() {
  const { context, isReady } = useMiniApp();

  if (!isReady) return <div>Loading...</div>;
  if (!context) return <div>Open in Farcaster</div>;

  return <div>Welcome {context.user?.displayName}</div>;
}
```

---

## Remove Old Providers

```typescript
// Delete all OnchainKit imports
import { OnchainKitProvider } from '@coinbase/onchainkit';
import { MiniKitProvider } from '@coinbase/onchainkit/minikit';
import '@coinbase/onchainkit/styles.css';

// Delete from .env
NEXT_PUBLIC_ONCHAINKIT_API_KEY=xxx
```

**Important**: Remove the entire `OnchainKitProvider` wrapper, not just `MiniKitProvider`. The Farcaster SDK does not use OnchainKit.
