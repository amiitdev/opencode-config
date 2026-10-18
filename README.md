# Why OpenCode's free tier worked on the laptop but not the phone

> **Debugging field notes** — a full, teacher-style walkthrough of one real session:
> from *"it just doesn't work over SSH"* all the way down to OpenCode Zen's per-IP rate limiter.

An interactive HTML version of these notes lives in [`index.html`](./index.html).

---

## Contents

1. [The Mission & the Symptom](#1-the-mission--the-symptom)
2. [The Starting Setup](#2-the-starting-setup)
3. [Two Different "OpenCode"s (v1 vs v2)](#3-two-different-opencodes-v1-vs-v2)
4. [Decoding the Error Message](#4-decoding-the-error-message)
5. [Attempt 1 — Upgrade the Phone to v2](#5-attempt-1--upgrade-the-phone-to-v2)
6. [Attempt 2 — Why v2 Can't Run on Termux](#6-attempt-2--why-v2-cant-run-on-termux)
7. [OmniRoute — "free router" that fails](#7-omniroute--free-router-that-fails)
8. [Same Wi-Fi = One Shared Quota](#8-same-wi-fi--one-shared-quota)
9. [Inside the Laptop's Working VPN](#9-inside-the-laptops-working-vpn)
10. [The Phone's VPN Was Doing Nothing](#10-the-phones-vpn-was-doing-nothing)
11. [The IPv6 Leak](#11-the-ipv6-leak)
12. [Proton VPN Settings That Matter](#12-proton-vpn-settings-that-matter)
13. [Reaching the Phone over USB (ADB)](#13-reaching-the-phone-over-usb-adb)
14. [The Decisive Experiments](#14-the-decisive-experiments)
15. [Server Roulette — The Exact IP Decides](#15-server-roulette--the-exact-ip-decides)
16. [Two Devices, Two Buckets — or One?](#16-two-devices-two-buckets--or-one)
17. [The "VPN Trick" — Why It Keeps Working After Disconnect](#17-the-vpn-trick--why-it-keeps-working-after-disconnect)
18. [Under the Hood — OpenCode Zen's Limiter](#18-under-the-hood--opencode-zens-limiter)
19. [The Toolbox](#19-the-toolbox)
20. [Decision Tree — "Free Tier Fails"](#20-decision-tree--free-tier-fails)
21. [Takeaways & Final State](#21-takeaways--final-state)

## 1. The Mission & the Symptom

**Goal:** use **OpenCode** (an AI coding agent) from a phone over SSH, exactly like it works on the laptop — including its **free models**.

**Symptom:** OpenCode launched fine on the phone, but *every* request failed:

```
AI_APICallError: OpenCode's free tier can only be used from within OpenCode.
Download it at https://opencode.ai and keep building.
```

The same command worked on the laptop — *same account, same model, same network.*

**What these notes teach:**

- How to read a misleading error message
- Why two devices on the same Wi-Fi share failure
- Two different "OpenCode" versions (v1 vs v2) and which runs on Android
- VPN mechanics: tunnels, IPv6 leaks, LAN blocking, server IPs
- Per-IP rate limiting and how to reliably find a working server
- A repeatable debugging method: isolate one variable at a time

---

## 2. The Starting Setup

```
  LAPTOP (Linux)                     PHONE (Android / Termux)
  192.168.29.133                     192.168.29.192
  opencode v2.0.29 (@opencode/cli)   opencode v1.18.22 (standalone binary)
        |                                    ^
        +---------- ssh -p 8022 -------------+
             (Termux sshd listens on 8022)
```

First contact:

```bash
ssh -p 8022 u0_a395@192.168.29.192 \
  'echo CONNECTED; uname -a; which opencode; opencode --version'
```

SSH worked — so the problem was **inside** OpenCode, not the connection.

| Check | Result |
|---|---|
| `uname -a` | aarch64 **Android** (not plain Linux!) |
| `which opencode` | `~/.opencode/bin/opencode` (standalone ELF) |
| `opencode --version` | **1.18.22** (laptop was 2.0.29) |

**Side-quest — sshd kept dying.** Termux's SSH server repeatedly died (Android battery killer). Fix on the phone:

```bash
termux-wake-lock
sshd
```

## 3. Two Different "OpenCode"s (v1 vs v2)

A **huge** source of confusion: there are two major product lines.

| | OpenCode 1 (v1) | OpenCode 2 (v2) |
|---|---|---|
| npm package | `opencode-ai` (latest 1.19.3) | `@opencode/cli` (2.0.32) |
| Install | curl script → `~/.opencode/bin` | npm global / `/v2/install` |
| Runtime | single Bun binary | multi-process (CLI + `serve`) |
| Laptop had | — | **2.0.29** |
| Phone had | **1.18.22** | — |

The installer trick:

```bash
curl -fsSL https://opencode.ai/install     # installs v1 ONLY
curl -fsSL https://opencode.ai/v2/install  # installs v2
```

The v1 installer **refuses** v2 versions on purpose (and vice-versa). Reading the installer script is how we discovered the `/v2/install` path.

---

## 4. Decoding the Error Message

The message *sounds* like a client/version problem, but it is reused for **several** different server-side gates. This is the #1 lesson.

```
"OpenCode's free tier can only be used from within OpenCode."
```

The same text is returned for (at least) two unrelated reasons:

- **Gate 1 — Client identity:** "Is the caller a genuine OpenCode client?" A third-party proxy (OmniRoute) or a raw `curl` fails this.
- **Gate 2 — Network / IP:** "From which IP is it asking?" Rate limits, reputation and region are all keyed on the egress IP.

```
LAPTOP  opencode v2 --direct--> Zen   identity OK   IP OK/FAIL
PHONE   opencode v1 -> OmniRoute -> Zen   identity FAIL   IP FAIL
        |
        +- VPN / fresh IP fixes the IP gate...
           ...but NOT the identity gate.  Same error text!
```

---

## 5. Attempt 1 — Upgrade the Phone to v2

npm refuses:

```
npm i -g @opencode/cli@latest

npm error code EBADPLATFORM
npm error notsup Unsupported platform for @opencode/cli@2.0.32:
  wanted {"os":"darwin,linux,win32","cpu":"arm64,x64"}
  (current: {"os":"android","cpu":"arm64"})
```

npm reports the phone's OS as `android`, and the package only ships binaries for darwin/linux/win32. There is **no `-android` build**.

Reading the installer saved us — Termux *is* Linux underneath, so the installer mapped `uname -s = Linux` + `aarch64` → `linux-arm64` and pulled a **glibc** build:

```bash
curl -fsSL https://opencode.ai/v2/install | bash
# installed: ~/.opencode/bin/opencode  (v2.0.32, ELF aarch64, glibc)
```

## 6. Attempt 2 — Why v2 Can't Run on Termux

v2 installed, printed a version, then crashed as soon as it needed its server:

```
Error: Server process exited with code 127
serve: error while loading shared libraries: serve: cannot open shared object file
```

**The underlying reason:**

- Android uses **bionic libc**, not glibc. The binary asks for `/lib/ld-linux-aarch64.so.1`, which **doesn't exist** on Android.
- Termux bridges the gap with **glibc-runner** (`grun`) — a shim that launches glibc binaries via the Termux glibc loader.
- v1 is a **single process**, so the shim is enough.
- v2 **re-execs itself** as a child (`serve`). Under the shim the process image is the *loader*, so the child spawn resolves wrong → exit 127.

```
v1:  [opencode]-----------------------> works
v2:  [opencode]--spawns--> [serve] X broken through glibc shim
```

Decision — roll back to v1:

```bash
mv ~/.opencode/bin/opencode        ~/.opencode/bin/opencode.v2.broken
mv ~/.opencode/bin/opencode.v1.bak ~/.opencode/bin/opencode   # back to v1.18.22
```

---

## 7. OmniRoute — "free router" that fails

The phone's config pointed its default model at `omniroute/auto/best-coding`. OmniRoute is a local proxy on the phone:

```
opencode --> OmniRoute (localhost:20128) --> OpenCode Zen free models
             blindly forwards                        |
                          "are you the real client?" +-- FAIL 403
```

**The key misunderstanding:** OmniRoute is **free software**, but it owns no models — it forwards to upstream "free" providers. The **free quota belongs to OpenCode Zen**, and Zen only serves the genuine client. A proxy trips **Gate 1 (identity)** — a VPN can't fix that.

Live proof:

```bash
curl localhost:20128/v1/chat/completions -d '{"model":"auto/best-coding",...}'
# {"error":{"message":"[403]: OpenCode's free tier can only be used from within
#  OpenCode... [oc/north-mini-code-free (400), felo/felo-chat (400) ...]",
#  "type":"permission_error"}}
```

OmniRoute itself was healthy (process alive, port answering). The rejection came from upstream.

Full removal:

```bash
sv down omniroute; rm -rf $PREFIX/var/service/omniroute   # stop + disable service
pkill -f "omn[i]route"                                    # kill processes
npm uninstall -g omniroute                                # remove package
rm -rf ~/.omniroute $PREFIX/var/log/omniroute             # remove data
jq 'del(.provider.omniroute)' ~/.config/opencode/opencode.json  # clean config
```

> **Gotcha:** `pkill -f omniroute` also matches your own SSH command line and kills your session. Use a bracket pattern like `omn[i]route`.

## 8. Same Wi-Fi = One Shared Quota

**The student's observation:** *"When I use up the free quota on either device, it stops working on **both**. But if I switch the phone to mobile data, it works again."*

**Why — the quota is per public IP:**

```
SAME WI-FI
  laptop 192.168.29.133 +
  phone  192.168.29.192 +--> ONE public IP --> ONE shared bucket
                                              either device drains it
                                              -> BOTH get 403

PHONE ON 4G/5G --> different public IP --> its own fresh bucket ✅
LAPTOP ON VPN  --> different public IP --> its own fresh bucket ✅
```

| Setup | Public IP | Result |
|---|---|---|
| Laptop on Wi-Fi | home | works until drained |
| Phone on same Wi-Fi | home (same!) | ❌ both die together |
| Phone on mobile data | carrier (new) | ✅ fresh bucket |
| Laptop on VPN | VPN (new) | ✅ fresh bucket |

**Conclusion:** one public IP = one daily free allowance. VPN, mobile data and hotspot are all the same trick — they change your egress IP.

---

## 9. Inside the Laptop's Working VPN

How to see a VPN on Linux:

```bash
ip -brief addr show          # look for tun0 / wg0 / proton0
ip route get 1.1.1.1         # which interface carries traffic?
curl -4 https://ifconfig.me  # the public IP everyone else sees
```

```
proton0  UNKNOWN  10.2.0.2/32
1.1.1.1 dev proton0 ...      <- default goes through the tunnel
egress: 212.8.253.37         <- the "outside" IP
```

Which server? (the big find)

```bash
grep -i endpoint \
  "/run/NetworkManager/system-connections/ProtonVPN NL-FREE#250.nmconnection"
# endpoint=212.8.253.22:443   <- server 250
# egress 212.8.253.37         <- its public IP
```

**Why the laptop's VPN "just worked":** it is a **full tunnel** — every app, every socket — and it also **blocks IPv6** (which matters a lot next).

## 10. The Phone's VPN Was Doing Nothing

Which VPN apps were installed?

```bash
pm list packages | grep -iE "vpn|proxy|wireguard|proton"
# package:com.shuttlevpn.free.proxy.gaming   <- "proxy gaming" VPN
# package:ch.protonvpn.android               <- real Proton VPN
# package:com.android.vpndialogs
```

A modded Proton APK (`Proton VPN v5.20.74.0 ... Mod signed.apk`) was also found in Downloads.

**The smoking gun:**

```
VPN OFF -> curl -4 ifconfig.me -> 49.47.132.205
VPN ON  -> curl -4 ifconfig.me -> 49.47.132.205   <- identical!
```

The "connected" VPN changed **nothing**. So the phone kept using the home IP (and its exhausted quota).

**Why:**

- ShuttleVPN-type apps are **proxy** VPNs: they route selected apps (browsers/games), not raw sockets from Termux.
- Mods often unlock the UI but not the real routing engine.
- Result: a "Connected" badge, but no `tun` interface covering Termux.

**Rule to remember:**

```
A VPN "works" only if curl shows a DIFFERENT public IP.
If the IP is unchanged, the VPN is not covering that app.
```

---

## 11. The IPv6 Leak

Once IPv4 was tunneled, opencode *still* failed. The reason: **IPv6 was leaking** straight out the home ISP.

```
opencode.ai  AAAA -> 2606:4700:78::90:0:143   (IPv6, Cloudflare)
             A    -> 172.65.90.20              (IPv4)

phone default connection -> IPv6   <- leaked home path -> 403
```

How we measured it:

```bash
curl -4 https://ifconfig.me   # IPv4 egress
curl -6 https://ifconfig.me   # IPv6 egress  <- the leak lives here
```

| Device | IPv4 | IPv6 |
|---|---|---|
| Laptop (VPN) | 212.8.253.37 | (empty — blocked ✅) |
| Phone (early) | 49.47.132.205 ❌ | 2405:201:a419:… (home!) |

**Fixes:** use a VPN that blocks/routes IPv6 (Proton does); Proton VPN → Settings → Connection → Advanced → **IPv6** toggle. Goal: `curl -6` returns *nothing*.

> **A wrong turn worth learning:** we tried pinning `opencode.ai` to an IPv4 in Termux's `/etc/hosts`. It **failed** — Termux's curl ignored it and still used IPv6. Lesson: don't fight the resolver; fix the tunnel.

## 12. Proton VPN Settings That Matter

| Setting | Where | Why |
|---|---|---|
| **Split tunneling = OFF** | Settings → Split Tunneling | Otherwise Termux may be excluded from the tunnel |
| **IPv6** | Connection → Advanced → IPv6 | OFF = block IPv6 (no leak); ON = tunnel it |
| **Kill switch** | Settings | Stops leaks if the tunnel drops |
| **Always-on VPN** | Android → VPN → gear | Forces all apps through the tunnel |
| **Block without VPN** | Android → VPN → gear | No traffic escapes if the VPN is down |
| **LAN connections** *(paid)* | Connection → Advanced | Keeps local network (SSH to the phone) reachable while on VPN |

**Surprise: VPN broke our SSH.** A full tunnel routes *everything* — including the LAN — so `ssh 192.168.29.192` from the laptop started **timing out**. That's expected, not a bug. Two ways around it: enable **LAN connections** (paid), or tunnel SSH over USB (next section — free and reliable).

---

## 13. Reaching the Phone over USB (ADB)

With the VPN fully on, the LAN route to the phone is blocked → no SSH. So we used **USB**:

```
phone:8022  <-- adb forward (USB cable) -->  laptop:8022

  ssh -p 8022 u0_a395@127.0.0.1     <- reaches Termux via the cable
```

Setup:

1. Phone → About → tap **Build number** ×7
2. Developer options → **USB debugging** ON
3. Plug in the cable, tap **Allow** on the phone

```bash
adb devices                       # must show the phone
adb forward tcp:8022 tcp:8022     # host:8022 -> device:8022
ssh -p 8022 u0_a395@127.0.0.1     # now SSH works despite the VPN
```

**Bonus — `adb reverse`** lets the *phone* use a service on the laptop:

```bash
adb reverse tcp:8888 tcp:8888
# phone's localhost:8888 now forwards to the laptop's localhost:8888
```

We used this exact trick to prove the phone's client was fine (next section).

## 14. The Decisive Experiments

**1. Is it just a header? → No**

```bash
for ua in "opencode/2.0.29" "opencode/1.18.22" "curl/8.0"; do
  curl -s $ZEN -H "User-Agent: $ua" -d "$BODY"
done
# ALL three -> 403 FreeTierError
```

Even the laptop's exact User-Agent failed from raw curl. So the gate is **not** a simple header — it needs the full genuine client shape.

**2. Route the phone through the laptop's IP.** We ran a tiny CONNECT proxy on the laptop (exits via its VPN), exposed it to the phone with `adb reverse`, and pointed opencode at it:

```bash
# laptop: minimal HTTP/CONNECT proxy on 127.0.0.1:8888
# phone:
HTTPS_PROXY=http://127.0.0.1:8888 \
  opencode run --model opencode/deepseek-v4.1-flash-free "reply exactly: PROXY-OK"
# -> PROXY-OK ✅
```

**Conclusion:** same phone, same v1.18.22, same free model — the **only** change was the egress IP.

> ✅ **The phone's client is fine. The problem is 100% the egress IP.**

---

## 15. Server Roulette — The Exact IP Decides

Same phone, same app, only the **server** changed each time:

| Server / country | Egress IPv4 | Result |
|---|---|---|
| Norway (Oslo) | 205.147.17.13 | ❌ 403 |
| Netherlands (some server) | 185.165.240.77 | ❌ 403 |
| Netherlands ~#250 | 212.8.253.87 | ✅ works |
| Laptop #250 | 212.8.253.37 | ✅ works |

**The lesson:** it's **not** the country and **not** the client. Each VPN server has its own public IP, and each IP has its own (shared) free bucket. You drew a bad IP, then a good one. That's the whole "server roulette".

## 16. Two Devices, Two Buckets — or One?

Zen keys the limiter on the **full IPv4 address**:

| Public IPs | Quota |
|---|---|
| Same IP (e.g. both on home Wi-Fi) | **ONE shared bucket** |
| Different IPs (laptop .37, phone .87) | **TWO independent buckets** |

Answering the class's question:

- On the same server but different IPs → **separate** quotas. Laptop running out does **not** kill the phone.
- Both on the same Wi-Fi → **shared** quota (that's why they died together earlier).
- Caveat: if the VPN NATs both devices through the *same* egress IP, they share again.

```bash
# quick check on each device — must be DIFFERENT to be independent:
curl -s -4 https://ifconfig.me
```

---

## 17. The "VPN Trick" — Why It Keeps Working After Disconnect

**Observation:** *"I connect the VPN for a few minutes, use opencode, disconnect — and it still works on my normal IP."*

**Why — the limit is a *decaying* meter, not a ban.** Zen runs several limiters; the one you usually trip is short-window:

| Limiter | Window | Recovery |
|---|---|---|
| Per-IP daily request count | ~24 h | slow (daily reset) |
| **TPM / TPS** (tokens per min/sec) | seconds–minutes | **fast** |
| Trial token budget | lifetime | never |

```
use opencode hard -> home IP trips rate limit -> 403
connect VPN       -> traffic leaves via VPN IP
                     home IP sits IDLE  -> its counter decays
disconnect VPN    -> home IP is under the limit again -> works ✅
```

The VPN isn't "unblocking" your IP — it's **parking** it so the meter resets. Confirmed live: phone back on home IP (no VPN) → `NOW-OK`.

**Takeaways:**

- Intermittent 403 on one IP = you burst → wait a minute or two.
- Persistent 403 = that IP's shared pool is saturated → switch server.
- Light/steady use often needs no VPN at all.

## 18. Under the Hood — OpenCode Zen's Limiter

From the gateway source (`zen/util/handler.ts`):

```js
const rawIp = req.headers.get("x-real-ip") ?? ""
const ip = rawIp.includes(":") ? rawIp.split(":").slice(0,4).join(":") : rawIp
//                                    ^ IPv6 -> /64 prefix ; IPv4 -> full address

const rateLimiter = modelInfo.allowAnonymous
  ? createIpRateLimiter(modelInfo.id, modelInfo.rateLimit, ip, req)   // <- per IP
  : createKeyRateLimiter(modelInfo.id, modelInfo.rateLimit, zenApiKey, req)

const stickyId = sessionId ? sessionId : (authInfo?.workspaceID ?? ip)
```

- **Anonymous** requests → limited **per IP**.
- **Keyed** requests → limited per account key.
- IPv4 counted per exact address; IPv6 per /64.

**It's a known, messy issue.** Reports describe the same misleading 403 as network-dependent and free-quota-related:

| Issue | What it shows |
|---|---|
| #56324 / #56334 / #56373 / #56398 | 403 depends on network / exit IP; same client works elsewhere |
| #42074 | Official client works, raw curl 429s → client shape matters |
| #42765 | Shared IPs (CGNAT) exhaust quota for everyone |
| #31839 | Two machines, same IP → one drains both |
| OmniRoute #13986 | Zen enforces client identity on the free tier |

---

## 19. The Toolbox

### `~/freecheck.sh` — probe a VPN server

```bash
#!/data/data/com.termux/files/usr/bin/bash
ip=$(curl -s -4 --max-time 10 https://ifconfig.me)
echo "egress IP : $ip"
out=$(timeout 60 opencode run --model opencode/deepseek-v4.1-flash-free \
        "reply exactly OK" 2>&1)
if echo "$out" | grep -qi "free tier"; then
  echo "RESULT    : FAIL - no quota on this IP. Switch server, rerun."
else
  echo "RESULT    : OK - free tier works on this server."
fi
```

Run it **on the phone, in Termux**, after connecting a server. `OK` = keep it; `FAIL` = switch and retry.

### Everyday diagnostic commands

| Command | Purpose |
|---|---|
| `curl -4 ifconfig.me` | IPv4 egress (is the VPN really on?) |
| `curl -6 ifconfig.me` | IPv6 egress (empty = no leak, good) |
| `opencode --version` | v1 vs v2 |
| `ip -brief addr` | interfaces (tun/wg/proton0) |
| `ssh -p 8022 u0_a395@<ip>` | into Termux |
| `freecheck.sh` | does this IP work right now? |

## 20. Decision Tree — "Free Tier Fails"

```
opencode free model returns 403
        |
        +- Is curl -4 ifconfig.me DIFFERENT with VPN on?
        |        |
        |        +- NO  -> VPN not covering Termux
        |        |        (split-tunnel / proxy VPN / wrong app)
        |        +- YES -> IPv4 fine, check IPv6:
        |                 curl -6 ifconfig.me
        |                   empty   -> good
        |                   home IP -> IPv6 LEAK -> block IPv6 in VPN
        |
        +- Still failing with a fresh, non-leaking IP?
        |        |
        |        +- 403 the moment you burst -> short-window limit -> WAIT
        |        +- 403 always on this IP    -> server IP saturated
        |                                      -> SWITCH SERVER (roulette)
        |
        +- Going through a proxy (OmniRoute)?
                 -> client-identity gate -> use a DIRECT model instead
```

### One-line truths

| Fact | Value |
|---|---|
| Quota is keyed on | **public egress IP** (IPv4 exact, IPv6 /64) |
| Same IP means | shared bucket |
| Different IP means | independent bucket |
| VPN fixes | the IP gate — *not* the client gate |
| A VPN "works" only if | `curl` shows a new IP |
| Free tier is | metered, shared, self-healing — never guaranteed |

---

## 21. Takeaways & Final State

**What we changed on the phone:**

| Item | Before | After |
|---|---|---|
| OpenCode version | v1.18.22 (kept — v2 can't run on Termux) | v1.18.22 |
| Default model | `omniroute/auto/best-coding` | `opencode/deepseek-v4.1-flash-free` |
| OmniRoute | installed + service + provider | **fully removed** |
| VPN | modded proxy "VPN" | Proton, full tunnel, no IPv6 leak |
| Helper | — | `~/freecheck.sh` |

**The 6 big lessons:**

1. A message can be **misleading** — the same 403 hides several different causes.
2. **Isolate one variable** at a time (we changed client, then IP, then IPv6).
3. Two apps on one Wi-Fi share **one** quota because they share one IP.
4. A "connected" VPN may route **nothing** — always verify the IP.
5. The free tier is per-**IP**, metered, and **decays** over time.
6. Server choice = **IP choice**. Same country ≠ same result.

> **If you only remember one thing:**
> Free tier works or fails based on which IP you exit from — verify with
> `curl -4 ifconfig.me`, and switch servers until one works.

---

<sub>Notes generated from a real debugging session · interactive version: [`index.html`](./index.html)</sub>

