# Nomos kernel registry

This repository is Captain App's upstream-compatible deployment of
Cloudflare's Apache-2.0 serverless OCI registry. It is a separate RAN component,
not vendored source or a Git submodule of Nomos. That keeps the registry's
deployment and upstream cadence independent from kernel and host releases while
preserving Cloudflare's commit history.

## Package contract

The canonical kernel package is an OCI artifact in `nomos/kernel`:

- identity: the immutable OCI manifest digest;
- executable layer: the canonical `wasm32-wasip1` module with media type
  `application/wasm`;
- artifact type: `application/vnd.nomos.kernel.v1`;
- contents and dependencies: a CycloneDX BOM attached as an OCI 1.1 referrer;
- build origin: SLSA/in-toto provenance attached as an OCI 1.1 referrer;
- compatibility: a CUE contract attached as an OCI 1.1 referrer;
- release channels: mutable tags such as `test-mint` and `production`.

The digest identifies bytes. A tag is only a convenience pointer. RAN is the
authority that may move a channel after it has checked the candidate, evidence,
compatibility, rollout policy, and predecessor state. Rollback moves the channel
to a retained, previously admitted digest; it never rebuilds an old kernel.

All OpenUSD fixtures used as evidence must themselves be deployable packages,
not a test-only representation of one.

## Authentication

The registry is private: `workers_dev` and preview URLs are disabled, and the
custom domain does not permit anonymous pulls. No credential is committed here.

Bootstrap uses Worker secrets `USERNAME` and `PASSWORD` for a registry-only
publisher. Host applications must not embed that credential. The intended
steady state is `JWT_REGISTRY_TOKENS_PUBLIC_KEY` in the Worker, with RAN's
provider adapter issuing short-lived, repository-scoped `pull` or `push`
capabilities. Switching to JWT replaces Basic authentication; the registry does
not enable both modes simultaneously.

## Retention and garbage collection

Every digest referenced by an active channel, a supported host compatibility
range, a scheduled release, or a rollback window is retained. Garbage
collection is a separately authorised RAN operation and is not part of deploy.
CycloneDX, provenance, and compatibility referrers are retained with their
subject.

## Deploy and upstream custody

The production Worker is `nomos-kernel-registry`, backed by the R2 bucket of the
same name and served only at `registry.nomos.cafe`.

Before the first deploy, create the bucket and set authentication secrets using
Wrangler against `deploy/nomos.wrangler.jsonc`. Never pass a secret on the
command line or place it in this repository.

Run the complete local gate with:

```sh
pnpm install --frozen-lockfile
pnpm run check
```

Deploy only an exact, reviewed commit from `main`:

```sh
pnpm run deploy:nomos
```

For upstream updates, fetch `upstream/main`, merge it into a branch with a merge
commit, run the complete gate, and merge the reviewed pull request without
squashing. Captain App changes stay small and auditable; upstream history stays
intact.
