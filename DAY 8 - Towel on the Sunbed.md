
### Challenge Briefing

Ponzi found the resort's wellness portal running a little side project called _Ponzi_ — a crypto rewards app, poolside edition. He set his towel down, claimed his daily reward, and went to reapply sunscreen. He came back to find the sunbed had been "claimed" three times over while he wasn't looking.

He's convinced the app owes him a spot in the Whale Vault. The app disagrees, politely, once every 24 hours. Somewhere between his request and the server's clock, there's a gap wide enough to walk a whale through.

Lab Machine: http://MACHINE_IP:3000

---
### Finding the Clue in the Source

When I first opened the app, I was greeted with a basic login page. After signing in, I noticed a daily reward button that gives you 50 "PONZI" tokens once every 24 hours. The main goal was to unlock the **Whale Vault**, but it required a balance of **150 PONZI**. Waiting three full days didn't sound like much fun, so I popped open the browser DevTools to see how the frontend worked under the hood.

![[page.png|392]]

![[login.png|476]]

Looking through the network tab and source files, I found a script loaded at `/js/dashboard.js`.

Inside, two main things caught my attention:

1. Clicking the claim button triggers a simple request: `POST /claim`.
2. The page uses a function called `loadDashboard()` that checks `secondsUntilClaim` and disables the button if you've already claimed your daily reward.

---

### The "Aha!" Moment: Spotting the Race Condition

While reading up on web vulnerabilities, I remembered learning about **Time-of-Check to Time-of-Use (TOCTOU)** race conditions.

I wondered: _What happens if the server checks whether I'm eligible to claim, but before it can record that I claimed, another request hits the server?_

If the backend isn't locked down properly, both requests might see `canClaim = true` at the exact same instant, awarding the 50 PONZI twice!



### Testing the Exploit in the Console

To test this, I didn't even need special tools like Burp Suite—I just used the browser's JavaScript console.

I wrote a quick loop using `Promise.all()` to send a batch of 30 `POST` requests to `/claim` all at once, as fast as the browser could fire them off:

```javascript
const claims = [];
for (let i = 0; i < 30; i++) {
  claims.push(fetch('/claim', { method: 'POST' }));
}

await Promise.all(claims);
```

![[console.png|791]]

By firing them simultaneously, a bunch of those requests snuck into that tiny processing window before the backend updated the 24-hour cooldown timer.

---
### Unlocking the Vault

After running the script, I refreshed the dashboard state. My balance jumped straight past the 250 PONZI target!

With the balance requirement met, the "Claim Vault" button was unlocked. Clicking it sent a request to `/vault`, and the server responded with the flag.

![[flag (copy 1).png|579]]

---
### What I Learned

This was a awesome hands-on demonstration of why atomic operations and database locking matter. If state updates aren't handled atomically on the backend, users can trigger unintended behaviors just by sending requests faster than the application expects!