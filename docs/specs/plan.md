# Production Deployment Runbook — policy_management (develop branch)

> **Target:** `erp.globalhealthadvisory.net` on `/opt/frappe_docker` (production server)
> **Company:** Global Health Advisory (GHA)
> **Source:** Dev environment at `/opt/frappe_docker_dev` on this machine

## Variables

Set these before running any commands. Copy-paste this block into your terminal:

```bash
SSH_HOST="frapppe"                          # SSH alias or user@ip for production
PROD_FRAPPE_DIR="/opt/frappe_docker"        # frappe_docker directory on production
PROD_SITE="erp.globalhealthadvisory.net"    # Production site name
IMAGE_TAG="v16-policy"                      # New image tag (NOT v16 — keep old for rollback)
```

---

## Phase 1: Pre-Deployment Checks (on production)

### Step 1.1 — Connect and verify production is running

```bash
ssh $SSH_HOST "cd $PROD_FRAPPE_DIR && docker compose ps && docker compose exec backend bench --site $PROD_SITE list-apps"
```

**Expected:** All containers running, `policy_management` NOT in the app list yet.

### Step 1.2 — Take a FULL BACKUP (critical)

```bash
ssh $SSH_HOST "cd $PROD_FRAPPE_DIR && docker compose exec backend bench --site $PROD_SITE backup --with-files"
```

**Verify:** Backup files exist in `private/backups/` and are non-trivial in size.

### Step 1.3 — Record current state (for post-deployment comparison)

```bash
ssh $SSH_HOST << 'SSHEOF'
cd /opt/frappe_docker
SITE="erp.globalhealthadvisory.net"
echo "=== Company ==="
docker compose exec backend bench --site $SITE execute 'frappe.db.get_single_value("Global Defaults", "default_company")'
echo
echo "=== Chart of accounts ==="
docker compose exec backend bench --site $SITE execute 'frappe.db.get_value("Company", "Global Health Advisory", "chart_of_accounts")'
echo
echo "=== Record counts (SAVE THESE) ==="
for dt in Customer Supplier Account Employee "Sales Invoice" "Payment Entry"; do
  count=$(docker compose exec -T backend bench --site $SITE execute "frappe.db.count('$dt')" 2>/dev/null | tail -1)
  echo "  $dt: $count"
done
echo
echo "=== Group accounts (verify parent names) ==="
docker compose exec -T backend bench --site $SITE execute 'frappe.db.sql("SELECT name, account_name, root_type FROM `tabAccount` WHERE company=%s AND is_group=1 ORDER BY root_type", ("Global Health Advisory",), as_dict=True)' 2>/dev/null
SSHEOF
```

**Verify:** The group accounts include these parent names (the install script fuzzy-matches them):
- `Direct Income` (under Income)
- `Current Liabilities` (under Liability)
- `Duties and Taxes` (under Liability)
- `Indirect Expenses` (under Expense)
- `Direct Expenses` (under Expense)
- `Accounts Receivable` (under Asset)
- `Bank Accounts` (under Asset)

If any are missing, the install will print `SKIP <account>` — you can create it manually after.

### Step 1.4 — Check if setup wizard is complete

```bash
ssh $SSH_HOST "cd $PROD_FRAPPE_DIR && docker compose exec backend bench --site $PROD_SITE execute 'frappe.db.get_single_value(\"System Settings\", \"setup_complete\")'"
```

**Expected:** `1` (setup complete). If `0`, complete the setup wizard first.

---

## Phase 2: Prepare and Transfer the Source

### Step 2.1 — Create a clean tarball on the dev machine

```bash
cd /opt/frappe_docker_dev/dev-apps
tar --exclude='policy_management/policy_management/frontend/node_modules' \
    --exclude='policy_management/**/__pycache__' \
    --exclude='policy_management/.git' \
    --exclude='policy_management/docs' \
    -czf /tmp/policy_management_prod.tar.gz policy_management/
ls -lh /tmp/policy_management_prod.tar.gz
```

### Step 2.2 — Transfer to production

```bash
scp /tmp/policy_management_prod.tar.gz $SSH_HOST:/tmp/
scp /opt/frappe_docker_dev/Dockerfile.policy $SSH_HOST:/tmp/Dockerfile.policy
```

---

## Phase 3: Build the Production Image

### Step 3.1 — Set up the directory structure on production

