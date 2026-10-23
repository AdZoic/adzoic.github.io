---
title: "PHP2Shell: find every vulnerable PHP install in one command"
description: "We built php2shell-check, a free single-binary scanner that finds every PHP runtime on a server or container image, tells you if it is exposed to PHP2Shell, and what to do next."
image: /assets/img/php2shell-card.png
tags: [PHP, CVE-2026-109618, CVE-2026-109619, Open source]
---

![Animated demo of php2shell-check scanning a Linux host](/assets/img/demo.svg)

**PHP2Shell** (CVE-2026-109618 and CVE-2026-109619) is a pair of critical, unauthenticated remote code execution flaws in PHP. They are triggered while PHP parses `multipart/form-data`, before a single line of your application runs. Your framework, your WAF rules in the app, your input validation: none of it gets a say.

It affects PHP-FPM, mod_php, php-cgi, FrankenPHP and LiteSpeed, back to at least PHP 5. The fixes shipped in **8.5.12, 8.4.27, 8.3.36 and 8.2.35**. Anything older is end-of-life and will not get an upstream fix.

The advice is simple: patch. The hard part is knowing *what* to patch.

## Why "just check the version" is not enough

On a real fleet, `php -v` on one box tells you very little:

- **There is rarely one PHP.** A cPanel or Plesk server can have six. Add Remi, SCL, Homebrew, Docker images and a forgotten `/opt/php74`, and nobody has the full list.
- **Distro back-ports lie to version scanners.** Ubuntu and RHEL patch the CVE but keep the old number. `8.3.6-0ubuntu0.24.04.13` is safe; a version-only scanner calls it vulnerable and teaches your team to ignore alerts.
- **Patched on disk is not patched in memory.** After `apt upgrade`, PHP-FPM workers keep running the old binary until they are restarted. This is the most common post-patch mistake.
- **Mitigations are per pool.** The interim setting `session.upload_progress.enabled=0` only helps if it is in effect in *every* FPM pool.

## What php2shell-check does

[php2shell-check](https://github.com/AdZoic/php2shell-check) is a single static Go binary with no dependencies, for Linux, macOS, Windows and FreeBSD on amd64 and arm64. Run it and it:

1. **Finds every PHP runtime**: `$PATH`, distro packages, Docker official images, Remi, SCL, cPanel EA-PHP, Plesk, CloudLinux, LiteSpeed, Homebrew, MAMP, Herd, XAMPP, Laragon, Scoop and more.
2. **Checks vendor back-ports** by reading the dpkg or rpm changelog, so patched distro builds show as `PATCHED_BACKPORT` instead of a false alarm.
3. **Finds stale workers** on Linux: php-fpm or `libphp.so` processes still running a replaced or deleted binary.
4. **Checks FPM pool mitigations** across every pool, including `php_admin_value` overrides.
5. **Fails closed.** Anything it cannot prove is patched is reported as needing attention, never as safe.

| Verdict | Meaning |
|---|---|
| `PATCHED` | At or above the fixed release |
| `PATCHED_BACKPORT` | Older number, but the vendor changelog lists both CVEs |
| `VULNERABLE` | Below the fixed release, no back-port evidence |
| `VULNERABLE_EOL` | Branch older than 8.2: no fix will ever ship |
| `UNVERIFIED` | Vendor build whose changelog could not be read |

## Try it

```bash
# Linux
curl -fsSLO https://github.com/AdZoic/php2shell-check/releases/latest/download/php2shell-check_linux_amd64
chmod +x php2shell-check_linux_amd64
sudo ./php2shell-check_linux_amd64

# Container images, from any OS with Docker or Podman
php2shell-check image php:8.3-fpm my-registry/app:prod
```

Exit codes are `0` safe, `1` vulnerable, `2` needs attention and `3` scan error, so it can gate a CI pipeline directly. There is also a GitHub Action:

```yaml
- uses: AdZoic/php2shell-check@v1
  with:
    images: my-registry/app:${{ "{{" }} github.sha {{ "}}" }}
```

## Safe by design

php2shell-check is a defensive tool. It only runs `php -v`, `php -i` and package-manager queries. It never executes PHP code, never sends requests to your servers and contains **no exploit code**. Image scans run in a throwaway container with no network, all capabilities dropped and a read-only filesystem, and the container is always removed afterwards.

## What to do today

1. Run php2shell-check on every server and in CI against every image you ship.
2. Upgrade anything `VULNERABLE` to 8.5.12, 8.4.27, 8.3.36 or 8.2.35 or later, or install your vendor's patched package.
3. **Restart PHP-FPM and your web server**, then run the scan again to confirm no stale workers remain.
4. Plan a migration off anything `VULNERABLE_EOL`. Interim settings reduce risk; only upgrading removes it.

The code is MIT-licensed on [GitHub](https://github.com/AdZoic/php2shell-check). Issues and pull requests are welcome, especially for PHP install locations we do not detect yet.
