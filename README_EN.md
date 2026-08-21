# lab-devspaces-ansible-exercise4

Lab: **generation of Execution Environments (EE)** with `ansible-builder` and execution with **ansible-navigator** from **OpenShift DevSpaces** (terminal of an Eclipse Che-style workspace).

## What is an Execution Environment?

An **Execution Environment** (EE) is a **container image** (OCI) that includes everything needed to run Ansible in a **reproducible** way: `ansible-core`, `ansible-runner`, the Python interpreter, system packages, Galaxy collections, and Python dependencies required by modules. That way you avoid the typical “it works on my machine”: the same environment is used in **ansible-navigator**, **Automation Controller**, or other compatible runtimes.

In practice, you describe the EE in a YAML file (by convention `execution-environment.yml`, although here we use names such as `ee-example.yml`). **Ansible Builder** translates that YAML into a `Containerfile`/`Dockerfile` and the build context; then **Podman** or **Docker** produce the image you will reference with `--eei` in `ansible-navigator` or on the automation platform.

Schema reference documentation: [Execution environment definition — Ansible Builder](https://docs.ansible.com/projects/builder/en/latest/definition/).

---

## Keys in the definition file (Ansible Builder 3.x)

The **`version: 3`** schema accepts **seven top-level blocks**. Each one controls a part of the build. Below, what each key does and a minimal illustrative example (you can combine them in a single file).

### `version`

Schema version of the file. For Ansible Builder 3.x it must be **`3`**; if omitted, the builder may assume the old schema (`1`).

```yaml
version: 3
```

### `images`

Defines the container **base image** (OS and preexisting packages). At minimum `base_image.name` is required (it is recommended to use the form `host/namespace/name:tag`).

| Nested key | Use |
| --- | --- |
| `base_image.name` | Image the EE starts from (required). |
| `base_image.signature_original_name` | If the image is **mirrored** in your registry but signed with the original name, put the “canonical” name here for Podman signature verification. |

```yaml
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9:latest
```

### `dependencies`

What is **installed in the final image**: Ansible, runner, collections, Python, system packages, and exclusions. Collections can bring transitive dependencies; `exclude` (from ansible-builder **3.1+**) lets you omit **first-level** requirements declared by collections.

| Key | Use |
| --- | --- |
| `ansible_core` | Dictionary with `package_pip`: **pip** specification for `ansible-core` (name, version `==`, URL, etc.). |
| `ansible_runner` | Same with `package_pip` for **ansible-runner**. |
| `python_interpreter` | `package_system` (RPM package for the interpreter) and/or `python_path` (path to the Python binary to use). |
| `galaxy` | Collections: path to a `requirements.yml`, embedded YAML, or list under `collections:` (with optional `name`/`version`). |
| `python` | pip packages: list or `requirements.txt`-style file ([PEP 508](https://peps.python.org/pep-0508/) syntax). |
| `system` | System packages in **bindep** format: list or `bindep.txt` file. |
| `exclude` | Subkeys `python`, `system`, and `all_from_collections`; regex with `~` prefix is accepted. |

```yaml
dependencies:
  ansible_core:
    package_pip: ansible-core==2.16.0
  ansible_runner:
    package_pip: ansible-runner
  galaxy:
    collections:
      - name: kubernetes.core
  python:
    - kubernetes
  system:
    - tar
```

### `build_arg_defaults`

Default values for container **build args** (equivalent to passing `--build-arg` on the CLI; the CLI **overrides** what is defined here). Ansible Builder injects some specific names into the `Containerfile`.

| Key | Use |
| --- | --- |
| `ANSIBLE_GALAXY_CLI_COLLECTION_OPTS` | Extra options for `ansible-galaxy collection install` (e.g. `--pre` for pre-releases). |
| `ANSIBLE_GALAXY_CLI_ROLE_OPTS` | Options for installing **roles** with galaxy. |
| `PKGMGR_PRESERVE_CACHE` | Control of the package manager cache during the build (`always` = do not clean as often as by default). |

```yaml
build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: '--pre'
```

### `additional_build_files`

Copies files from the repo into the **build context** under `_build/`, so you can reference them with `COPY` in custom steps.

Each element is a dictionary with:

| Key | Use |
| --- | --- |
| `src` | Source: absolute path or relative to the EE YAML; can be a **glob**. If it is a directory, all of its content is copied. |
| `dest` | Subdirectory **inside** `_build/` of the context (no absolute paths or `..`). |

```yaml
additional_build_files:
  - src: files/ansible.cfg
    dest: configs
```

### `additional_build_steps`

Instructions **inserted into the Containerfile** at different **phases** of the multi-stage build. Each phase accepts a list of strings or a multiline block. **Do not** use `USER` directives here (it can break the build); for the final user, use `options.user`.

| Key | Use |
| --- | --- |
| `prepend_base` | Before completing the **base** stage. |
| `append_base` | After the **base** stage. |
| `prepend_galaxy` | Before the **galaxy** stage (collection install). |
| `append_galaxy` | After the **galaxy** stage. |
| `prepend_builder` | Before the **builder** stage. |
| `append_builder` | After the **builder** stage. |
| `prepend_final` | Before the **final** image. |
| `append_final` | After the **final** image (typical for `RUN`, extra binaries, etc.). |

```yaml
additional_build_steps:
  prepend_galaxy:
    - COPY _build/configs/ansible.cfg /etc/ansible/ansible.cfg
  append_final:
    - RUN curl -L -o /usr/bin/oc https://example.com/oc && chmod +x /usr/bin/oc
```

### `options`

Adjustments for the **container runtime** and for the builder’s own build process.

| Key | Use |
| --- | --- |
| `container_init` | Advanced dictionary for `ENTRYPOINT`/`CMD` and the init pip package (e.g. `dumb-init`). If you override one part, **the other default values of that block are cancelled**; use it only if you know the impact. |
| `container_init.cmd` | Literal value of the container `CMD` (by default something equivalent to running `bash`). |
| `container_init.entrypoint` | Literal value of the `ENTRYPOINT`. |
| `container_init.package_pip` | pip package for entrypoint support (default `dumb-init==1.2.5`). |
| `package_manager_path` | Path to the RPM manager (`/usr/bin/dnf`, `/usr/bin/microdnf`, etc.). Minimal images often need **microdnf**. Only **RPM** managers are supported (not `apt`, `apk`, etc.). |
| `skip_ansible_check` | If `true`, do not check that the final image has Ansible and Ansible Runner installed. |
| `skip_pip_install` | If `true`, do not try to install **pip** on the base (you take care of it). |
| `relax_passwd_permissions` | If `true` (default), `/etc/passwd` permissions are relaxed for environments with a dynamic user. |
| `workdir` | Default working directory (default `/runner`). |
| `user` | Default user or UID in the final image (default `1000`). |
| `tags` | List of name:tag values for the resulting image if the build succeeds (default `ansible-execution-env:latest`). |

```yaml
options:
  package_manager_path: /usr/bin/microdnf
  workdir: /runner
  tags:
    - mi-ee-lab:latest
```

---

## Practical lab contents

This project has **two EE definitions**:


| File                      | Brief description                                                                                                                                                                       |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ee-example.yml`          | Starts from an already published image (`quay.io/adelahoz/ee_cap_aap_2.6`) and adds Python/K8s collections, `oc`/`kubectl`, and dependencies to work against OpenShift/Kubernetes.     |
| `ee-example-not-exec.yml` | Builds on `registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9` and installs AAP collections from Automation Hub (requires Red Hat authentication and a Hub token).     |


---

## Requirements in DevSpaces

- Workspace with an image that includes **Podman** (or **Kubedock** integration, depending on the environment devfile; many Ansible devfiles have `KUBEDOCK_ENABLED=true`).
- **ansible-builder** and **ansible-navigator** available in the development container (the `ghcr.io/ansible/ansible-devspaces` image usually ships them).
- To **push** the image: an account on **Quay.io** (or another registry) and replace `quay.io/xxxx/yyyy` with your repository.
- For `**ee-example-not-exec.yml`**: `podman login registry.redhat.io` with a Red Hat username/password or pull secret (without this the build will fail when downloading the base image).

Open the workspace **Terminal** (**Terminal → New Terminal** menu or the integrated terminal panel) and go to the root of the cloned repository.

```bash
cd "${PROJECTS_ROOT}/lab-devspaces-ansible-exercise4"
# If your project folder has another name, adjust the path.
```

---

## Part 1 — Review the Execution Environment files

### 1.1 `ee-example.yml` (EE “on CAP / already curated image”)

```bash
less ee-example.yml
# or: cat ee-example.yml
```

Review in particular:

- `**images.base_image**`: base image you start from.
- `**dependencies.galaxy.collections**` and `**dependencies.python**`: what will be installed in the EE layer.
- `**additional_build_steps**`: environment variables and download of the `oc` client into `/usr/bin`.

### 1.2 `ee-example-not-exec.yml` (minimal RHEL9 EE + AAP from Hub)

```bash
less ee-example-not-exec.yml
```

Review:

- `**images.base_image**`: `ee-minimal-rhel9` on `registry.redhat.io` (restricted access).
- `**prepend_galaxy**`: Galaxy and Automation Hub URLs; `**ARG ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_CERTIFIED_TOKEN**` requires passing the token at build time (see below).

---

## Part 2 — Generate the build context (`create`)

`ansible-builder` can first generate the context directory (Dockerfile and auxiliary files) so you can inspect it before building.

From the project root:

```bash
# EE based on quay.io/adelahoz/ee_cap_aap_2.6
ansible-builder create -f ee-example.yml

# EE based on ee-minimal-rhel9 (optional, if you have Red Hat credentials)
ansible-builder create -f ee-example-not-exec.yml
```

By default the context is usually created in `**context/**` (or the name shown in the command output).

### Review the generated context

```bash
ls -la context/
less context/Dockerfile
# If it exists:
ls -la context/_build/
```

This shows how **ansible-builder** translates the YAML into container layers and `RUN`/`COPY` steps.

---

## Part 3 — Image build and publication

### 3.1 Build with `ee-example.yml`

```bash
ansible-builder build -f ee-example.yml
```

When it finishes, note the image name shown in the output (or the one you defined in the definition file). To tag and push to your Quay:

```bash
# Replace with your namespace and image on Quay
podman tag localhost/<generated_local_name> quay.io/xxxx/yyyy:latest
podman login quay.io
podman push quay.io/xxxx/yyyy:latest
```

### 3.2 Build with `ee-example-not-exec.yml` (Automation Hub token)

You must pass the **token** at build time so `ansible-galaxy` can download certified collections:

```bash
ansible-builder build -f ee-example-not-exec.yml \
  --build-arg ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_CERTIFIED_TOKEN="<your_token>"
```

Then `podman tag` / `podman push` the same as above.

---

## Part 4 — ansible-navigator with the already published image

Goal: students see how **ansible-navigator** uses `**--eei`** (execution environment image), pulls or reuses the image in Podman, and runs the playbook **inside the EE**.

The reference image in this lab is the one already published:

`quay.io/adelahoz/ee_cap_aap_2.6`

### 4.1 Playbook against Fedora (SSH + key mounted in the EE container)

The `inventory` points at a `fedora_servers` host. The private key is mounted in the EE container at `/runner/artifacts/id_rsa` (read-only recommended; the `:Z` suffix helps in SELinux/Podman environments):

```bash
ANSIBLE_NAVIGATOR_EXECUTION_ENVIRONMENT_VOLUME_MOUNTS="$(pwd)/ssh_tests_connections/id_fedora_new:/runner/artifacts/id_rsa:Z" \
ansible-navigator run test-exec-fedora.yaml \
  -i inventory \
  --eei quay.io/adelahoz/ee_cap_aap_2.6 \
  --pp missing \
  --m stdout \
  --penv ANSIBLE_HOST_KEY_CHECKING=False \
  -- --private-key /runner/artifacts/id_rsa
```

Check that the IP/user in `inventory` are reachable from the DevSpaces environment (lab routes/firewall).

### 4.2 Playbook against the OpenShift API (namespaces)

Example run with the other image (no remote inventory: the play is `localhost`):

```bash
ansible-navigator run test-exec-openshift.yaml \
  --eei quay.io/adelahoz/ee_kube_exec \
  --pp missing \
  --m stdout \
  -e '{"openhift_url_api_client":"https://api.<cluster>:6443","openhift_user":"<user>","openhift_password":"<password>"}'
```

**Important:** the playbook variables are defined with the typo `**openhift_*`** (three letters in “open”). If you pass them with `-e`, the names must match the YAML exactly, or the playbook would need to be fixed (recommended: rename to `openshift_*`).

---

## Part 5 — Recommended changes to run the playbooks against a real OpenShift cluster

### `test-exec-fedora.yaml` + `inventory`

- Adjust `**ansible_host**`, `**ansible_user**`, and the **SSH key** to the cluster/lab environment.
- From DevSpaces, the Fedora host must be **reachable on the network** from the workspace pod (it is not enough that it works from your laptop).

### `test-exec-openshift.yaml` (OpenShift API)

1. **API server URL**
  Use the API URL (typically `https://api.<domain>:6443`), not the web console (`console-openshift-console...`).
2. **Variables with a typo**
  Fix `openhift_url_api_client`, `openhift_user`, `openhift_password`, `openhift_validate_certs` → consistent `openshift_*` names and update all references in tasks.
3. **Security: password on the command line**
  `oc login -u ... -p ...` leaves the password in history/processes. Better:
  - `oc login --token=<token>` with a session or ServiceAccount token, or
  - mount `**KUBECONFIG`** in the EE and use `kubernetes.core` with `kubeconfig` / context, without storing a password in the playbook.
4. `**kubernetes.core.k8s_info` and `host**`
  The `host` parameter must be the **Kubernetes API FQDN** (same host as in `oc login` to the API server). The `oc whoami --show-token` token must be valid for that cluster.
5. **Certificates**
  If the cluster uses internal certificates, `validate_certs: true` may fail until you trust the CA or use `validate_certs: false` only in the lab (not in production).
6. **In-cluster execution (optional advanced)**
  If the playbook ran **from a pod in the same OpenShift**, you could use the token mounted at `/var/run/secrets/kubernetes.io/serviceaccount` and the internal `kubernetes` service, instead of username/password.
7. **RBAC permissions**
  The identity used must have at least **list** permissions on `namespaces` (e.g. `cluster-reader` role or the lab equivalent).

---

## Notes — Reference commands (original README)

The following are the commands that already appeared in this README; they are kept here as a quick reference.

```bash
ansible-builder build -f ee-example.yml
podman push quay.io/xxxx/yyyy:latest
```

```bash
ANSIBLE_NAVIGATOR_EXECUTION_ENVIRONMENT_VOLUME_MOUNTS="$(pwd)/ssh_tests_connections/id_fedora_new:/runner/artifacts/id_rsa:Z" \
ansible-navigator run test-exec-fedora.yaml \
  -i inventory \
  --eei quay.io/adelahoz/ee_cap_aap_2.6 \
  --pp missing \
  --m stdout \
  --penv ANSIBLE_HOST_KEY_CHECKING=False \
  -- --private-key /runner/artifacts/id_rsa
```

```bash
ansible-navigator run get-namespaces.yaml \
  --eei quay.io/adelahoz/ee_cap_aap_2.6 \
  --pp missing \
  --m stdout
```

**Note:** `get-namespaces.yaml` is not in the repository; use `test-exec-openshift.yaml` or create `get-namespaces.yaml` from it (see section 4.2).
