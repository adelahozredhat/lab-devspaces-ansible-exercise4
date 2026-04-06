# lab-devspaces-ansible-exercise4

Laboratorio: **generación de Execution Environments (EE)** con `ansible-builder` y ejecución con **ansible-navigator** desde **OpenShift DevSpaces** (terminal del workspace tipo Eclipse Che).

En este proyecto hay **dos definiciones de EE**:

| Fichero | Descripción breve |
|--------|---------------------|
| `ee-example.yml` | Parte de una imagen ya publicada (`quay.io/adelahoz/ee_cap_aap_2.6`) y añade colecciones Python/K8s, `oc`/`kubectl` y dependencias para trabajar contra OpenShift/Kubernetes. |
| `ee-example-not-exec.yml` | Construye sobre `registry.redhat.io/ansible-automation-platform-26/ee-minimal-rhel9` e instala colecciones de AAP desde Automation Hub (requiere autenticación Red Hat y token de Hub). |

---

## Requisitos en DevSpaces

- Workspace con imagen que incluya **Podman** (o integración **Kubedock**, según el devfile del entorno; en muchos devfiles de Ansible aparece `KUBEDOCK_ENABLED=true`).
- **ansible-builder** y **ansible-navigator** disponibles en el contenedor de desarrollo (la imagen `ghcr.io/ansible/ansible-devspaces` suele traerlos).
- Para **subir** la imagen: cuenta en **Quay.io** (u otro registro) y sustituir `quay.io/xxxx/yyyy` por tu repositorio.
- Para **`ee-example-not-exec.yml`**: `podman login registry.redhat.io` con usuario/contraseña de Red Hat o pull secret (sin esto el build fallará al descargar la imagen base).

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

- **`images.base_image`**: imagen base desde la que partes.
- **`dependencies.galaxy.collections`** y **`dependencies.python`**: lo que se instalará en la capa del EE.
- **`additional_build_steps`**: variables de entorno y descarga de cliente `oc` en `/usr/bin`.

### 1.2 `ee-example-not-exec.yml` (EE minimal RHEL9 + AAP desde Hub)

```bash
less ee-example-not-exec.yml
```

Revisa:

- **`images.base_image`**: `ee-minimal-rhel9` en `registry.redhat.io` (acceso restringido).
- **`prepend_galaxy`**: URLs de Galaxy y Automation Hub; el **`ARG ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_CERTIFIED_TOKEN`** exige pasar el token en el build (ver más abajo).

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

Por defecto suele crearse el contexto en **`context/`** (o el nombre que indique la salida del comando).

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

Objetivo: que los alumnos vean cómo **ansible-navigator** usa **`--eei`** (execution environment image), descarga o reutiliza la imagen en Podman y ejecuta el playbook **dentro del EE**.

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

En el repositorio el playbook equivalente a un “get namespaces” es **`test-exec-openshift.yaml`**. El README histórico mencionaba `get-namespaces.yaml`; si quieres ese nombre, puedes copiarlo:

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

**Importante:** las variables del playbook están definidas con el typo **`openhift_*`** (tres letras en “open”). Si las pasas con `-e`, los nombres deben coincidir exactamente con el YAML o habría que corregir el playbook (recomendado: renombrar a `openshift_*`).

---

## Parte 5 — Cambios recomendados para ejecutar los playbooks contra un cluster OpenShift real

### `test-exec-fedora.yaml` + `inventory`

- Ajusta **`ansible_host`**, **`ansible_user`** y la **clave SSH** al entorno del cluster/laboratorio.
- Desde DevSpaces, el host Fedora debe ser **accesible por red** desde el pod del workspace (no basta con que funcione desde tu portátil).

### `test-exec-openshift.yaml` (API de OpenShift)

1. **URL del API server**  
   Usa la URL del API (típicamente `https://api.<dominio>:6443`), no la consola web (`console-openshift-console...`).

2. **Variables con typo**  
   Corrige `openhift_url_api_client`, `openhift_user`, `openhift_password`, `openhift_validate_certs` → nombres consistentes `openshift_*` y actualiza todas las referencias en tareas.

3. **Seguridad: contraseña en línea de comandos**  
   `oc login -u ... -p ...` deja la contraseña en historial/procesos. Mejor:
   - `oc login --token=<token>` con token de sesión o de ServiceAccount, o
   - montar **`KUBECONFIG`** en el EE y usar `kubernetes.core` con `kubeconfig` / contexto, sin guardar contraseña en el playbook.

4. **`kubernetes.core.k8s_info` y `host`**  
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
