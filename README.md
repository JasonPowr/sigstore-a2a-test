# rh-sigstore-a2a ambient credential test

This repository validates Agent Card signing with an ambient GitHub Actions
OIDC credential. No identity token or GitHub secret is stored in the
repository.

## What the workflow tests

The workflow grants `id-token: write`, installs `rh-sigstore-a2a`, and runs the
documentation command with concrete file paths:

```console
rh-sigstore-a2a sign agent-card.json \
  --output agent-card.signed.json \
  --use_ambient_credentials
```

The result is checked as JSON and uploaded as the `signed-agent-card` workflow
artifact for seven days.

## Publish this test repository

Create an empty GitHub repository, then run these commands from this directory.
Replace `OWNER` with your GitHub user or organization:

```console
git init
git add .
git commit -m "Add ambient credential signing test"
git branch -M main
git remote add origin https://github.com/OWNER/sigstore-a2a-test.git
git push -u origin main
```

The Agent Card must be committed because GitHub Actions checks out only files
that have been pushed to the repository.

## Run the test

1. Open the repository on GitHub.
2. Select **Actions**.
3. Select **Test ambient Agent Card signing**.
4. Select **Run workflow**, and then confirm **Run workflow**.
5. Open the completed workflow run.
6. Download **signed-agent-card** from the **Artifacts** section.

Alternatively, with the GitHub CLI installed and authenticated:

```console
gh workflow run test-ambient-signing.yml
gh run watch
```

The signing step should report:

```text
✓ Agent Card signed successfully
Signed card written to: agent-card.signed.json
```

## Why GitHub Actions is required

`--use_ambient_credentials` asks Sigstore to discover an OIDC identity from the
execution environment. GitHub exposes the token request variables only when the
workflow or job has this permission:

```yaml
permissions:
  contents: read
  id-token: write
```

Running the command in an ordinary local shell is expected to fail because the
GitHub OIDC variables are absent.

## Transparency notice

The default command uses Sigstore's public production services. The signing
event and certificate identity are recorded in the public Rekor transparency
log. The certificate identifies the GitHub workflow and repository that
performed the signing.

## Test explicit identity tokens

The `Test explicit identity-token signing` workflow requests a GitHub OIDC
token with the `sigstore` audience and passes it directly to the CLI. It tests
both documented forms:

```console
rh-sigstore-a2a sign agent-card.json \
  --identity_token "$OIDC_TOKEN" \
  --output identity-token.signed.json
```

```console
rh-sigstore-a2a sign agent-card.json \
  --identity_token "$OIDC_TOKEN" \
  --client_id sigstore \
  --output identity-token-client-id.signed.json
```

Run **Test explicit identity-token signing** from the repository's **Actions**
page. The two signed files are uploaded together as the
`explicit-token-signed-agent-cards` artifact. The workflow masks each token and
keeps it only in the signing step's process environment.

## Test SLSA provenance metadata

The `Test SLSA provenance signing` workflow signs the Agent Card twice:

- once with explicit repository, commit SHA, and workflow reference values;
- once using repository and commit metadata detected from GitHub Actions.

Both commands add `--use_ambient_credentials` because signing inside GitHub
Actions requires non-interactive OIDC authentication. The signed cards are
uploaded as the `provenance-signed-agent-cards` artifact.

The workflow summary reports the provenance fields that were actually
serialized. With `rh-sigstore-a2a 0.0.1rc8`, repository, revision, builder ID,
run ID, and start time are present. The requested workflow reference is not
serialized, so the workflow emits a warning documenting that implementation
gap without failing the signing test.

## Test signed Agent Card verification

The `Test signed Agent Card verification` workflow creates a signed fixture and
then exercises the documented normal and verbose verification commands. It
derives the expected GitHub workflow identity from `github.workflow_ref` and
uses `https://token.actions.githubusercontent.com` as the expected issuer.

The workflow also supplies a deliberately incorrect signer URI and requires
that verification return a non-zero status. The successfully verified signed
card is uploaded as the `verified-signed-agent-card` artifact.
