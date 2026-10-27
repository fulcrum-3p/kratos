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

The workflows expect the `flcrm-gh-standard` self-hosted runner and the
`AWS_ECR_ROLE_ARN` Actions variable to be available to `fulcrum-3p/kratos`.
The runner must be registered or shared with that organization; the checked-in
runner-set configuration currently targets `fulcrumapp`.

Configure the AWS role's GitHub OIDC trust for the
`token.actions.githubusercontent.com` provider, audience `sts.amazonaws.com`,
and subject `repo:fulcrum-3p/kratos:ref:refs/heads/fulcrum_main`. The role must
be able to publish `kratos` in `280296955917` / `us-east-2`, pull that image,
and push it to each destination above. For the `833023401363` repositories,
the ECR repository policies must also permit that role. The role ARN is
configuration, not a long-lived AWS credential; the workflows do not require
static AWS access-key secrets.

Pull-request builds run only for branches in this repository. Fork pull
requests are skipped because the build uses an internal self-hosted runner;
they must not be allowed to execute untrusted code on a runner with AWS
identity.
