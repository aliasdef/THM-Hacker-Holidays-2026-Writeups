
# TryHackMe - Hacker Holidays 2026: Day 6 - Ponzi's Crypto Rewards

* **Category:** Web / API
* **Difficulty:** Easy
* **Vulnerability:** Race Condition (Time-of-Check to Time-of-Use — TOCTOU)
* **Target:** Resort's Crypto Rewards Portal (Whale Vault)

---

## 📋 Overview
Ponzi found the resort's wellness portal running a crypto rewards app, poolside edition. The goal is to accumulate enough points to unlock access to the exclusive **Whale Vault**, which normally requires waiting 24 hours between reward claims. By exploiting a **Race Condition** in the application's reward-claiming logic, we can bypass the time restriction and instantly farm the required points.

---

## 🔍 Phase 1: Reconnaissance & Analysis
1. **Account Registration:** Register a new guest account on the portal (no strict validation required).
2. **Dashboard Review:** 
   * Account balance tracker.
   * **"Claim"** button that grants +50 points daily.
   * **"Whale Vault"** lock, which requires a minimum balance of 150 points to unlock.
3. **The Bottleneck:** Clicking "Claim" increments the balance by 50 and triggers a 24-hour server-side cooldown timer. Waiting days for points is inefficient for a CTF scenario.
4. **The Hint:** *"Somewhere between his request and the server's clock, there's a gap wide enough to walk a whale through."* — A dead giveaway for a **Race Condition (TOCTOU)** vulnerability.

---

## ⚙️ Phase 2: Exploitation (Single-Packet Race Condition)
Because the server fails to use atomic transactions or proper database locks when checking and updating the user's cooldown status, sending rapid parallel requests allows us to bypass the 24-hour check.

### Execution via Burp Suite:
1. Intercept the HTTP POST request triggered when clicking the **"Claim"** button.
2. Send the intercepted request to **Burp Repeater**.
3. Duplicate the request tab 30 times and group them together (`Create tab group`).
4. Configure the send mode to **"Send group in parallel (single-packet attack)"**.
5. Fire the group.

This technique packages the requests into a single network window, forcing the backend server to process multiple reward claims simultaneously before the database can write the lock state.

---

## 🏆 Phase 3: Looting the Flag
1. Refresh the web dashboard in the browser.
2. Observe that the parallel processing flaw credited points for multiple simultaneous requests, pushing the balance well past the 150-point threshold.
3. The **Whale Vault** is now unlocked. Navigate to the vault section to capture the flag.

**Captured Flag:**
```text
THM{***_race_condition_whale_vault_***}
