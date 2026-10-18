# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** yes

**Which ones?** Codex

---

## 1. Key prompts

The following prompts are paraphrased for clarity:

1. Read the README and the rest of the project files, explain what the project does, and summarize what I need to do.
2. The proxy image build fails with the error `dpkg: error processing package nginx-common (--configure): end of file on stdin at conffile prompt`. What causes this, and how can I fix it?
3. The proxy reports `Connection refused` when connecting to the backend during a run of `./scripts/run-tests.sh`. What should I check to understand and fix the problem?
4. Explain every part of the backend health check and why it is necessary.
5. Modify the proxy to run as a non-root user and explain each change.

---

## 2. Where the AI got it wrong

The AI suggested using `nginx:stable-alpine` as the proxy's base image. I suspected there may be a smaller nginx image so I then told him to check the nginx docker tags and find the smallest one. It then found that `nginx:stable-alpine-slim` was a smaller alternative and switched to it. I compared the built images using `docker image ls`. In my measurements, the slim version reduced disk usage by about 71 MB and content size by about 21 MB. I used the slim version instead and ran the integration tests to verify that the proxy still worked. The original suggestion worked, but it was not the smallest suitable option for this project.

---

## 3. Anything in your submission you could not fully explain?

nothing


