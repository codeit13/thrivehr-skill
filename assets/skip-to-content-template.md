# Skip to Main Content Template

First focusable control on every page. Required for WCAG 2.4.1 Bypass Blocks.

Copy to `src/components/layout/skip-to-content.tsx` and render at the top of `AppShell` (before header).

```tsx
import { cn } from '@/lib/utils';

export function SkipToContent({ targetId = 'main-content' }: { targetId?: string }) {
  return (
    <a
      href={`#${targetId}`}
      className={cn(
        'skip-link',
        'sr-only focus:not-sr-only focus:absolute focus:left-4 focus:top-4 focus:z-[100]',
        'rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground',
        'focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2',
      )}
    >
      Skip to main content
    </a>
  );
}
```

### App shell usage

```tsx
import { SkipToContent } from '@/components/layout/skip-to-content';

export function AppShell({ children }: { children: React.ReactNode }) {
  return (
    <>
      <SkipToContent />
      <header role="banner" className="…">…</header>
      <div className="flex">
        <nav aria-label="Primary navigation" className="…">…</nav>
        <main id="main-content" tabIndex={-1} className="flex-1 outline-none focus-visible:ring-2 focus-visible:ring-ring">
          {children}
        </main>
      </div>
    </>
  );
}
```

### Route change focus (optional)

```tsx
import { useEffect, useRef } from 'react';
import { useLocation } from 'react-router-dom';

export function MainContent({ children }: { children: React.ReactNode }) {
  const mainRef = useRef<HTMLElement>(null);
  const { pathname } = useLocation();

  useEffect(() => {
    mainRef.current?.focus();
  }, [pathname]);

  return (
    <main id="main-content" ref={mainRef} tabIndex={-1}>
      {children}
    </main>
  );
}
```

Pair with unique `document.title` per route.
