---
sidebar_position: 3
---

# Terraform State

Terraform stores state about your managed infrastructure and configuration. This state is used by Terraform to map real world resources to your configuration, keep track of metadata, and to improve performance for large infrastructures. For a detailed understanding of terraform state, please visit [the official docs](https://www.terraform.io/language/state).

### Where is the state?

By default the terraform state for all [Configurations](docs/terranetes-controller/reference/configurations.terraform.appvia.io.md) is stored in [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/) located in the controller namespace. The following template is used as the backend.

* Namespace is always the controller namespace.
* Suffix is the [Configuration](docs/terranetes-controller/reference/configurations.terraform.appvia.io.md) UUID.
* Note the [kubernetes backend](https://www.terraform.io/language/settings/backends/kubernetes) adds a prefix `tfstate-` so the state secrets will be named `tfstate-UUID`.

```go
var backendTF = `
terraform {
  backend "kubernetes" {
    in_cluster_config = true
    namespace         = "{{ .controller.namespace }}"
    secret_suffix     = "{{ .controller.suffix }}"
  }
}
```

### How to change state backend?

:::tip
Note the ability to override the backend is only available with version >= v0.3.1
:::

While using Kubernetes as a backend has it's benefits in terms of ease of use, there's a few downsides as well.

* You need to ensure backups are performed on the state secrets.
* Its harder operate or manipulate the terraform state, using taints for example.
* The terraform state is not versioned so rollbacks are harder to performed.
* You are unable to reference the state using [remote_state_data](https://www.terraform.io/language/state/remote-state-data) resource.
* The terraform state is ultimately bound to the Cluster.

Platform administrators can change the backend using the following steps.

#### Create a template for the backend to use

1. Create a kubernetes secret in the controller namespace containing the template

```bash
cat <<EOF > backend.tf
terraform {
  backend "s3" {
    bucket     = "terranetes-controller-state"
    key        = "cluster_one/{{ .namespace }}/{{ .name }}"
    region     = "eu-west-2"
    access_key = "AWS_ACCESS_KEY_ID"
    secret_key = "AWS_SECRET_ACCESS_KEY"
  }
}
EOF
```

:::tip
We inject the entire [Configuration](docs/terranetes-controller/reference/configurations.terraform.appvia.io.md) resource into the context on the template, so you can reference anything side via `configuration.PATH`
:::

Create a kubernetes secret from the above file

```shell
kubectl -n terraform-system create secret generic backend-s3 --from-file=backend.tf=backend.tf
```

2. Update the controller to use the backend template

If you are using the helm chart you simply have to update

```yaml
controller:
  # Name of a secret in the controller namespace which contains the template to use
  # for the backend state
  backendTemplate: backend-s3
```

If you are deploying the controller yourself, update the `--backend-template=backend-s3` command line flag.
