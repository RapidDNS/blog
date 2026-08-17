+++
date = '2026-08-17T21:19:00+08:00'
draft = false
title = 'Earning Bug Bounties with AI-Driven Automated Vulnerability Hunting (Part 1)'
tags = ["Bug Bounty", "AI", "Automation", "rapiddns-cli", "CodeQL"]
+++

A while back I wrote about building a vuln-hunting pipeline with rapiddns-cli and a few other tools. The one thing missing from that post was a real, paid bounty case. So today I'm sharing one that actually paid out.

I tend to pick a single hunting method and then run it at scale for SRC bounties. Base-model AI keeps getting stronger, and Agent engineering has come a long way too — from prompt tweaking to the "harness" approach everyone's into now. I pulled one of this year's real cases and stripped it down to a workflow you can repeat with RapidDNS.

## 0x00 The workflow

Here's the method, step by step:

1. Collect info with rapiddns-cli (<https://github.com/RapidDNS/rapiddns-cli>).
2. Pull HTTP and related data with httpx.
3. Batch-scan sites and recover their sourcemap files with SourceMapX (<https://github.com/insightglacier/SourceMapX>).
4. Point the AI at the recovered JavaScript to hunt for bugs.

That's the whole idea: find sites that ship `.map` files, reverse the webpack bundle back into source, then audit that source for vulnerabilities.

## 0x01 What I actually got

I scripted the pipeline above and ran it against a target's subdomains as a test. It turned up a postMessage-based arbitrary JavaScript code execution bug, and that earned me a bounty worth around $450.

![Target bounty — $450 USD](/images/shot1_reward_en.png)

Theory covered, result covered. Now for the real process.

## 0x02 How the AI bounty hunting works

rapiddns-cli runs in export mode to dump every subdomain:

```
rapiddns-cli export start <domain>
```

When it finishes you get something like `result/xxxxxx.csv`. I pull out the domains that resolve to external IPs and probe those next.

For httpx:

```
httpx -l <subdomains.txt> -mc 200,301,302 -silent -o <domain>_http.txt
```

For sourcemap recovery, I use the SourceMapX tool I built (<https://github.com/insightglacier/SourceMapX>):

```
python SourceMapX.py -m R -d urls.txt
```

After that, the custom AI-based JS vuln-hunting agent takes over.

I built this agent from the ground up. It leans on CodeQL and checks for XSS, prototype pollution, postMessage issues, and similar classes. CodeQL's query output goes to the AI for analysis; once it spots something, it verifies the finding for real — it drives a browser and actually executes the payload to confirm. Final results get one last pass where the AI judges them against several criteria. That's the loop. Honestly, once the concept clicks, you can throw it together with vibe coding without much trouble.

Why this works: CodeQL handles JavaScript well and doesn't need a build environment. Its rules also lift coverage and keep the AI from drifting or hallucinating. Beyond finding bugs, the AI writes new CodeQL rules on the fly based on the code it sees. We have hard rules and requirements to keep false positives low, and part of that verification is dynamic, which makes the results more credible. Doing it with AI plus scripts also burns far fewer tokens, so you can scan in bulk for almost nothing. It's practical, not theoretical. The case I just described cost under $15 to run, on the GPT-5.4 model. Here's a peek at how the findings get reported out.

> **Finding:** hardcoded token; postMessage source not validated, so attacker input reaches `eval` and executes JS.

![FuzzEvolve finding report](/images/shot2_finding_en.png)

I'm skipping most of the implementation details and the gotchas — there are a lot, and I'd rather not sprawl. Maybe a dedicated post later. If you're interested, or you'd want an open-source release to try, drop a comment. If enough people ask, I'll clean it up and open it. Questions welcome too.

## 0x03 RapidDNS Pro

One thing to note: using rapiddns-cli the way I showed needs a RapidDNS Pro (or higher) membership. For pricing and how to sign up, just open <https://rapiddns.io/pricing>.
