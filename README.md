# lab-devspaces-ansible-exercise4

Laboratorio: **generación de Execution Environments (EE)** con `ansible-builder` y ejecución con **ansible-navigator** desde **OpenShift DevSpaces** (terminal del workspace tipo Eclipse Che).

## ¿Qué es un Execution Environment?

Un **Execution Environment** (EE) es una **imagen de contenedor** (OCI) que incluye todo lo necesario para ejecutar Ansible de forma **reproducible**: `ansible-core`, `ansible-runner`, el intérprete Python, paquetes de sistema, colecciones de Galaxy y dependencias Python que exigen los módulos. Así evitas el típico “en mi máquina funciona”: el mismo entorno se usa en **ansible-navigator**, **Automation Controller** u otros runtimes compatibles.

En la práctica, describes el EE en un fichero YAML (por convención `execution-environment.yml`, aunque aquí usamos nombres como `ee-example.yml`). **Ansible Builder** traduce ese YAML a un `Containerfile`/`Dockerfile` y al contexto de build; luego **Podman** o **Docker** generan la imagen que referenciarás con `--eei` en `ansible-navigator` o en la plataforma de automatización.

Documentación de referencia del esquema: [Execution environment definition — Ansible Builder](https://docs.ansible.com/projects/builder/en/latest/definition/).

---

## Claves del fichero de definición (Ansible Builder 3.x)

El esquema **`version: 3`** admite **siete bloques de primer nivel**. Cada uno controla una parte del build. Debajo, qué hace cada clave y un ejemplo mínimo ilustrativo (puedes combinarlos en un solo fichero).

### `version`

Versión del esquema del fichero. Para Ansible Builder 3.x debe ser **`3`**; si no se indica, el builder puede asumir el esquema antiguo (`1`).

```yaml
version: 3
```

### `images`

Define la **imagen base** del contenedor (SO y paquetes previos). Como mínimo hace falta `base_image.name` (recomendable usar la forma `host/espacio/nombre:tag`).

| Clave anidada | Uso |
| --- | --- |
| `base_image.name` | Imagen desde la que parte el EE (obligatoria). |
| `base_image.signature_original_name` | Si la imagen está **espejada** en tu registro pero firmada con el nombre original, indica aquí el nombre “canónico” para la verificación de firma con Podman. |

```yaml
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9:latest
```

### `dependencies`

Qué se **instala en la imagen final**: Ansible, runner, colecciones, Python, sistema y exclusiones. Las colecciones pueden traer dependencias transitivas; `exclude` (desde ansible-builder **3.1+**) permite omitir requisitos de **primer nivel** declarados por colecciones.

| Clave | Uso |
| --- | --- |
| `ansible_core` | Diccionario con `package_pip`: especificación para **pip** de `ansible-core` (nombre, versión `==`, URL, etc.). |
| `ansible_runner` | Igual con `package_pip` para **ansible-runner**. |
| `python_interpreter` | `package_system` (paquete RPM del intérprete) y/o `python_path` (ruta del binario Python a usar). |
| `galaxy` | Colecciones: ruta a un `requirements.yml`, YAML incrustado, o lista bajo `collections:` (con `name`/`version` opcional). |
| `python` | Paquetes pip: lista o fichero tipo `requirements.txt` (sintaxis [PEP 508](https://peps.python.org/pep-0508/)). |
| `system` | Paquetes de sistema en formato **bindep**: lista o fichero `bindep.txt`. |
| `exclude` | Subclaves `python`, `system` y `all_from_collections`; admite regex con prefijo `~`. |

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

Valores por defecto de **build args** del contenedor (equivalente a pasar `--build-arg` en la CLI; la CLI **pisa** lo definido aquí). Ansible Builder inyecta algunos nombres concretos en el `Containerfile`.

| Clave | Uso |
| --- | --- |
| `ANSIBLE_GALAXY_CLI_COLLECTION_OPTS` | Opciones extra para `ansible-galaxy collection install` (p. ej. `--pre` para pre-releases). |
| `ANSIBLE_GALAXY_CLI_ROLE_OPTS` | Opciones para la instalación de **roles** con galaxy. |
| `PKGMGR_PRESERVE_CACHE` | Control de la caché del gestor de paquetes durante el build (`always` = no limpiar con la misma frecuencia por defecto). |

```yaml
build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: '--pre'
```

### `additional_build_files`

Copia ficheros del repo al **contexto de build** bajo `_build/`, para poder referenciarlos con `COPY` en pasos personalizados.

Cada elemento es un diccionario con:

| Clave | Uso |
| --- | --- |
| `src` | Origen: ruta absoluta o relativa al YAML del EE; puede ser un **glob**. Si es directorio, se copia todo su contenido. |
| `dest` | Subdirectorio **dentro** de `_build/` del contexto (sin rutas absolutas ni `..`). |

```yaml
additional_build_files:
  - src: files/ansible.cfg
    dest: configs
```

### `additional_build_steps`

Instrucciones **insertadas en el Containerfile** en distintas **fases** del build multi-stage. Cada fase admite una lista de cadenas o un bloque multilínea. **No** uses directivas `USER` aquí (puede romper el build); para el usuario final, usa `options.user`.

| Clave | Uso |
| --- | --- |
| `prepend_base` | Antes de completar la etapa **base**. |
| `append_base` | Después de la etapa **base**. |
| `prepend_galaxy` | Antes de la etapa **galaxy** (instalación de colecciones). |
| `append_galaxy` | Después de la etapa **galaxy**. |
| `prepend_builder` | Antes de la etapa **builder**. |
| `append_builder` | Después de la etapa **builder**. |
| `prepend_final` | Antes de la imagen **final**. |
| `append_final` | Después de la imagen **final** (típico para `RUN`, binarios extra, etc.). |

```yaml
additional_build_steps:
  prepend_galaxy:
    - COPY _build/configs/ansible.cfg /etc/ansible/ansible.cfg
  append_final:
    - RUN curl -L -o /usr/bin/oc https://example.com/oc && chmod +x /usr/bin/oc
```

### `options`

Ajustes del **runtime del contenedor** y del propio proceso de build en el builder.

| Clave | Uso |
| --- | --- |
| `container_init` | Diccionario avanzado para `ENTRYPOINT`/`CMD` y paquete pip del init (p. ej. `dumb-init`). Si sobrescribes una parte, **se anulan los demás valores por defecto** de ese bloque; úsalo solo si sabes el impacto. |
| `container_init.cmd` | Valor literal del `CMD` del contenedor (por defecto algo equivalente a ejecutar `bash`). |
| `container_init.entrypoint` | Valor literal del `ENTRYPOINT`. |
| `container_init.package_pip` | Paquete pip para el soporte del entrypoint (por defecto `dumb-init==1.2.5`). |
| `package_manager_path` | Ruta al gestor RPM (`/usr/bin/dnf`, `/usr/bin/microdnf`, etc.). Imágenes minimal suelen necesitar **microdnf**. Solo se soportan gestores **RPM** (no `apt`, `apk`, etc.). |
| `skip_ansible_check` | Si es `true`, no se comprueba que la imagen final tenga Ansible y Ansible Runner instalados. |
| `skip_pip_install` | Si es `true`, no se intenta instalar **pip** en la base (tú te encargas). |
| `relax_passwd_permissions` | Si es `true` (por defecto), se relajan permisos de `/etc/passwd` para entornos con usuario dinámico. |
| `workdir` | Directorio de trabajo por defecto (por defecto `/runner`). |
| `user` | Usuario o UID por defecto en la imagen final (por defecto `1000`). |
| `tags` | Lista de nombres:etiqueta para la imagen resultante si el build termina bien (por defecto `ansible-execution-env:latest`). |

```yaml
options:
  package_manager_path: /usr/bin/microdnf
  workdir: /runner
  tags:
    - mi-ee-lab:latest
```

---

## Contenido práctico del laboratorio

En este proyecto hay **dos definiciones de EE**:


| Fichero                   | Descripción breve                                                                                                                                                                       |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ee-example.yml`          | Parte de una imagen ya publicada (`quay.io/adelahoz/ee_cap_aap_2.6`) y añade colecciones Python/K8s, `oc`/`kubectl` y dependencias para trabajar contra OpenShift/Kubernetes.           |
| `ee-example-not-exec.yml` | Construye sobre `registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9` e instala colecciones de AAP desde Automation Hub (requiere autenticación Red Hat y token de Hub). |


---

## Requisitos en DevSpaces

- Workspace con imagen que incluya **Podman** (o integración **Kubedock**, según el devfile del entorno; en muchos devfiles de Ansible aparece `KUBEDOCK_ENABLED=true`).
- **ansible-builder** y **ansible-navigator** disponibles en el contenedor de desarrollo (la imagen `ghcr.io/ansible/ansible-devspaces` suele traerlos).
- Para **subir** la imagen: cuenta en **Quay.io** (u otro registro) y sustituir `quay.io/xxxx/yyyy` por tu repositorio.
- Para `**ee-example-not-exec.yml`**: `podman login registry.redhat.io` con usuario/contraseña de Red Hat o pull secret (sin esto el build fallará al descargar la imagen base).

Abre la **Terminal** del workspace (menú **Terminal → New Terminal** o el panel de terminal integrado) y sitúate en la raíz del repositorio clonado.

```bash
cd "${PROJECTS_ROOT}/lab-devspaces-ansible-exercise4"
# Si tu carpeta del proyecto tiene otro nombre, ajusta la ruta.
```

---

## Parte 1 — Revisar los ficheros del Execution Environment

### 1.1 `ee-example.yml` (EE “sobre CAP / imagen ya curada”)

```bash
less ee-example.yml
# o: cat ee-example.yml
```

Revisa en especial:

- `**images.base_image**`: imagen base desde la que partes.
- `**dependencies.galaxy.collections**` y `**dependencies.python**`: lo que se instalará en la capa del EE.
- `**additional_build_steps**`: variables de entorno y descarga de cliente `oc` en `/usr/bin`.

### 1.2 `ee-example-not-exec.yml` (EE minimal RHEL9 + AAP desde Hub)

```bash
less ee-example-not-exec.yml
```

Revisa:

- `**images.base_image**`: `ee-minimal-rhel9` en `registry.redhat.io` (acceso restringido).
- `**prepend_galaxy**`: URLs de Galaxy y Automation Hub; el `**ARG ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_CERTIFIED_TOKEN**` exige pasar el token en el build (ver más abajo).

---

## Parte 2 — Generar el contexto de build (`create`)

`ansible-builder` puede generar primero el directorio de contexto (Dockerfile y ficheros auxiliares) para inspeccionarlo antes de construir.

Desde la raíz del proyecto:

```bash
# EE basado en quay.io/adelahoz/ee_cap_aap_2.6
ansible-builder create -f ee-example.yml

# EE basado en ee-minimal-rhel9 (opcional, si tienes credenciales Red Hat)
ansible-builder create -f ee-example-not-exec.yml
```

Por defecto suele crearse el contexto en `**context/**` (o el nombre que indique la salida del comando).

### Revisar el contexto generado

```bash
ls -la context/
less context/Dockerfile
# Si existe:
ls -la context/_build/
```

Así se ve cómo **ansible-builder** traduce el YAML a capas de contenedor y pasos `RUN`/`COPY`.

---

## Parte 3 — Build de la imagen y publicación

### 3.1 Build con `ee-example.yml`

```bash
ansible-builder build -f ee-example.yml
```

Al finalizar, anota el nombre de imagen que muestre la salida (o el que hayas definido en el fichero de definición). Para etiquetar y subir a tu Quay:

```bash
# Sustituye por tu namespace e imagen en Quay
podman tag localhost/<nombre_local_generado> quay.io/xxxx/yyyy:latest
podman login quay.io
podman push quay.io/xxxx/yyyy:latest
```

### 3.2 Build con `ee-example-not-exec.yml` (token de Automation Hub)

Hay que pasar el **token** en tiempo de build para que `ansible-galaxy` pueda descargar colecciones certificadas:

```bash
ansible-builder build -f ee-example-not-exec.yml \
  --build-arg ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_CERTIFIED_TOKEN="<tu_token>"
```

Luego `podman tag` / `podman push` igual que arriba.

---

## Parte 4 — ansible-navigator con la imagen ya publicada

Objetivo: que los alumnos vean cómo **ansible-navigator** usa `**--eei`** (execution environment image), descarga o reutiliza la imagen en Podman y ejecuta el playbook **dentro del EE**.

La imagen de referencia en este laboratorio es la que ya está publicada:

`quay.io/adelahoz/ee_cap_aap_2.6`

### 4.1 Playbook contra Fedora (SSH + clave montada en el contenedor del EE)

El inventario `inventory` apunta a un host `fedora_servers`. La clave privada se monta en el contenedor del EE en `/runner/artifacts/id_rsa` (solo lectura recomendable; el sufijo `:Z` ayuda en entornos SELinux/Podman):

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

Comprueba que la IP/usuario en `inventory` sean alcanzables desde el entorno DevSpaces (rutas/firewall del laboratorio).

### 4.2 Playbook contra la API de OpenShift (namespaces)

En el repositorio el playbook equivalente a un “get namespaces” es `**test-exec-openshift.yaml**`. El README histórico mencionaba `get-namespaces.yaml`; si quieres ese nombre, puedes copiarlo:

```bash
cp test-exec-openshift.yaml get-namespaces.yaml
```

Ejemplo de ejecución con la misma imagen (sin inventario remoto: el play es `localhost`):

```bash
ansible-navigator run test-exec-openshift.yaml \
  --eei quay.io/adelahoz/ee_cap_aap_2.6 \
  --pp missing \
  --m stdout \
  -e '{"openhift_url_api_client":"https://api.<cluster>:6443","openhift_user":"<usuario>","openhift_password":"<contraseña>"}'
```

**Importante:** las variables del playbook están definidas con el typo `**openhift_*`** (tres letras en “open”). Si las pasas con `-e`, los nombres deben coincidir exactamente con el YAML o habría que corregir el playbook (recomendado: renombrar a `openshift_*`).

---

## Parte 5 — Cambios recomendados para ejecutar los playbooks contra un cluster OpenShift real

### `test-exec-fedora.yaml` + `inventory`

- Ajusta `**ansible_host**`, `**ansible_user**` y la **clave SSH** al entorno del cluster/laboratorio.
- Desde DevSpaces, el host Fedora debe ser **accesible por red** desde el pod del workspace (no basta con que funcione desde tu portátil).

### `test-exec-openshift.yaml` (API de OpenShift)

1. **URL del API server**
  Usa la URL del API (típicamente `https://api.<dominio>:6443`), no la consola web (`console-openshift-console...`).
2. **Variables con typo**
  Corrige `openhift_url_api_client`, `openhift_user`, `openhift_password`, `openhift_validate_certs` → nombres consistentes `openshift_*` y actualiza todas las referencias en tareas.
3. **Seguridad: contraseña en línea de comandos**
  `oc login -u ... -p ...` deja la contraseña en historial/procesos. Mejor:
  - `oc login --token=<token>` con token de sesión o de ServiceAccount, o
  - montar `**KUBECONFIG`** en el EE y usar `kubernetes.core` con `kubeconfig` / contexto, sin guardar contraseña en el playbook.
4. `**kubernetes.core.k8s_info` y `host**`
  El parámetro `host` debe ser el **FQDN del API Kubernetes** (mismo host que en `oc login` al servidor API). El token de `oc whoami --show-token` debe ser válido para ese cluster.
5. **Certificados**
  Si el cluster usa certificados internos, `validate_certs: true` puede fallar hasta confiar en el CA o usar `validate_certs: false` solo en laboratorio (no en producción).
6. **Ejecución dentro del cluster (opcional avanzado)**
  Si el playbook se ejecutara **desde un pod en el mismo OpenShift**, podrías usar el token montado en `/var/run/secrets/kubernetes.io/serviceaccount` y el `kubernetes` service interno, en lugar de usuario/contraseña.
7. **Permisos RBAC**
  La identidad usada debe tener al menos permisos de **list** sobre `namespaces` (p. ej. rol `cluster-reader` o equivalente del laboratorio).

---

## Notas — Comandos de referencia (originales del README)

Los siguientes son los comandos que ya figuraban en este README; se mantienen aquí como referencia rápida.

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

**Nota:** `get-namespaces.yaml` no está en el repositorio; usa `test-exec-openshift.yaml` o crea `get-namespaces.yaml` a partir de él (ver sección 4.2).