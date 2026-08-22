# Remediation Plan — bauenliving.com Gambling-Redirect Breach

**Case:** 2026-0822
**Site:** bauenliving.com (Plesk, 163.44.196.32)
**Root cause:** Black-hat SEO "doorway page" kit planted in the `vt.bauenliving.com` subdomain, live since ~18 Feb 2025. It 302-redirects any visitor arriving from a Google search to an illegal gambling site.
**Malicious location:** `/var/www/vhosts/bauenliving.com/vt.bauenliving.com/video/`
**Status at time of writing:** Redirect still live and reproducible. Main WordPress site is clean.

> **Verified reproduction**
> - Request **with** a `google.com` referer → `302 Found → https://kutt.arrehlah.com/alexsagapung`
> - Request **without** referer → `200 OK` (normal page)
> This conditional cloaking is why the site looks clean on casual checks.

---

## How to use this plan

Work top to bottom. **Phase 1 is urgent** — it stops the active redirect and locks attackers out. Phases 2–4 clean up the search-engine damage and close the hole that let this happen. Check each box as you go.

Take a backup of the `video/` directory before deleting, in case it's needed as evidence:
```bash
tar -czf ~/video-kit-evidence-2026-0822.tar.gz \
  -C /var/www/vhosts/bauenliving.com/vt.bauenliving.com video
```

---

## Phase 1 — Stop the bleeding (do today)

### 1.1 Remove the doorway kit
The entire `video/` directory is malicious. Remove it (or take the whole `vt` subdomain offline if it isn't in active use).

```bash
# after taking the evidence backup above
rm -rf /var/www/vhosts/bauenliving.com/vt.bauenliving.com/video
```

- [ ] `video/` directory removed
- [ ] Confirm redirect no longer fires:
  ```bash
  curl -I -A "Mozilla/5.0" -e "https://www.google.com/search?q=pg+slot" \
    "https://vt.bauenliving.com/video/?website=pg+slot"
  # Expect: 404 Not Found (not 302)
  ```

### 1.2 Rotate every credential on the account
Treat everything issued before today as compromised — the FTP password was already dead when the investigation started.

- [ ] **FTP** password (`bauen` user) — reset in Plesk
- [ ] **Plesk** login (`root`) — change password
- [ ] **SSH / system root** password
- [ ] **MySQL** DB user `wp_zkxje` — reset, then update `DB_PASSWORD` in `wp-config.php`
- [ ] **WordPress admins** — force a password reset for `odin` and `Juthatip`
- [ ] Update the local `ftp.md` notes file with the new credentials (and keep it out of any synced/committed location)

### 1.3 Confirm the WordPress admin accounts
Three users exist; confirm each is legitimate.

| ID | Login | Role | Registered | Action |
|----|-------|------|-----------|--------|
| 4 | worrasing | editor | 2023-08-23 | Confirm still needed |
| 10 | odin | administrator | 2023-06-02 | Confirm — this is the main admin |
| 11 | **Juthatip** | administrator | **2026-04-20** | **Verify with your team — disable if unrecognized** |

- [ ] `Juthatip` account confirmed as a real team member, **or** disabled/deleted

---

## Phase 2 — Undo the search-engine damage (this week)

The attacker verified ownership of the subdomain in Google Search Console and submitted **14,072 spam URLs** for indexing. These must be removed or they will keep surfacing your domain in gambling searches.

- [ ] Log into **Google Search Console** for the account tied to bauenliving.com
- [ ] Find and **remove the unrecognized property / verification** for `vt.bauenliving.com` (the planted file was `googlec6c682899ba0e5d5.html`)
- [ ] Delete the planted verification file if it still exists on disk:
  ```bash
  rm -f /var/www/vhosts/bauenliving.com/vt.bauenliving.com/video/googlec6c682899ba0e5d5.html
  # (already gone if you removed the whole video/ dir in 1.1)
  ```
- [ ] Submit the spam URL pattern for **removal** via Search Console → *Removals*
  (target: `https://vt.bauenliving.com/video/`)
- [ ] Once `video/` returns 404, the pages will drop from the index over time — monitor `site:vt.bauenliving.com` in Google over the following weeks

---

## Phase 3 — Verify nothing else is infected (this week)

- [ ] Run a full malware scan across the **whole server**, not just this account (Imunify360 and Revisium are already installed — ask your host to widen the scan):
  ```bash
  imunify360-agent malware malicious list
  ```
- [ ] Search the account for any other doorway-kit artifacts:
  ```bash
  find /var/www/vhosts/bauenliving.com \
    \( -iname 'thaikeyword*' -o -iname 'param.txt' -o -iname 'sitemap.xml' \) 2>/dev/null
  ```
- [ ] Confirm no PHP files hiding in uploads outside known caches:
  ```bash
  find /var/www/vhosts/bauenliving.com/httpdocs/wp-content/uploads \
    -iname '*.php*' 2>/dev/null | grep -v '/cache/wpml/'
  ```
- [ ] Scan for obfuscated payloads in WordPress code:
  ```bash
  grep -rlIE 'eval\s*\(\s*(base64_decode|gzinflate|str_rot13)' \
    /var/www/vhosts/bauenliving.com/httpdocs/wp-content --include='*.php'
  ```

**Already verified clean during investigation** (re-check only if you want fresh confidence): WordPress core, all 20 active plugins, Avada theme, `wp_options`/posts/widgets in the DB, root `.htaccess`, server cron, and vhost config.

---

## Phase 4 — Close the hole (this month)

- [ ] **Find the entry point.** Pull server access logs from around **February 2025** to see how the `video/` directory was first written:
  ```bash
  zgrep -i "video/" /var/www/vhosts/system/bauenliving.com/logs/access_ssl_log.processed.*.gz \
    | grep -iE "POST|PUT" | head -50
  ```
- [ ] **Decommission or lock down `vt.bauenliving.com`** if the SCORM/e-learning content it hosts is no longer in use — it was the exploited entry point, not the main site.
- [ ] **Update everything**: WordPress core, all plugins, and themes to current versions.
- [ ] **Add file-integrity monitoring** on subdomain docroots *outside* the main WordPress install. This kit sat undetected for 18 months precisely because it lived outside the WordPress tree where scanners were focused.
- [ ] Review Plesk for any leftover FTP sub-accounts, scheduled tasks, or additional subscriptions that shouldn't be there.

---

## Sign-off checklist

- [ ] Phase 1 complete — redirect stopped, credentials rotated
- [ ] Phase 2 complete — Search Console cleaned, removals submitted
- [ ] Phase 3 complete — server-wide scan clean
- [ ] Phase 4 complete — entry point identified and closed
- [ ] Confirmed `site:vt.bauenliving.com` returns no gambling spam in Google

---

## Indicators of Compromise (IOCs) — for reference

| Type | Value |
|------|-------|
| Malicious directory | `vt.bauenliving.com/video/` |
| Redirect target | `https://kutt.arrehlah.com/alexsagapung` |
| Trigger condition | `HTTP_REFERER` matching `google.` |
| Planted GSC verify file | `googlec6c682899ba0e5d5.html` |
| Keyword dictionary | `thaikeyword.txt` (14,071 Thai gambling terms) |
| Page generator | `index.php` (458 KB), param `?website=` |
| Spam sitemap | `sitemap.xml` (14,072 URLs) |
| First planted | ~18 Feb 2025 |
| Imunify signature | `SMW-BLKH-SA-CLOUDAV-php.tool.spam-AUTO10-3` |
| Suspicious WP admin | `Juthatip` (ID 11, added 2026-04-20) |
