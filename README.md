### GitHub
# Accessablity:
    - Public
    - Anyone can create MR
        - Only Admin can push or merge
    - Forking enabled
    - Only Admin can create delete tags
# Default Branch (Master)
    - Protected
    - Merge only via merge request
# Development Branches
    - Development Branch (develop)
    - Feature Branch (feature)
    - Bug Branch (bug)
    - Hotfix Branch (hotfix)
# Merge Requests
    - One Approval
    - Reset approvals when new commits are pushed
    - Prevent the author from approving their own MR(Accept for Admin(me))
    - Require a resolved discussion / pipeline success before merge
    - Merge method: merge commit vs squash (policy choice, not really security)
# Push rules (repo-wide)
    - Reject unsigned commits (if you will actually use GPG/SSH signing)
    - Reject secrets (GitLab’s built-in secret push check)
    - Reject unsigned/unverified users
    - Commit message / branch-name regex only if you will enforce them
    - Deny files over a size cap
 # CI
    - Merge request pipelines required to pass
    - Protected variables only on protected branches
    - Don’t let MRs from forks run privileged jobs unless you mean to

### Pipeline
1. Cloud platform. Pick where the software runs (AWS, GCP, Azure, or your own machines). That choice sets identity, networking, regions, and which registries and build machines are native. Design the account or project layout now: one account for builds, separate ones for staging and production, with production deploy rights narrower than build rights.

2. Version control. Put code, infrastructure, config, and pipeline definitions in one Git host (GitHub, GitLab, Bitbucket). Decide repo shape (one repo or many), who can merge, and that the default branch only accepts reviewed changes. The Git host is the trigger for everything after it.

3. CI/CD platform. Pick the system that runs the work: GitHub Actions or GitLab CI if you want it tied to the Git host, or a cloud builder (CodeBuild, Cloud Build) or a separate runner system (Buildkite, Tekton, Jenkins) if builds must stay inside your network. Run untrusted builds on ephemeral runners, and let the pipeline assume a short-lived cloud role instead of storing long-lived keys.

4. Artifact storage. Add a registry the cloud can pull from: a container registry (ECR, Artifact Registry, ACR, GHCR) and, if you ship more than images, a package or binary store. Publish each build once, tag it with the commit, and let later environments pull that same object. Block overwrites of tags you have already released.

5. Pipeline building. Only now write the pipeline, in the repo, on top of those four. A commit in version control starts the CI platform, which builds on the cloud, pushes one artifact to storage, and later deploys that artifact with a cloud role scoped to the target environment.

The usual pairing is GitHub + Actions + ECR on AWS, or GitLab + GitLab CI + Artifact Registry on GCP. Change that pairing only when a constraint forces it: builds must run in a private network, you deploy to more than one cloud, or policy forbids the Git host from holding deploy rights.
commit → build artifact → unit and integration tests → security scan → deploy to staging → smoke test → manual approve → canary or rolling deploy → watch metrics → auto-rollback
### BitFlipper
a small tool that reads bytes, flips a chosen number of bits, and writes the damaged bytes back out. Plain and encrypted input are the same job: the tool never interprets the contents, it only changes bits. Size the platform to that.

Cloud platform. None for the first release. Bitflipper runs where the verification tests run: a developer machine or a CI job. Add a cloud account only when that later test suite already lives in one.

Version control. One Git repository named bitflipper on GitHub. The repo holds the program, its tests, and the pipeline file. The default branch accepts changes only through review, so a bad flipper cannot land and then poison the verification tests.

CI/CD platform. GitHub Actions, on hosted runners. Each push builds the tool and runs it. No long-lived cloud keys, because nothing is deployed.

Artifact storage. GitHub Releases. A version tag publishes one binary per platform you care about (bitflipper for macOS and Linux is enough to start). The verification project later pins a version, so a test failure is either the verifier or a known flipper release, not a rebuild from a moving branch.

Pipeline. On every commit: build, then run tests with a fixed seed and a known input file, and check both the output bytes and a record of which bit positions changed. On a version tag: publish that same binary to Releases. The bit-position record is what makes this useful later. A verifier failure is only meaningful if you know exactly which bits were flipped.

Two rules to lock in before writing the pipeline:

Same input, seed, and flip count always produce the same output. Tests can assert exact bytes.
The tool records the offsets it changed. The later verification suite needs that list to know what it should have detected.