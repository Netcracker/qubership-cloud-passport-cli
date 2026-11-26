# Qubership cloud-passport-cli

---

Command-line tool that discovers a Kubernetes or OpenShift cluster and generates a **Cloud Passport**: a YAML
description of the cloud and of the solution instances deployed in it. Secrets are written to a separate encrypted file.

The tool reads the namespaces that are marked with `discovery.cli.io/*` labels and annotations, collects their data,
and renders it into your template.

## Installation

---

Requires Python 3.8 or later. Use package version 2.3.4 or later.

- Install it via `pip`:

```bash
pip install qubership-cloud-passport-cli
```

- Or add it as a dependency of your Poetry project:

```bash
poetry add qubership-cloud-passport-cli
```

After installation the CLI is available as `cloud_passport_cli`:

```bash
cloud_passport_cli --help
```

## Usage

---

### 1. Prepare the cluster

The CLI reads only namespaces that are marked for your solution instance. Each one needs:

| Name                       | Kind       | Value                                                       |
|----------------------------|------------|-------------------------------------------------------------|
| `discovery.cli.io/type`    | label      | Namespace type, for example `kafka`, `postgres`, `provider` |
| `discovery.cli.io/level`   | label      | `infra` or `apps`                                           |
| `discovery.cli.io/subtype` | label      | Optional                                                    |
| `discovery.cli.io/si`      | annotation | Instance names as a list, for example `['env-1']`           |

The easiest way is `put_label` with a schema file. The format is in
[configuration/cluster_labels_schema.yaml](configuration/cluster_labels_schema.yaml):

```bash
cloud_passport_cli put_label -c ~/.kube/config -s cluster_labels_schema.yaml
```

Exactly one namespace must have type `provider`. It needs:

- A secret `discovery-cli-sa-token` (key `token`) in the provider namespace.
- A secret `project-sa-token` (key `token`) in the `default` namespace.
- Optional config maps and secrets labelled `discovery.cli.io/type=external` and `discovery.cli.io/subtype=<name>`,
  one per external service (for example `smtp`, `storage`).

### 2. Write a template

Create `template_<instance>.yaml`, for example `template_env-1.yaml`. Use:

- `{{ collector.<type>.<key> }}` for a discovered value.
- `{{ secrets('collector.<type>.<key>') }}` for a secret. The value goes to the encrypted credentials file.

An example is in [configuration/cloud_passport_template.yaml](configuration/cloud_passport_template.yaml). The available
keys are defined in [cloud_passport_cli/scrapers.py](cloud_passport_cli/scrapers.py).

### 3. Create a secret key

```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
export SECRET_KEY="<key>"
```

Keep the key. It is needed to decrypt the credentials file.

### 4. Generate the Cloud Passport

```bash
cloud_passport_cli create -c ~/.kube/config -i env-1 -s ./templates -d ./out
```

Output in `./out`:

- `new_passport_env-1.yaml`: the Cloud Passport
- `data_sensitive_env-1.yaml`: encrypted credentials

To decrypt the credentials (the values are printed to the log):

```bash
cloud_passport_cli decode_sensitive -s ./out/data_sensitive_env-1.yaml -k ./key
```

## Commands

---

| Command            | What it does                                                               |
|--------------------|----------------------------------------------------------------------------|
| `create`           | Discovers the solution instance and renders the passport and credentials   |
| `put_label`        | Adds the discovery labels and annotations to namespaces from a schema file |
| `visual_diagram`   | Draws a diagram from `cluster_schema.yaml` (needs Graphviz)                |
| `decode_sensitive` | Decrypts a credentials file and prints it to the log                       |

Run `cloud_passport_cli <command> --help` for all options. Connect with either `--config` (kubeconfig) or
`--host` and `--api-token`, not both.

## Troubleshooting

---

| Problem                                             | Fix                                                            |
|-----------------------------------------------------|----------------------------------------------------------------|
| `No namespaces with ... 'provider' found`           | Mark exactly one namespace as type `provider`                  |
| `More than one namespace with ... 'provider' found` | Remove the `provider` type from the extra namespaces           |
| A namespace is missing from the result              | Check its labels, the `si` annotation, and that it is `Active` |
| The CLI asks to generate a key file                 | Set `SECRET_KEY` or pass `--secret-key <file>`                 |
| The passport contains `Undefined` or empty values   | The key does not exist for that type. Check `scrapers.py`      |

## Development

---

```bash
poetry install
poetry run pytest
```

## License

---

Apache License 2.0, see [LICENSE](LICENSE).
