# Vested Web3 Copytrading Platform - Zero-Error Deployment Guide

## CRITICAL: How Environment Variables Work in Vercel

**VERCEL DOES NOT USE vercel.json FOR ENV VARIABLES**

Environment variables are set in the **Vercel Dashboard UI**, NOT in configuration files. The `vercel.json` file does NOT contain an `env` property.

Your `vercel.json` is CORRECT - it has NO env property. All variables are set in Vercel Dashboard Settings.

---

## Pre-Deployment Checklist

### 1. Get Your Supabase Credentials

From your Supabase project, gather these THREE values:

| Variable | Where to Find | What It Looks Like |
|----------|---------------|-------------------|
| `VITE_SUPABASE_URL` | Supabase Dashboard → Project Settings → API → "Project URL" | `https://xyz.supabase.co` |
| `VITE_SUPABASE_ANON_KEY` | Supabase Dashboard → Project Settings → API → "anon public" | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...` (long string) |
| `VITE_ADMIN_EMAIL` | You decide | `admin@yourcompany.com` |

**NEVER expose these in code or git - they go ONLY in Vercel Dashboard**

### 2. Supabase Database Setup
Ensure the following tables exist in your Supabase project:

**Required Tables**:
- `users` - User accounts with `email`, `role` (admin/user), `full_name`
- `user_balance` - User balances with `user_email`, `balance_usd`, `total_invested`, `total_profit_loss`
- `portfolio` - User holdings with `user_email`, `crypto_symbol`, `amount`, `avg_buy_price`
- `cryptocurrencies` - Available cryptos with `symbol`, `price`, `change_24h`, `is_active`
- `copy_traders` - Approved traders with `trader_name`, `is_approved`, `min_allocation`, `profit_split_pct`
- `copy_trade` - Copy trading relationships with `user_email`, `trader_id`, `allocation`, `is_active`
- `transactions` - Transaction history with `user_email`, `type`, `amount`, `status`, `created_date`

Run the schema setup from `src/schema.sql` in your Supabase SQL Editor to initialize all tables.

---

## DEPLOYMENT STEPS

### Step 1: Push Code to GitHub

```bash
git add .
git commit -m "fix: production deployment ready"
git push origin main
```

### Step 2: Import Project to Vercel

1. Go to https://vercel.com/new
2. Click "Import Git Repository"
3. Authenticate with GitHub
4. Select your repository
5. Click "Import Project"

### Step 3: ADD ENVIRONMENT VARIABLES (MOST CRITICAL)

**This is where environment variables go - NOT in vercel.json**

1. After importing, you'll see "Configure Project" page
2. Scroll to "Environment Variables" section
3. Add these 3 variables:

**Variable 1:**
```
Name: VITE_SUPABASE_URL
Value: https://your-project.supabase.co
```

**Variable 2:**
```
Name: VITE_SUPABASE_ANON_KEY
Value: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9... (paste your actual key)
```

**Variable 3:**
```
Name: VITE_ADMIN_EMAIL
Value: admin@yourcompany.com
```

4. For each variable, select checkboxes:
   - ☑ Production
   - ☑ Preview
   - ☑ Development

5. Click "Add" for each variable
6. Verify all 3 appear in the list

### Step 4: Deploy

1. Click "Deploy" button (bottom right)
2. Wait 2-3 minutes for build
3. See green checkmark = SUCCESS

### Step 5: Verify Deployment

After successful deployment:

1. Click your project domain to open the app
2. Test **Login** - verify Supabase auth works
3. Test **Dashboard** - verify data loads from Supabase
4. Test **Admin Panel** (if superadmin)
5. Check **Browser Console** (F12) for any errors

**Expected Result**: Zero console errors, app fully functional

---

## TROUBLESHOOTING

### ERROR: "Invalid request: `env` should be object" or "buildGroups not allowed"
**Root Cause**: Incorrect vercel.json syntax
**FIXED**: vercel.json now has NO `env` property (it's correct)
**Action**: Verify vercel.json doesn't have `env` or `buildGroups` properties - it doesn't

### ERROR: "VITE_SUPABASE_URL is undefined" or "Cannot read properties"
**Root Cause**: Environment variables not set in Vercel Dashboard
**Solution**:
1. Go to Vercel Dashboard > Your Project > Settings
2. Click "Environment Variables" (left sidebar)
3. Add all 3 variables again:
   - `VITE_SUPABASE_URL` = your URL
   - `VITE_SUPABASE_ANON_KEY` = your key
   - `VITE_ADMIN_EMAIL` = admin email
4. Make sure all 3 environments are checked (Production, Preview, Development)
5. **Redeploy** from Deployments tab

### ERROR: "Module not found @/api/base44Client"
**Root Cause**: Old base44 imports still in code
**FIXED**: All files already updated to use Supabase
**Verify**: Run `grep -r "base44" src/` - should return nothing
**If not fixed**: 
1. Check src/pages/*.jsx for any `import ... base44`
2. Replace with `import { supabase } from "@/api/supabaseClient"`
3. Rebuild locally with `pnpm build`
4. Push to GitHub - Vercel will auto-redeploy

### ERROR: "App loads but no data displays"
**Root Cause**: Supabase connection issue or missing data
**Solution**:
1. Check console (F12) for error messages
2. Verify Supabase project is active
3. Verify environment variables are exactly correct (no extra spaces)
4. Check Supabase database has tables:
   - users
   - user_balance
   - portfolio
   - transaction
   - cryptocurrencies
5. Add sample data if tables are empty
6. Clear browser cache and reload

### ERROR: "Cannot GET /" after deployment
**Root Cause**: SPA routing not configured
**FIXED**: vercel.json has correct rewrites configuration
**Solution**: Clear Vercel cache and redeploy

## Security Best Practices

### Supabase Configuration

1. **Row Level Security (RLS)**:
   - Enable RLS on all tables in Supabase:
   ```sql
   ALTER TABLE users ENABLE ROW LEVEL SECURITY;
   ALTER TABLE portfolio ENABLE ROW LEVEL SECURITY;
   ALTER TABLE copy_trade ENABLE ROW LEVEL SECURITY;
   ```

2. **RLS Policies**:
   ```sql
   -- Users can only see their own data
   CREATE POLICY "Users see own records" ON portfolio
     FOR SELECT USING (auth.uid() = user_id);
   ```

3. **API Keys**:
   - Never commit `VITE_SUPABASE_ANON_KEY` to git
   - Use Vercel Environment Variables for secrets
   - Rotate keys periodically in Supabase

### Application Security

1. **Authentication**:
   - All user data queries filter by authenticated user's email
   - Admin routes check for `role = 'admin'` in database

2. **Data Validation**:
   - All numeric inputs validated before database operations
   - SQL injection prevented via Supabase parameterized queries

3. **CORS & HTTPS**:
   - Vercel automatically uses HTTPS
   - Supabase CORS configured for your domain

## Performance Optimization

### Caching Strategy

The application uses Recharts for charts and React Query for data fetching:
- Asset caching: 31536000 seconds (1 year) for hashed assets
- HTML caching: 0 seconds (must-revalidate) for `index.html`
- API responses: Cached in-memory by React Query

### Monitoring

1. **Vercel Analytics**:
   - Navigate to Analytics tab to monitor performance
   - Check Core Web Vitals monthly

2. **Supabase Monitoring**:
   - Check Database section for query performance
   - Monitor Storage and Bandwidth usage

3. **Error Tracking**:
   - Monitor browser console for errors
   - Check Vercel Function logs for any API issues

## Rollback Procedure

If deployment causes issues:

1. **Immediate Rollback** (Vercel Dashboard):
   - Go to Deployments
   - Find previous successful deployment
   - Click three dots > Promote to Production

2. **Code Rollback** (Git):
   ```bash
   git revert <commit-hash>
   git push origin main
   # Vercel will auto-redeploy
   ```

## Post-Deployment Tasks

### Day 1: Verification
- [ ] Application loads without errors
- [ ] Login/authentication works
- [ ] Dashboard displays user balance correctly
- [ ] Portfolio page fetches holdings from Supabase
- [ ] Copy trading traders list appears
- [ ] Admin dashboard accessible to admin users
- [ ] Mobile navigation works on small screens

### Week 1: Monitoring
- [ ] Monitor error logs in browser console
- [ ] Check Vercel deployment logs for any warnings
- [ ] Verify Supabase database queries complete within SLA
- [ ] Test transaction processing (deposits/withdrawals)
- [ ] Monitor user reports of issues

### Ongoing: Maintenance
- [ ] Review analytics weekly
- [ ] Update dependencies monthly (`pnpm outdated`)
- [ ] Monitor Supabase storage/bandwidth usage
- [ ] Backup database monthly
- [ ] Review security logs in Supabase

## Contact & Support

For deployment issues:
1. Check build logs in Vercel Dashboard
2. Review error messages in browser console (F12)
3. Verify environment variables in Vercel Settings
4. Check Supabase status page
5. Contact Vercel support if infrastructure issue

## Technology Stack Reference

| Component | Technology | Version |
|-----------|-----------|---------|
| Frontend | React | 19.2+ |
| Bundler | Vite | 6.4.1+ |
| Styling | Tailwind CSS | 4.0+ |
| UI Components | shadcn/ui | Latest |
| Database | Supabase (PostgreSQL) | Latest |
| Authentication | Supabase Auth | Latest |
| Deployment | Vercel | Latest |
| Package Manager | pnpm | 9.0+ |

## Architecture Notes

### Data Flow
1. **User Actions** �� React Components (CopyTrading, Portfolio, etc.)
2. **API Calls** → Supabase Client (`supabaseClient.js`)
3. **Database** → PostgreSQL (Supabase)
4. **Real-time Updates** → React Query with manual polling
5. **Error Handling** → Toast notifications via Sonner

### Key Files
- `src/api/supabaseClient.js` - Database connection
- `vercel.json` - Deployment configuration
- `src/schema.sql` - Database schema
- `.env.local` - Local development environment variables (never commit)

---

**Last Updated**: March 29, 2026
**Status**: Production Ready ✓
**No Breaking Changes**: All functionality preserved during Base44 → Supabase migration
