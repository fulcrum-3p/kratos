# CI and container image publishing

GitHub Actions replaces the Tekton Pipelines-as-Code configuration:

- `container-image-pr.yml` builds the amd64 image for pull requests targeting
  `fulcrum_main`. It does not publish an image.
- `publish-container-images.yml` builds and pushes an image tagged with the
  commit SHA on each push to `fulcrum_main`, then copies it to the regional ECR
  registries with Skopeo.

The published image is built from `.docker/Dockerfile-build`, matching the
Tekton Skaffold `kaniko` profile. The copy job runs once per destination:

| AWS account | Region |
| --- | --- |
| `280296955917` | `us-west-2` |
| `833023401363` | `eu-west-1` |
| `833023401363` | `us-west-2` |
| `833023401363` | `us-east-1` |
| `833023401363` | `ap-southeast-2` |
| `833023401363` | `eu-central-1` |
| `833023401363` | `ap-southeast-4` |

## GitHub and AWS configuration

The workflows use GitHub-hosted `ubuntu-latest` runners. No self-hosted runner
registration or runner IAM role is required. The runner needs outbound access
to AWS STS and the listed ECR regions, plus `quay.io` to download the Skopeo
image.

Set the `AWS_ECR_ROLE_ARN` repository Actions secret for `fulcrum-3p/kratos` to
the ARN of an AWS role configured for GitHub OIDC. Keep the provider audience
set to `sts.amazonaws.com` and keep a `sub` condition in the role trust policy. To
allow any branch in any `fulcrum-3p` repository to assume the role, use this
`StringLike` condition:

```json
{
  "token.actions.githubusercontent.com:sub": "repo:fulcrum-3p/*:ref:refs/heads/*"
}
```

The `refs/heads/` portion limits the subject to branches; the trailing
wildcard accepts branch names such as `main`, `master`, or `fulcrum_main`.
This trust is intentionally organization-wide: any branch in any repository
that can request a GitHub OIDC token can use the role's ECR permissions, not
just this Kratos workflow. It does not match tag or pull-request subjects.

The role must be able to publish `kratos` in `280296955917` / `us-east-2`, pull
that image, and push it to each destination above. For the
`833023401363` repositories, the ECR repository policies must also permit that
role. The role ARN is configuration, not a long-lived AWS credential; the
workflows do not require static AWS access-key secrets.

Pull-request builds can run for forked pull requests because they use an
ephemeral GitHub-hosted runner, have no AWS OIDC permission, and do not publish
the image.
