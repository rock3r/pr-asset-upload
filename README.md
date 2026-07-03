# pr-asset-upload

An [agentskills.io](https://agentskills.io)-format skill that lets a coding agent upload a
screenshot or short demo clip and get back a URL to embed or link in a GitHub PR — instead of
committing binary files to the repo under review.

See [SKILL.md](SKILL.md) for the full spec: request/response format, headers, error codes, and
usage guidance for the agent.

This repo documents the client-facing contract only. It doesn't include the server implementation
(a Cloudflare Worker + R2 bucket) — that's operated privately.
