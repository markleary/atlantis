# Repo Level atlantis.yaml Config
An `atlantis.yaml` file specified at the root of a Terraform repo allows you
to instruct Atlantis on the structure of your repo and set custom workflows.

[[toc]]

## Do I need an atlantis.yaml file?
`atlantis.yaml` files are only required if you wish to customize some aspect of Atlantis.
The default Atlantis config works for many users without changes.

Read through the [use-cases](#use-cases) to determine if you need it.

## Enabling atlantis.yaml
By default, all repos are allowed to have an `atlantis.yaml` file,
but some of the keys are restricted by default.

Restricted keys can be set in the server-side `repos.yaml` repo config file.
You can enable `atlantis.yaml` to override restricted
keys by setting the `allowed_overrides` key there. See the [Server Side Repo Config](server-side-repo-config.html) for
more details.

**Notes**
* By default, repo root `atlantis.yaml` file is used.
* You can change this behaviour by setting [Server Side Repo Config](server-side-repo-config.html)

::: danger DANGER
Atlantis uses the `atlantis.yaml` version from the pull request, similar to other
CI/CD systems. If you're allowing users to [create custom workflows](server-side-repo-config.html#allow-repos-to-define-their-own-workflows)
then this means
anyone that can create a pull request to your repo can run arbitrary code on the
Atlantis server.

By default, this is not allowed.
:::

::: warning
Once an `atlantis.yaml` file exists in a repo, Atlantis won't try to determine
where to run plan automatically. Instead it will just follow the project configuration.
This means that you'll need to define each project in your repo.

If you have many directories with Terraform configuration, each directory will
need to be defined.
:::

## Example Using All Keys

```yaml
version: 3
automerge: true
delete_source_branch_on_merge: true
parallel_plan: true
parallel_apply: true
projects:
- name: my-project-name
  branch: /main/
  dir: .
  workspace: default
  terraform_version: v0.11.0
  delete_source_branch_on_merge: true
  repo_locking: true
  autoplan:
    when_modified: ["*.tf", "../modules/**/*.tf"]
    enabled: true
  plan_requirements: [mergeable, approved, undiverged]
  apply_requirements: [mergeable, approved, undiverged]
  import_requirements: [mergeable, approved, undiverged]
  workflow: myworkflow
workflows:
  myworkflow:
    plan:
      steps:
      - run: my-custom-command arg1 arg2
      - init
      - plan:
          extra_args: ["-lock", "false"]
      - run: my-custom-command arg1 arg2
    apply:
      steps:
      - run: echo hi
      - apply
allowed_regexp_prefixes:
- dev/
- staging/
```

## Example of DRYing up projects using YAML anchors

```yaml
projects:
  - &template
    name: template
    workflow: custom
    autoplan:
      enabled: true
      when_modified:
        - "./terraform/modules/**/*.tf"
        - "**/*.tf"

  - <<: *template
    name: ue1-prod-titan
    dir: ./terraform/titan
    workspace: ue1-prod

  - <<: *template
    name: ue1-stage-titan
    dir: ./terraform/titan
    workspace: ue1-stage

  - <<: *template
    name: ue1-dev-titan
    dir: ./terraform/titan
    workspace: ue1-dev
```

## Auto generate projects

This is useful if you have many projects in a repository. This assumes the `default` workspace (or no workspace).

Run this in the root of your repository. This will use gnu `grep` to search terraform files for an S3 backend (terraform dir), retrieve the directory path, retrieve the unique entries, and then use `yq` to return the YAML of a simple project dir setup which can then be modified to your liking.

```sh
grep -P 'backend[\s]+"s3"' **/*.tf |
  rev | cut -d'/' -f2- | rev |
  sort |
  uniq |
  while read d; do \
    echo '[ {"name": "'"$d"'","dir": "'"$d"'", "autoplan": {"when_modified": ["**/*.tf.*"] }} ]' | yq -PM; \
  done
```

## Use Cases
### Disabling Autoplanning
```yaml
version: 3
projects:
- dir: project1
  autoplan:
    enabled: false
```
This will stop Atlantis automatically running plan when `project1/` is updated
in a pull request.

### Run plans and applies in parallel

```yaml
version: 3
parallel_plan: true
parallel_apply: true
```

This will run plans and applies for all of your projects in parallel.

Enabling these options can significantly reduce the duration of plans and applies, especially for repositories with many projects.

Use the `--parallel-pool-size` to configure the max number of plans and applies that can run in parallel. The default is 15.

Parallel plans and applies work across both multiple directories and multiple workspaces.

### Configuring Planning

Given the directory structure:

```
.
├── modules
│   └── module1
│       ├── main.tf
│       ├── outputs.tf
│       └── submodule
│           ├── main.tf
│           └── outputs.tf
└── project1
    └── main.tf
```

If you want Atlantis to plan `project1/` whenever any `.tf` files under `module1/` change or any `.tf` or `.tfvars` files under `project1/` change you could use the following configuration:


```yaml
version: 3
projects:
- dir: project1
  autoplan:
    when_modified: ["../modules/**/*.tf", "*.tf*"]
```

Note:
* `when_modified` uses the [`.dockerignore` syntax](https://docs.docker.com/engine/reference/builder/#dockerignore-file)
* The paths are relative to the project's directory.
* `when_modified` will be used by both automatic and manually run plans.
* `when_modified` will continue to work for manually run plans even when autoplan is disabled.

### Supporting Terraform Workspaces
```yaml
version: 3
projects:
- dir: project1
  workspace: staging
- dir: project1
  workspace: production
```
With the above config, when Atlantis determines that the configuration for the `project1` dir has changed,
it will run plan for both the `staging` and `production` workspaces.

If you want to `plan` or `apply` for a specific workspace you can use
```
atlantis plan -w staging -d project1
```
and
```
atlantis apply -w staging -d project1
```

### Using .tfvars files
See [Custom Workflow Use Cases: Using .tfvars files](custom-workflows.html#tfvars-files)

### Adding extra arguments to Terraform commands
See [Custom Workflow Use Cases: Adding extra arguments to Terraform commands](custom-workflows.html#adding-extra-arguments-to-terraform-commands)

### Custom init/plan/apply Commands
See [Custom Workflow Use Cases: Custom init/plan/apply Commands](custom-workflows.html#custom-init-plan-apply-commands)

### Terragrunt
See [Custom Workflow Use Cases: Terragrunt](custom-workflows.html#terragrunt)

### Running custom commands
See [Custom Workflow Use Cases: Running custom commands](custom-workflows.html#running-custom-commands)

### Terraform Versions
If you'd like to use a different version of Terraform than what is in Atlantis'
`PATH` or is set by the `--default-tf-version` flag, then set the `terraform_version` key:

```yaml
version: 3
projects:
- dir: project1
  terraform_version: 0.10.0
```

Atlantis will automatically download and use this version.

### Requiring Approvals For Production
In this example, we only want to require `apply` approvals for the `production` directory.
```yaml
version: 3
projects:
- dir: staging
- dir: production
  plan_requirements: [approved]
  apply_requirements: [approved]
  import_requirements: [approved]
```
:::warning
`plan_requirements`, `apply_requirements` and `import_requirements` are restricted keys so this repo will need to be configured
to be allowed to set this key. See [Server-Side Repo Config Use Cases](server-side-repo-config.html#repos-can-set-their-own-apply-an-applicable-subcommand).
:::

### Order of planning/applying
```yaml
version: 3
projects:
- dir: project1
  execution_order_group: 2
- dir: project2
  execution_order_group: 1
```
With this config above, Atlantis runs planning/applying for project2 first, then for project1.
Several projects can have same `execution_order_group`. Any order in one group isn't guaranteed.
`parallel_plan` and `parallel_apply` respect these order groups, so parallel planning/applying works
in each group one by one.

### Custom Backend Config
See [Custom Workflow Use Cases: Custom Backend Config](custom-workflows.html#custom-backend-config)

## Reference
### Top-Level Keys
```yaml
version: 3
automerge: false
delete_source_branch_on_merge: false
projects:
workflows:
allowed_regexp_prefixes:
```
| Key                           | Type                                                     | Default | Required | Description                                                                                                                          |
|-------------------------------|----------------------------------------------------------|---------|----------|--------------------------------------------------------------------------------------------------------------------------------------|
| version                       | int                                                      | none    | **yes**  | This key is required and must be set to `3`.                                                                                         |
| automerge                     | bool                                                     | `false` | no       | Automatically merges pull request when all plans are applied.                                                                        |
| delete_source_branch_on_merge | bool                                                     | `false` | no       | Automatically deletes the source branch on merge.                                                                                    |
| projects                      | array[[Project](repo-level-atlantis-yaml.html#project)]  | `[]`    | no       | Lists the projects in this repo.                                                                                                     |
| workflows<br />*(restricted)* | map[string: [Workflow](custom-workflows.html#reference)] | `{}`    | no       | Custom workflows.                                                                                                                    |
| allowed_regexp_prefixes       | array[string]                                            | `[]`    | no       | Lists the allowed regexp prefixes to use when the [`--enable-regexp-cmd`](server-configuration.html#enable-regexp-cmd) flag is used. |

### Project
```yaml
name: myname
branch: /mybranch/
dir: mydir
workspace: myworkspace
execution_order_group: 0
delete_source_branch_on_merge: false
repo_locking: true
autoplan:
terraform_version: 0.11.0
plan_requirements: ["approved"]
apply_requirements: ["approved"]
import_requirements: ["approved"]
workflow: myworkflow
```

| Key                                      | Type                  | Default     | Required | Description                                                                                                                                                                                                                               |
|------------------------------------------|-----------------------|-------------|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| name                                     | string                | none        | maybe    | Required if there is more than one project with the same `dir` and `workspace`. This project name can be used with the `-p` flag.                                                                                                         |
| branch                                   | string                | none        | no       | Regex matching projects by the base branch of pull request (the branch the pull request is getting merged into). Only projects that match the PR's branch will be considered. By default, all branches are matched.                       |
| dir                                      | string                | none        | **yes**  | The directory of this project relative to the repo root. For example if the project was under `./project1` then use `project1`. Use `.` to indicate the repo root.                                                                        |
| workspace                                | string                | `"default"` | no       | The [Terraform workspace](https://developer.hashicorp.com/terraform/language/state/workspaces) for this project. Atlantis will switch to this workplace when planning/applying and will create it if it doesn't exist.                    |
| execution_order_group                    | int                   | `0`         | no       | Index of execution order group. Projects will be sort by this field before planning/applying.                                                                                                                                             |
| delete_source_branch_on_merge            | bool                  | `false`     | no       | Automatically deletes the source branch on merge.                                                                                                                                                                                         |
| repo_locking                             | bool                  | `true`      | no       | Get a repository lock in this project when plan.                                                                                                                                                                                          |
| autoplan                                 | [Autoplan](#autoplan) | none        | no       | A custom autoplan configuration. If not specified, will use the autoplan config. See [Autoplanning](autoplanning.html).                                                                                                                   |
| terraform_version                        | string                | none        | no       | A specific Terraform version to use when running commands for this project. Must be [Semver compatible](https://semver.org/), ex. `v0.11.0`, `0.12.0-beta1`.                                                                              |
| plan_requirements<br />*(restricted)*   | array[string]         | none        | no       | Requirements that must be satisfied before `atlantis plan` can be run. Currently the only supported requirements are `approved`, `mergeable`, and `undiverged`. See [Command Requirements](command-requirements.html) for more details.  |
| apply_requirements<br />*(restricted)*   | array[string]         | none        | no       | Requirements that must be satisfied before `atlantis apply` can be run. Currently the only supported requirements are `approved`, `mergeable`, and `undiverged`. See [Command Requirements](command-requirements.html) for more details.  |
| import_requirements<br />*(restricted)*  | array[string]         | none        | no       | Requirements that must be satisfied before `atlantis import` can be run. Currently the only supported requirements are `approved`, `mergeable`, and `undiverged`. See [Command Requirements](command-requirements.html) for more details. |
| workflow <br />*(restricted)*            | string                | none        | no       | A custom workflow. If not specified, Atlantis will use its default workflow.                                                                                                                                                              |

::: tip
A project represents a Terraform state. Typically, there is one state per directory and workspace however it's possible to
have multiple states in the same directory using `terraform init -backend-config=custom-config.tfvars`.
Atlantis supports this but requires the `name` key to be specified. See [Custom Backend Config](custom-workflows.html#custom-backend-config) for more details.
:::

### Autoplan
```yaml
enabled: true
when_modified: ["*.tf", "terragrunt.hcl"]
```
| Key                   | Type          | Default        | Required | Description                                                                                                                                                                                                                                                       |
|-----------------------|---------------|----------------|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| enabled               | boolean       | `true`         | no       | Whether autoplanning is enabled for this project.                                                                                                                                                                                                                 |
| when_modified         | array[string] | `["**/*.tf*"]` | no       | Uses [.dockerignore](https://docs.docker.com/engine/reference/builder/#dockerignore-file) syntax. If any modified file in the pull request matches, this project will be planned. See [Autoplanning](autoplanning.html). Paths are relative to the project's dir. |
