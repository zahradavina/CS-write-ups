<H1>Hacker Holiday 2026 - Day 08 by TryHackMe</H1>
<p>Greetings People! Today we are going to take a challenge from TryHackMe which is Hacker Holiday Day 08 - Towel on the sun bed.
It is an annual challenge provided practices and has several level! For the Day 08 it is medium level, and this is my write ups
about this challenge. 

<H2>Challenge Description</H2>
<p>The challenge presents a cryptocurrency portfolio application called Ponzi Portfolio. According to the briefing,</p>
<img width="584" height="596" alt="Screen Shot 2026-08-10 at 17 13 02" src="https://github.com/user-attachments/assets/b517b57a-9c68-49c0-b7d7-6d18166d57bb" />
<p>The hint strongly suggests a race condition vulnerability in the staking reward mechanism. A race condition occurs when multiple requests are processed simultaneously before the application updates its state, allowing an action intended to happen once to be executed multiple times</p>

<h2>Enumeration</h2>
<p>We surf into Http://$MACHINE_IP:3000 And then we trying to register a new account and logging in. It should shown like below,</p>

<p>Then we scrolls down and it contains: </p>
<img width="1101" height="630" alt="Screen Shot 2026-08-10 at 17 15 38" src="https://github.com/user-attachments/assets/f0a6767a-cb8d-41a0-a2d3-0707bda47ddf" />



