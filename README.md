# Custom Execution Environment Supported (AAP 2.6 / RHEL 9)

Repositorio para la construcción, validación, publicación e integración de un **Execution Environment (EE)** empresarial basado en la imagen oficial soportada por Red Hat para **Ansible Automation Platform (AAP) 2.6** sobre **Red Hat Enterprise Linux 9**.

Este EE está preparado para entornos híbridos y multicloud, integrando de serie soporte para automatización de infraestructura en **AWS, Azure, VMware, Windows (WinRM/Kerberos), NetApp ONTAP e Infoblox**.

---

## 📋 Especificaciones Técnicas

| Componente | Versión / Detalle |
| :--- | :--- |
| **Plataforma Objetivo** | Red Hat Ansible Automation Platform (AAP) 2.6 |
| **Imagen Base** | `registry.redhat.io/ansible-automation-platform-26/ee-supported-rhel9:latest` |
| **Ansible Core** | `ansible-core 2.16.x` (Soportado oficialmente por Red Hat) |
| **Python Runtime** | `Python 3.12` |
| **Herramienta de Build** | `ansible-builder` v3 (`version: 3`) |
| **Motor de Contenedores** | Podman |

---

## 📦 Contenido y Dependencias

### 1. Colecciones de Ansible (requirements.yml)
- **Infraestructura AAP**: infra.aap_configuration, infra.ee_utilities, infra.aap_utilities
- **Cloud & Virtualización**: amazon.aws, azure.azcollection, community.vmware
- **Sistemas Operativos**: community.windows, community.general, community.crypto
- **Networking & Almacenamiento**: netapp.ontap, infoblox.nios_modules

### 2. Dependencias Python (requirements.txt)
- **Windows / Autenticación**: pywinrm[kerberos,credssp], requests-credssp, requests-ntlm, requests-kerberos
- **VMware**: pyVmomi, pyVim
- **AWS**: boto3, botocore
- **Azure**: azure-mgmt-compute, azure-mgmt-network, azure-mgmt-resource, azure-mgmt-storage, azure-identity
- **NetApp & Infoblox**: netapp-lib, infoblox-client
- **Core / Criptografía**: cryptography, jmespath

### 3. Paquetes del Sistema (bindep.txt)
- **Runtime**: git, rsync, krb5-libs, krb5-workstation, libffi, openssl-libs, systemd-libs
- **Compilación (Build stage)**: gcc, make, python3-devel, krb5-devel, libffi-devel, openssl-devel, openldap-devel, systemd-devel, pkgconf-pkg-config

---

## 🚀 Paso 1: Requisitos Previos

1. Añadir repositorio de AAP 2.6:
 
    subscription-manager repos --enable=ansible-automation-platform-2.6-for-rhel-9-x86_64-rpms

2. Instalar podman, ansible-builder (v3) y ansible-navigator:

    dnf install podman ansible-builder ansible-navigator -y

3. Autenticación en el registro oficial de Red Hat:

    podman login registry.redhat.io

4. Disponer de un Token de Red Hat Automation Hub (para descargar colecciones certificadas).

---

## 🔐 Paso 2: Configuración de Credenciales de Build

Por motivos de seguridad, el archivo de configuración con credenciales no está versionado en Git.

1. Copiar la plantilla de ejemplo:

    cp ansible.cfg.example ansible.cfg

2. Editar ansible.cfg e introducir el token de Automation Hub:
   
    [galaxy]
    server_list = automation_hub, release_galaxy

    [galaxy_server.automation_hub]
    url=https://console.redhat.com/api/automation-hub/content/published/
    auth_url=https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
    token=TU_OFFLINE_TOKEN_AQUI

    [galaxy_server.release_galaxy]
    url=https://galaxy.ansible.com/

Nota de aislamiento: La directiva additional_build_files inyecta ansible.cfg únicamente durante la etapa galaxy. El archivo nunca se transfiere a la imagen final.

---

## 🛠️ Paso 3: Construcción del Execution Environment

Construir la imagen del EE localmente ejecutando en la raíz del repositorio:

    ansible-builder build -t localhost/custom-ee-supported:2.6.0 --container-runtime podman -v 3

---

## 🧪 Paso 4: Validación y Smoke Testing

Antes de distribuir la imagen, validar que todas las dependencias y colecciones operen limpiamente:

1. Verificación interactiva de SDKs y librerías:
   
    podman run --rm -it localhost/custom-ee-supported:2.6.0 python3 -c '
    import winrm, boto3, pyVmomi, netapp_lib, azure.identity, cryptography
    print("VALIDACION EXITOSA - TODAS LAS LIBRERIAS CARGAN DE FORMA NATIVA")
    '

2. Ejecución del Playbook de Auditoría con ansible-navigator:
   
    ansible-navigator run test-ee.yml -m stdout --eei localhost/custom-ee-supported:2.6.0

---

## 📤 Paso 5: Publicación en el Registro de Contenedores

Subir la imagen validada a Private Automation Hub (PAH), Red Hat Quay o registro corporativo:

1. Iniciar sesión en el registro destino:
   
    podman login <hub-o-registro.tu-empresa.com>

2. Etiquetar la imagen para el registro:
   
    podman tag localhost/custom-ee-supported:2.6.0 <hub-o-registro.tu-empresa.com>/ansible/custom-ee-supported:2.6.0

3. Empujar la imagen:
   
    podman push <hub-o-registro.tu-empresa.com>/ansible/custom-ee-supported:2.6.0

---

## ⚙️ Paso 6: Integración en Ansible Automation Platform (AAP) 2.6

Para poner el nuevo EE a disposición de Automation Controller:

1. Crear la Credencial de Registro (Registry Credential):
   - En Automation Controller: Administration > Credentials > Add.
   - Name: Registry - Private Automation Hub.
   - Credential Type: Container Registry.
   - Completar Authentication URL, Username y Password / Token.

2. Registrar el Execution Environment:
   - En Automation Controller: Administration > Execution Environments > Add.
   - Name: Custom EE Supported RHEL9 2.6.
   - Image: <hub-o-registro.tu-empresa.com>/ansible/custom-ee-supported:2.6.0.
   - Pull: Always o Missing.
   - Registry Credential: la credencial creada en el paso anterior.

3. Asignar el EE a Proyectos o Job Templates:
   - Nivel Organización: Access > Organizations (EE por defecto).
   - Nivel Proyecto: Automation Execution > Projects.
   - Nivel Job Template: Automation Execution > Templates > Template > Execution Environment.

---

## 🔄 Paso 7: Mantenimiento y Actualizaciones

1. Actualizar las versiones deseadas en requirements.yml o requirements.txt.
2. Incrementar el tag de versión (ejemplo: 2.6.1).
3. Recompilar con ansible-builder y repetir la validación y publicación.
4. Actualizar la referencia del tag en Automation Controller.

---

## 📄 Estructura del Repositorio

- .gitignore: Exclusiones de Git.
- README.md: Documentación técnica y ciclo de vida.
- ansible.cfg.example: Plantilla de configuración para Automation Hub.
- bindep.txt: Paquetes RPM de sistema y cabeceras de compilación.
- execution-environment.yml: Definición de arquitectura en ansible-builder v3.
- requirements.txt: Librerías y SDKs de Python.
- requirements.yml: Colecciones de Ansible requeridas.
- test-ee.yml: Playbook de prueba y auditoría del runtime.