```bash
ssh $SSH_HOST << 'SSHEOF'
cd /opt/frappe_docker
mkdir -p dev-apps
cd dev-apps
tar -xzf /tmp/policy_management_prod.tar.gz
echo "=== Extracted ==="
ls -la policy_management/
echo
echo "=== Verify key fix files ==="
echo "crm.py preferred_insurer: $(grep -c 'preferred_insurer' policy_management/policy_management/integrations/crm.py)"
echo "client_tier.py tier_rank: $(grep -c 'tier_rank' policy_management/policy_management/doctype/client_tier/client_tier.py)"
echo "daily_exchange_rate.py from_currency: $(grep -c 'from_currency' policy_management/policy_management/doctype/daily_exchange_rate/daily_exchange_rate.py)"
echo "install_erp_prerequisites.py get_single_value: $(grep -c 'get_single_value' policy_management/policy_management/setup/install_erp_prerequisites.py)"
SSHEOF
```

**Expected:** All counts should be > 0 (verifies the bug fixes are in the source).

### Step 3.2 — Copy Dockerfile.policy into the frappe_docker root

```bash
ssh $SSH_HOST "cp /tmp/Dockerfile.policy /opt/frappe_docker/Dockerfile.policy && cat /opt/frappe_docker/Dockerfile.policy"
```

### Step 3.3 — Build the image (this takes 5-10 minutes)

```bash
ssh $SSH_HOST "cd /opt/frappe_docker && docker build -f Dockerfile.policy -t mbugua/erpnext-custom:$IMAGE_TAG . 2>&1 | tail -20"
```

**Verify:** The build should end with `DONE` and no errors.

### Step 3.4 — Update .env to use the new image tag

```bash
ssh $SSH_HOST << SSHEOF
cd /opt/frappe_docker
cp .env .env.bak.$(date +%Y%m%d%H%M%S)
sed -i 's/^CUSTOM_TAG=.*/CUSTOM_TAG=$IMAGE_TAG/' .env
grep -q '^CUSTOM_TAG=' .env || echo 'CUSTOM_TAG=$IMAGE_TAG' >> .env
echo "Updated .env:"
grep CUSTOM_TAG .env
SSHEOF
```

---

## Phase 4: Deploy

### Step 4.1 — Stop production services

```bash
ssh $SSH_HOST "cd /opt/frappe_docker && docker compose down"
```

### Step 4.2 — Start with the new image

```bash
ssh $SSH_HOST "cd /opt/frappe_docker && docker compose up -d && sleep 20 && docker compose ps"
```

**Verify:** All containers should be `Up`. The configurator should have exited with code 0.

### Step 4.3 — Verify the site is accessible

```bash
ssh $SSH_HOST << 'SSHEOF'
SITE="erp.globalhealthadvisory.net"
echo "=== Login page ==="
curl -sI -H "Host: $SITE" http://127.0.0.1:8080/login | head -3
echo
echo "=== API ping ==="
curl -s -H "Host: $SITE" http://127.0.0.1:8080/api/method/ping
echo
echo "=== CSS bundle (should be 200) ==="
curl -sI -H "Host: $SITE" http://127.0.0.1:8080/assets/frappe/dist/css/login.bundle.*.css | head -3
SSHEOF
```

**Expected:** Login page returns 200, API returns `{"message":"pong"}`, CSS bundle returns 200.

### Step 4.4 — Install the app

```bash
ssh $SSH_HOST "cd /opt/frappe_docker && docker compose exec backend bench --site $PROD_SITE install-app policy_management 2>&1 | tail -30"
```

**Expected output:**
```
  Succeeded: 20
  Skipped:   0
  Failed:    0
```

If any steps show `[FAIL]`, read the error message. Common issues:
- `SKIP <account>: could not find parent account` — non-fatal, create the account manually later
- `Could not find Parent Customer Group: All Customer Groups` — run step 4.4b

### Step 4.4b — Fix missing ERPNext prerequisites (if needed)

If the install failed because "All Customer Groups", "All Item Groups", or "Nos" UOM are missing:

```bash
ssh $SSH_HOST << 'SSHEOF'
cd /opt/frappe_docker
SITE="erp.globalhealthadvisory.net"
echo "=== Creating missing prerequisites ==="
docker compose exec backend bench --site $SITE execute 'frappe.get_doc({"doctype":"Customer Group","customer_group_name":"All Customer Groups","is_group":1}).insert(ignore_permissions=True)' 2>&1 | tail -1
docker compose exec backend bench --site $SITE execute 'frappe.get_doc({"doctype":"Item Group","item_group_name":"All Item Groups","is_group":1}).insert(ignore_permissions=True)' 2>&1 | tail -1
docker compose exec backend bench --site $SITE execute 'frappe.get_doc({"doctype":"UOM","uom_name":"Nos","must_be_whole_number":1}).insert(ignore_permissions=True)' 2>&1 | tail -1
echo
echo "=== Re-running install ==="
docker compose exec backend bench --site $SITE execute policy_management.install.after_install 2>&1 | tail -10
SSHEOF
```

