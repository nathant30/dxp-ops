# Xpress DXP — Claude Code Instructions

## Data Contract (read first)

Before modifying **any** of the following, read `XPRESS_DATA_CONTRACT.md`:

- `supabase/functions/databricks-proxy/index.ts`
- Any `DB_*` variable assignment in `index.html`
- Any `StatusId` grouping or `VehicleTypeId` scope in SQL

Update `XPRESS_DATA_CONTRACT.md` **in the same commit** when you:

- Add or rename a field in any edge function query
- Add a new query `type`
- Change a `DB_*` mapping in `index.html`
- Change any `StatusId` grouping or `VehicleTypeId` scope
- Change `PERF_CONFIG` keys or their defaults

## Hard rules — never do without explicit instruction

1. **Never remove a Required field** — frontend silently shows ₱0 or "No data"
2. **Never set `verify_jwt: true`** — frontend sends only `apikey`, not JWT; causes 401 on all data
3. **Never rewrite the edge function from scratch** — extend it; rewriting drops required fields
4. **Never change StatusId groupings** without updating the contract
5. **Never remove a VehicleTypeId** from `services_overview` — all 5 must always be IN (1,2,3,10,16)

## Edge function deploy command

```bash
cd /tmp/dxp-edge
SUPABASE_ACCESS_TOKEN=sbp_76ba408056a12de1ca12253cc2d84b1c8ae9fefe \
  npx supabase functions deploy databricks-proxy \
  --project-ref nycqnxaxcpybjpvslwoc \
  --use-api \
  --no-verify-jwt
```

## Repo and tech

- **GitHub Pages** static site — single `index.html`, all JS/CSS inline
- **Supabase project:** `nycqnxaxcpybjpvslwoc`
- **Edge function:** `databricks-proxy` — the only backend; routes all Databricks SQL
- **Frontend:** vanilla JS, no bundler; `DB_*` globals populated at refresh time
- All timestamps in Databricks are UTC — always use `CONVERT_TIMEZONE('UTC','Asia/Manila',col)`

## Syntax validation before every commit

```bash
node -e "
const fs = require('fs'), vm = require('vm');
const html = fs.readFileSync('index.html','utf8');
const m = html.match(/<script>([\s\S]*?)<\/script>\s*<\/body>/);
new vm.Script(m[1]);
console.log('✓ OK');
"
```

Run this before `git commit` on any `index.html` change.
