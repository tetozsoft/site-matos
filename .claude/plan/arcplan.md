# arcplan

## meta
- task: Set up cdn.tetoz.com.br custom domain for R2 bucket, replacing pub-XXX.r2.dev URLs to route CDN traffic through Cloudflare edge network for reliable Brazilian ISP delivery
- repos: backend, site-template-base, portal
- execution-order: infrastructure (Cloudflare R2 custom domain) -> backend -> site-template-base -> portal -> existing deployments migration

## cross-repo

### contracts
- name: R2_PUBLIC_URL env var
  type: env-var
  producer: infrastructure (Cloudflare R2 custom domain setup)
  consumer: backend
  shape: R2_PUBLIC_URL=https://cdn.tetoz.com.br

- name: NEXT_PUBLIC_CDN_URL env var
  type: env-var
  producer: backend (WebsiteLauncherConfig passes to CF Pages at project creation)
  consumer: site-template-base (deployed company websites)
  shape: NEXT_PUBLIC_CDN_URL=https://cdn.tetoz.com.br

- name: Photo/avatar URLs in API responses
  type: api-response-field
  producer: backend
  consumer: portal, app
  shape: Full URLs like https://cdn.tetoz.com.br/{company_id}/media/imoveis/{property_id}/foto-1.webp (new uploads) or https://pub-XXX.r2.dev/... (existing DB rows)

### dependencies
- Cloudflare R2 custom domain must be active before changing backend R2_PUBLIC_URL
  reason: new uploads would reference a domain that does not resolve yet
- Backend env var change must happen before site-template-base CF Pages env var update
  reason: backend passes R2_PUBLIC_URL as NEXT_PUBLIC_CDN_URL when creating new CF Pages projects

### risks
- risk: extract_key_from_url fails on old URLs after R2_PUBLIC_URL change
  impact: Avatar deletion, photo deletion, watermark re-application from existing photos all fail silently (return None)
  mitigation: Add R2_LEGACY_PUBLIC_URL env var support to R2Client; extract_key_from_url tries new prefix first, falls back to legacy

- risk: Existing deployed company websites still have old NEXT_PUBLIC_CDN_URL
  impact: They continue working (old .r2.dev URL still resolves) but do not benefit from CDN edge routing
  mitigation: Update env vars on all existing CF Pages projects via Cloudflare API; trigger redeploy

- risk: DNS propagation delay for cdn.tetoz.com.br
  impact: Brief window where new uploads reference cdn.tetoz.com.br but some DNS resolvers have not propagated
  mitigation: Cloudflare proxy (orange cloud) handles this quickly; TTL is typically seconds for proxied records

## repo:backend

### scope
Add legacy URL support to R2Client so extract_key_from_url handles both old (pub-XXX.r2.dev) and new (cdn.tetoz.com.br) URL prefixes. Update R2Config to load an optional R2_LEGACY_PUBLIC_URL env var. Update .env.example documentation. The R2_PUBLIC_URL env var value itself is changed in the .env file (not in code).

### affected-modules
- module: crates/tetoz_core/src/storage/config.rs
  action: modify
  reason: Add legacy_public_url: Option<String> field to R2Config, loaded from R2_LEGACY_PUBLIC_URL env var

- module: crates/tetoz_core/src/storage/r2.rs
  action: modify
  reason: Add legacy_public_url: Option<String> field to R2Client; update extract_key_from_url to try current prefix first, then legacy prefix; update new() constructor

- module: .env.example
  action: modify
  reason: Add R2_LEGACY_PUBLIC_URL documentation; update R2_PUBLIC_URL example value to https://cdn.tetoz.com.br

### requirements
1. R2Config must load optional R2_LEGACY_PUBLIC_URL env var and pass it to R2Client
2. R2Client::extract_key_from_url must try stripping {public_url}/ first, then {legacy_public_url}/ if present, returning the first match
3. No database migration needed -- old URLs continue to work via R2 default .r2.dev endpoint
4. After deployment, change R2_PUBLIC_URL in backend .env to https://cdn.tetoz.com.br and set R2_LEGACY_PUBLIC_URL=https://pub-0a21161ea37a4400b21faa6831855135.r2.dev

### constraints
- Follow existing module patterns in storage/ -- no new files, modify existing config.rs and r2.rs
- R2_LEGACY_PUBLIC_URL is optional -- system works without it (backward compatible)
- NEVER run Rust locally -- test via podman-compose up --build

### interfaces
- exposes: No new API endpoints. Internal change to R2Client methods only.
- env-vars: New optional R2_LEGACY_PUBLIC_URL; existing R2_PUBLIC_URL value changes from https://pub-XXX.r2.dev to https://cdn.tetoz.com.br

### reference-patterns
- R2Config::load() at crates/tetoz_core/src/storage/config.rs:27-55 -- follow same env::var().ok() pattern for new field
- R2Client::new() at crates/tetoz_core/src/storage/r2.rs:17-41 -- add field to constructor
- R2Client::extract_key_from_url() at crates/tetoz_core/src/storage/r2.rs:237-239 -- modify to try both prefixes

## repo:site-template-base

### scope
Update next.config.ts to add cdn.tetoz.com.br to image remotePatterns. Update .env default value. Update README documentation to reference new CDN URL pattern.

### affected-modules
- module: next.config.ts
  action: modify
  reason: Add cdn.tetoz.com.br hostname to images.remotePatterns; keep old r2.dev patterns for backward compatibility

- module: .env
  action: modify
  reason: Change NEXT_PUBLIC_CDN_URL default to https://cdn.tetoz.com.br

- module: README.md
  action: modify
  reason: Update CDN URL references from pub-xxx.r2.dev to cdn.tetoz.com.br

### requirements
1. next.config.ts must include BOTH cdn.tetoz.com.br AND existing **.r2.dev patterns (backward compat for transition period)
2. .env must show the new default CDN URL
3. README documentation must reference new CDN URL in all examples

### constraints
- This repo uses output: export (static site) -- remotePatterns are advisory since images.unoptimized: true
- Do NOT modify any .tsx files, lib/cdn.ts, or hooks/use-cdn.ts -- they already read NEXT_PUBLIC_CDN_URL from env and require no code changes

### interfaces
- consumes: NEXT_PUBLIC_CDN_URL env var (set per-deployment via CF Pages env vars)

### reference-patterns
- src/lib/cdn.ts:4 -- reads process.env.NEXT_PUBLIC_CDN_URL (no change needed)
- src/hooks/use-cdn.ts -- React Query hooks over cdn.ts (no change needed)

## repo:portal

### scope
Update next.config.ts to add cdn.tetoz.com.br to image remotePatterns alongside existing r2.dev entries.

### affected-modules
- module: next.config.ts
  action: modify
  reason: Add cdn.tetoz.com.br hostname to images.remotePatterns for future next/image compatibility with new CDN domain

### requirements
1. Add { protocol: https, hostname: cdn.tetoz.com.br, pathname: /** } to images.remotePatterns
2. Keep existing pub-XXX.r2.dev and *.r2.dev patterns for backward compatibility (existing URLs in DB)

### constraints
- Portal currently uses raw img tags for R2 images, not next/image, so this is a preventive change
- Follow existing pattern in the remotePatterns array

### interfaces
- consumes: Full URLs from backend API responses (no env var for CDN URL in portal)

### reference-patterns
- portal/next.config.ts:36-52 -- existing remotePatterns array structure
