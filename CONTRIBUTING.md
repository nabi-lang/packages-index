# Contributing

This repository is the index of packages.nabi-lang.org, the Nabi package registry. It is not a
code project: it holds one file per published package.

## Publishing a package

Lines are written only by the registry. Publish with `nabi publish` (or trusted
publishing from GitHub Actions); the registry checks the version and
commits its line here. Pull requests that add or change lines are not accepted.
See the README for the layout and the publish flow. Names that differ only in
case or in `-` / `_` are the same package.

## Reporting a problem

Open an issue for a malformed entry, or report a malicious package privately (see
SECURITY.md).

## Sign your work

Nabi uses the [Developer Certificate of Origin 1.1](https://developercertificate.org/)
(DCO) instead of a CLA. By adding a `Signed-off-by` line to a commit you
certify that you wrote the change, or otherwise have the right to submit it
under this repository's license. `git commit -s` adds the line:

```text
Signed-off-by: Your Name <you@example.com>
```

Use your name and an email you can be reached at; the email must be the
commit's author email. The `DCO / Check sign-off` check fails a pull request with an
unsigned commit. To fix it, run `git commit --amend -s` (last commit) or
`git rebase --signoff main` (every commit), then push again. The full text
of the DCO is in the
[organization's CONTRIBUTING.md](https://github.com/nabi-lang/.github/blob/main/CONTRIBUTING.md#developer-certificate-of-origin).
