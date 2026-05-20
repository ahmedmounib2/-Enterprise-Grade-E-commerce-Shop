# Mitigating GHSA-pxg6-pf52-xh8x in `csurf`

npm audit flagged a low-severity advisory because `csurf@1.11.0` pins `cookie@0.4.0`, and that
release accepts out-of-bounds characters when formatting cookie names, domains, or paths. Attackers
who can control any of those fields can push carriage returns or line feeds into HTTP headers,
opening the door to response splitting, cache poisoning, or session fixation. That violates several
compliance baselines (OWASP ASVS 3.3.2, PCI DSS 6.2) and undermines assurances we offer around
session integrity and CSRF protection. We therefore patch rather than accept the risk before launch.

## Upgrade plan

Because the upstream Express team has archived `csurf` and not released a fixed build, we vendor a
minimal fork that only updates the transitive `cookie` dependency:

1. Check out the repository and install dependencies to regenerate the lockfile.
2. Use the workspace package `packages/csurf-patched` (version `1.11.0-patched`), which is a direct
   copy of `csurf@1.11.0` with the following change:
   - Runtime dependency `cookie` upgraded from `0.4.0` to `0.7.2` (first patched series) while
     retaining the original public API.
   - Development-only metadata removed so we do not bloat our install or toolchain.
3. Point the backend workspace at the patched package via
   `"csurf": "file:../packages/csurf-patched"` so Express middleware continues to load transparently
   with no UI/UX impact.
4. Run `npm install --package-lock-only --no-progress` (or full `npm install`) from the repo root to
   update `package-lock.json`.
5. Verify with `npm audit --omit=dev` that the vulnerability count drops to zero.

No runtime code paths changed, so UI behaviour remains identical. Deployment artefacts continue to
ship the familiar `csurf` middleware—only the underlying cookie serialization hardens against
malicious header injection.
