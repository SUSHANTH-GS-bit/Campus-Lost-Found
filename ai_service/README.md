# Optional AI matching service

This service is intentionally separate from the static website. It uses
`sentence-transformers` and the CLIP model, which bring large ML dependencies and
model weights. Do not deploy this directory as a Vercel serverless function when
the platform enforces a 500 MB function bundle limit.

Deploy this folder as a Docker/container service on a platform that supports large
images and adequate RAM. Keep `SUPABASE_SERVICE_KEY` in the server's environment
variables only; never expose it to browser code.