### Step 4.5 — Migrate

```bash
ssh $SSH_HOST "cd /opt/frappe_docker && docker compose exec backend bench --site $PROD_SITE migrate 2>&1 | tail -10"
```

### Step 4.6 — Enable scheduler and clear cache

```bash
ssh $SSH_HOST "cd /opt/frappe_docker && \
  docker compose exec backend bench --site $PROD_SITE enable-scheduler && \
  docker compose exec backend bench --site $PROD_SITE clear-cache && \
  docker compose exec backend bench --site $PROD_SITE scheduler status"
```

---

## Phase 5: Post-Deployment Verification

### Step 5.1 — Verify app installation

```bash
ssh $SSH_HOST << 'SSHEOF'
cd /opt/frappe_docker
SITE="erp.globalhealthadvisory.net"
echo "=== Apps ==="
docker compose exec backend bench --site $SITE list-apps
echo
echo "=== DocType count (should be 58) ==="
docker compose exec -T backend bench --site $SITE execute 'frappe.db.count("DocType", {"module": "Policy Management"})' 2>/dev/null | tail -1
echo
echo "=== Report count (should be 18) ==="
docker compose exec -T backend bench --site $SITE execute 'frappe.db.count("Report", {"module": "Policy Management"})' 2>/dev/null | tail -1
echo
echo "=== Workspace ==="
docker compose exec -T backend bench --site $SITE execute 'frappe.db.exists("Workspace", "Insurance")' 2>/dev/null | tail -1
echo
echo "=== Scheduled jobs (should be 11) ==="
docker compose exec -T backend bench --site $SITE execute 'frappe.db.count("Scheduled Job Type", {"method": ["like", "%policy_management%"]})' 2>/dev/null | tail -1
SSHEOF
```

### Step 5.2 — Verify the SPA

```bash
ssh $SSH_HOST << 'SSHEOF'
SITE="erp.globalhealthadvisory.net"
echo "=== SPA route (should be 403 = needs auth) ==="
curl -sI -H "Host: $SITE" http://127.0.0.1:8080/insurance | head -3
echo
echo "=== SPA assets (should be 200) ==="
curl -sI -H "Host: $SITE" http://127.0.0.1:8080/assets/policy_management/frontend/assets/index-*.js | head -3
echo
echo "=== App logo (should be 200) ==="
curl -sI -H "Host: $SITE" http://127.0.0.1:8080/assets/policy_management/images/logo.svg | head -3
SSHEOF
```

### Step 5.3 — Verify existing data is intact

```bash
ssh $SSH_HOST << 'SSHEOF'
cd /opt/frappe_docker
SITE="erp.globalhealthadvisory.net"
echo "=== Record counts (compare with Step 1.3) ==="
for dt in Customer Supplier Account Employee "Sales Invoice" "Payment Entry"; do
  count=$(docker compose exec -T backend bench --site $SITE execute "frappe.db.count('$dt')" 2>/dev/null | tail -1)
  echo "  $dt: $count"
done
SSHEOF
```

**Verify:** The counts should match the pre-deployment numbers from Step 1.3. If any count decreased, restore from backup immediately.

### Step 5.4 — Verify new insurance accounts were created

```bash
ssh $SSH_HOST << 'SSHEOF'
cd /opt/frappe_docker
SITE="erp.globalhealthadvisory.net"
echo "=== Insurance accounts (should have 16 new accounts) ==="
docker compose exec -T backend bench --site $SITE execute 'frappe.db.sql("SELECT name, account_name FROM `tabAccount` WHERE company=%s AND account_name IN (\"Brokerage Income\", \"Premiums Collected\", \"Premium Trust Payable\", \"Insurer Premium Payable\", \"Commission Payable\", \"Stamp Duty Payable\", \"WHT Payable\", \"Bad Debts\", \"WHT Expense\", \"IRA Levy Expense\", \"Agent Commission Expense\", \"Introducer Fee Expense\", \"Forex Gain Loss\", \"Insurance Debtors\", \"Commission Receivable\", \"Premium Trust Account\", \"Operating Account\")", ("Global Health Advisory",), as_dict=True)' 2>/dev/null
SSHEOF
```

---

## Phase 6: Go-Live Configuration

### Step 6.1 — Assign insurance roles to users (Desk UI)

1. Go to **User** list in Desk
2. For each insurance staff member, add the appropriate role:
   - **Insurance Manager** — full access to all insurance records
   - **Insurance Agent** — row-level restricted to own records only
   - **Insurance Accounts** — financial functions (invoicing, commissions)
   - **Insurance Claims Handler** — claim management
   - **Insurance Compliance** — compliance and regulatory functions
   - **Insurance Viewer** — read-only access

