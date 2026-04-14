# Task Plan: Update .env Default and README CDN References

## Analysis

### .env (2 lines)
- Line 1: `NEXT_PUBLIC_CDN_URL=https://pub-0a21161ea37a4400b21faa6831855135.r2.dev` -- change to `https://cdn.tetoz.com.br`
- Line 2: `NEXT_PUBLIC_COMPANY_ID=e068a043-7384-48bd-bcb3-a99a4cf73328` -- NO CHANGE

### README.md (3 locations)
1. Line 44 area: Env vars table example column shows `https://pub-xxx.r2.dev` -- change to `https://cdn.tetoz.com.br`
2. Lines 501-504 area: Vercel deployment `vercel env add` examples use `https://pub-xxx.r2.dev` -- change to `https://cdn.tetoz.com.br`
3. Line 36 area: `.env.local` example uses `https://sua-cdn.example.com` -- keep as-is (it's a placeholder showing override mechanism, not an r2.dev reference)

## Implementation

### Step 1: Update .env
- Replace the NEXT_PUBLIC_CDN_URL value from the r2.dev URL to cdn.tetoz.com.br

### Step 2: Update README.md
- Replace `https://pub-xxx.r2.dev` in the env vars table example
- Replace `https://pub-xxx.r2.dev` in the Vercel deployment examples (2 occurrences for production and preview)
- Keep `.env.local` example as-is (already uses generic placeholder)

## Files Modified
- `.env` (1 line changed)
- `README.md` (3 occurrences changed)
