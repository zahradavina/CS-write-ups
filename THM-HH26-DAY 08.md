<H1>Hacker Holiday 2026 - Day 08 by TryHackMe</H1>
<p>Greetings People! Today we are going to take a challenge from TryHackMe which is Hacker Holiday Day 08 - Towel on the sun bed.
It is an annual challenge provided practices and has several level! For the Day 08 it is medium level, and this is my write ups
about this challenge. 

<h2>Classification</h2>
<p>Attack Vector: Concurrent Request Manipulation</p>
<p>CWE: CWE-362</p>
<P>OWASP: Business Logic Vulnerability</P>
<P>Vulnerability: Race Condition (TOCTOU)</p>
<p>Technique: Parallel Request Execution</p>
<p>Tool: Burp Suite Repeater - Send Group (Parallel)</p>
<p>Impact: Multiple reward claims within a single 24-hour period</p>

<H2>Challenge Description</H2>
<p>The challenge presents a cryptocurrency portfolio application called Ponzi Portfolio. According to the briefing,</p>
<img width="584" height="596" alt="Screen Shot 2026-08-10 at 17 13 02" src="https://github.com/user-attachments/assets/b517b57a-9c68-49c0-b7d7-6d18166d57bb" />
<p>The hint strongly suggests a race condition vulnerability in the staking reward mechanism. A race condition occurs when multiple requests are processed simultaneously before the application updates its state, allowing an action intended to happen once to be executed multiple times</p>

<h2>Enumeration</h2>
<p>We surf into Http://$MACHINE_IP:3000 And then we trying to register a new account and logging in. It should shown like below,</p>
<img width="1085" height="537" alt="Screen Shot 2026-08-10 at 17 13 24" src="https://github.com/user-attachments/assets/29d999d5-8a36-4ee2-9e05-648237d953d8" />

<p>Then we scrolls down and it contains: </p>
<img width="1101" height="630" alt="Screen Shot 2026-08-10 at 17 15 38" src="https://github.com/user-attachments/assets/f0a6767a-cb8d-41a0-a2d3-0707bda47ddf" />
<p>Basically, it says we need 150 Ponzi to open the vault. But, it shows that we earned 50 Ponzi every 24 hours which quite take some times and we don't want that..</p>

<H2>Exploitation</H2>
<p>Now we're using Burp suite to do the exploitation. First we turned on the intercept in "proxy" tab (Burp Suite>proxy>intercept),</p>
<img width="1438" height="603" alt="Screen Shot 2026-08-10 at 17 11 42" src="https://github.com/user-attachments/assets/90649a59-02e4-47e6-9625-01ef2f0b2217" />
<p>once the intercept turn on, we go back to the dashboard website and clicked the claim buttons and then we check on the Burp Suite if the REQUEST is already shown. Then we forwarding the REQUEST to the repeater by clicking the right on the mouse then forward to repeater. and then we drop the intercept thus the REQUEST hasn't reached the server. Then we click the "Repeater" tabs in Burp Suite to begin our exploitation. The REQUEST should have shown like below,</p>
<img width="1440" height="638" alt="Screen Shot 2026-08-10 at 17 12 03" src="https://github.com/user-attachments/assets/ffece90b-14cc-4474-8f98-f1e6fda0d139" />
<p>I make a groups of request and duplicating the request 6 times (6 duplicate + 1 original = 7 request) so we earned 350 Ponzi, which every request has 50 Ponzi in it therefore 7 x 50 PONZI = 350 Ponzi. Well, that exceeds the required points which that's ok.. but if you want to earned as the requirements which is 150 PONZI you can duplicate the request twice, which 2 duplicate and 1 original request equal to 3 x 50 PONZI.</p>
<img width="589" height="156" alt="Screen Shot 2026-08-11 at 14 28 15" src="https://github.com/user-attachments/assets/3dbcf3e2-85ca-4700-8c17-646eec1252cf" />
<p>I naming the group of request as "headshot" which you can naming it anything as you want.</p>
<p>The vulnerability was exploited using a Race Condition attack against the /claim endpoint. By duplicating the reward claim request in Burp Suite Repeater and sending multiple requests simultaneously via the Send Group (Parallel) feature, several requests passed the reward eligibility check before the server updated the user's claim status. This resulted in multiple staking rewards being credited within a single claim window, allowing the account balance to exceed the Whale Vault threshold.</p>

<p>Then after we exploit by using multiple request, the response should be like this:</p>
<img width="1434" height="643" alt="Screen Shot 2026-08-10 at 17 10 33" src="https://github.com/user-attachments/assets/f962a34e-3535-4268-8da3-1e540e64eb18" />
<p>then we turned off the intercept and refreshed the dashboard, It should have shown the point is already updated.</p>
<img width="1083" height="524" alt="Screen Shot 2026-08-10 at 17 13 49" src="https://github.com/user-attachments/assets/ec91d2a1-26d6-4373-8f47-3f529b45c6c3" />
<p>then we scrolls down, to check the whale vault.</p>
</p>Voila! the "open vault" buttons is clickable. by clicking the buttons we obtained the flag.</p>
<img width="1092" height="538" alt="Screen Shot 2026-08-10 at 17 14 10" src="https://github.com/user-attachments/assets/ff2d4f73-6a49-4239-92d6-848fc64a72a3" />

<h2>Mitigation</h2>
<p>Developers can prevent this issue by:</p>
<ol>
<li>Using database transactions.</li>
<li>Applying row-level locking (SELECT ... FOR UPDATE).</li>
<li>Performing atomic balance updates.</li>
<li>Implementing server-side mutexes or distributed locks.</li>
<li>Marking rewards as claimed before issuing tokens.</li>
</ol>

<h2>Conclusion</h2>
<p>This challenge demonstrated a classic Race Condition (CWE-362) vulnerability within the application's staking reward system. The /claim endpoint failed to handle concurrent requests securely, allowing multiple reward claims to be processed before the server updated the user's claim status.

By capturing the reward request in Burp Suite and leveraging the Send Group (Parallel) feature, multiple identical requests were sent simultaneously. Due to a Time-of-Check to Time-of-Use (TOCTOU) flaw, each request successfully passed the eligibility check and credited the reward before the cooldown mechanism was enforced.

As a result, the account balance increased from 0 PONZI to 350 PONZI, bypassing the intended 24-hour restriction and automatically upgrading the account from SHRIMP to WHALE tier. This allowed access to the Whale Vault, successfully completing the challenge.

This challenge highlights the importance of implementing proper synchronization, atomic transactions, and server-side locking mechanisms when handling critical business logic. Without these protections, attackers can abuse concurrent requests to bypass restrictions, gain unauthorized rewards, and compromise the integrity of an application's functionality.</p>