> **WARNING:** Users with "Insurance Agent" role will ONLY see policies, claims, and commissions where they are the `originating_agent`. Do not assign this role to managers or supervisors.

### Step 6.2 — Configure Insurance Settings (Desk UI)

1. Go to the **Insurance** workspace
2. Open **Insurance Settings** (Single doctype)
3. Set the following:
   - **Trust Account** → `Premium Trust Account - GHA`
   - **Operating Account** → `Operating Account - GHA`
   - **Commission Receivable Account** → `Commission Receivable - GHA`
   - **Brokerage Income Account** → `Brokerage Income - GHA`
   - **Premiums Collected Account** → `Premiums Collected - GHA`
   - **Stamp Duty Payable Account** → `Stamp Duty Payable - GHA`
   - **WHT Payable Account** → `WHT Payable - GHA`
   - **Agent Commission Expense Account** → `Agent Commission Expense - GHA`
   - **IRA Levy Rate** → (as applicable, e.g., 0.25%)
   - **WHT Rate** → (as applicable, e.g., 5%)
   - **Stamp Duty Rate** → (as applicable)

### Step 6.3 — Test the SPA

1. Navigate to `https://erp.globalhealthadvisory.net/insurance`
2. Login with an Insurance Manager user
3. Verify:
   - Dashboard loads with charts
   - Policy list loads
   - Claim list loads
   - Commission list loads
   - Can create a new policy quotation

### Step 6.4 — Monitor for 24 hours

```bash
# Check scheduler logs
ssh $SSH_HOST "cd /opt/frappe_docker && docker compose logs --tail=50 scheduler"

# Check backend error logs
ssh $SSH_HOST "cd /opt/frappe_docker && docker compose logs --tail=50 backend 2>&1 | grep -i error"
```

---

## Rollback Plan

### Quick Rollback (app only)

```bash
ssh $SSH_HOST << 'SSHEOF'
cd /opt/frappe_docker
SITE="erp.globalhealthadvisory.net"
echo "=== Uninstalling policy_management ==="
docker compose exec backend bench --site $SITE uninstall-app policy_management
echo
echo "=== Reverting to old image ==="
cp .env.bak.* .env
docker compose down
docker compose up -d
SSHEOF
```

### Full Rollback (restore from backup)

```bash
ssh $SSH_HOST << 'SSHEOF'
cd /opt/frappe_docker
SITE="erp.globalhealthadvisory.net"
echo "=== Stopping services ==="
docker compose down
echo
echo "=== Restoring from backup ==="
# Find the latest backup file
BACKUP_DIR="/home/frappe/frappe-bench/sites/$SITE/private/backups"
LATEST=$(docker compose run --rm backend bash -c "ls -t $BACKUP_DIR/*.sql.gz 2>/dev/null | head -1")
echo "Restoring: $LATEST"
docker compose run --rm backend bench --site $SITE restore "$LATEST"
echo
echo "=== Reverting image ==="
cp .env.bak.* .env
docker compose up -d
SSHEOF
```

---

## Summary Checklist

- [ ] Step 1.1 — Production is running, policy_management not yet installed
- [ ] Step 1.2 — Full backup taken and verified
- [ ] Step 1.3 — Current state recorded, parent accounts verified
- [ ] Step 1.4 — Setup wizard is complete
- [ ] Step 2.1 — Clean tarball created
- [ ] Step 2.2 — Tarball transferred to production
- [ ] Step 3.1 — Source extracted and fix files verified
- [ ] Step 3.2 — Dockerfile.policy in place
- [ ] Step 3.3 — Image built successfully
- [ ] Step 3.4 — .env updated with new tag
- [ ] Step 4.1 — Services stopped
- [ ] Step 4.2 — Services started with new image
- [ ] Step 4.3 — Site accessible, CSS/API working
- [ ] Step 4.4 — App installed (20/20 succeeded)
- [ ] Step 4.5 — Migrate completed
- [ ] Step 4.6 — Scheduler enabled, cache cleared
- [ ] Step 5.1 — 58 DocTypes, 18 reports, workspace, 11 jobs
- [ ] Step 5.2 — SPA route returns 403, assets return 200
- [ ] Step 5.3 — Existing data counts match pre-deployment
- [ ] Step 5.4 — 16 insurance accounts created
- [ ] Step 6.1 — Roles assigned to users
- [ ] Step 6.2 — Insurance Settings configured
- [ ] Step 6.3 — SPA tested and working
- [ ] Step 6.4 — No errors in 24-hour monitoring
