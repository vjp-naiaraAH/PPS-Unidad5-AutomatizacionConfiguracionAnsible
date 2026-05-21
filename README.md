# Actividad Unidad 5 - Automatización de la Configuración: Ansible
## PREPARANDO EL LABORATORIO
Creo una carpeta para la actividad y al haber realizado la de **kubernetes** puedo copiar la carpeta con los archivos:
```bash
# crea carpeta donde vas a realizar 
mkdir -p Actividad-Ansible
# copiar los archivos de la actividad de kubernetes
cp -rp Actividad_despliegue_app_store/* Actividad-Ansible/
# colocarse en la carpeta
cd Actividad-Ansible``` 
Además al coger los archivos de la actividad de kubernetes NO NECESITO descomprimir nada 
![carpeta actividad](/images/100.png)
```
La descarga de los archivos no necesito hacerla porque ya se usaron en la actividad de **kubernetes**, aún así adjunto captura de la ejecución del comando:
```bash
ls
```
![visualizacion de archivos](/images/101.png)

Compruebo que tengo instalado **kubect**l en el equipo con el comando:
```bash
kubectl version --client=true
```
![version kubectl](/images/102.png)

Inicio minikube con los comandos
```bash
# Iniciar un clúster de 3 nodos
minikube start --nodes 3 -p ansible
# -p ansible es el nombre del perfil (puedes elegir otro).
# Verifica los nodos creados
kubectl get nodes
```
Esto lo que ahce es crear un clúster con 
+ 1 nodo de control (***ansible***)
+ 2 nodos workers (****ansible-m02***, ***ansible-m03***)
![incicio de kubernetes y nodos](/images/103.png)

Etiqueto los nodos. Etiqueto los nodos para que los *pods* **store-app** se despliegue en modo ***ansible-m02*** y el **store-db** en el nodo ***ansible-m03***
```bash
# Etiquetar worker2 para aplicación
kubectl label nodes ansible-m02 app-role=frontend

# Etiquetar worker3 para base de datos  
kubectl label nodes ansible-m03 app-role=backend

# Verificar etiquetas
kubectl get nodes --show-labels
```
![ejecucion de nodos](/images/104.png)

Me muevo a 
```bash
cd store-app
```
Para configurar el ***docker*** de **minikube**, construir imágen y aplicar el despliegue:
```bash
# Construye imagen store-app preparado para postgree
docker build -t store-app:latest .
# cargamos la imagen en el contesto ya que sino sólo está en el nodo de control
minikube image load store-app:latest --profile=ansible
# aplica manifiesto kubernetes de despliegue
kubectl apply -f store-app-k8s.yaml  
# esperamos unos minutos que se despliegue todo
sleep 100
# obtenemos información de kubectl
kubectl get all
# consultamos la información de pods y vemos que se han creado en los nodos correspondientes 
kubectl get pods -o wide
# Ejecutamos servicio minikube para que nos muestre la dirección de la aplicación.
minikube service store-app --url -p ansible
# esperar que tarda unos minutos en arrancar todo el despliegue
# mientras tanto queda el terminal ocupado y trabajando en segundo plano
# cuando termine nos muestra la dirección del NodePort de Minikube 
```
![configuracion de docker y creacion de imagen](/images/105.png)

Compruebo que la aplicación funciona navegando a http://192.168.76.2:30583 como indica al terminar los comandos que ejecutamos anteriormente.
![firefox en la web de store-app](/images/106.png)

---

## INSTALAR ANSIBLE EN EQUIPO LOCAL
Para la instalación de **Ansible** ejecutaré los siguientes comandos:
```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt update
sudo apt install ansible
 # Verificar que se ha instalado
ansible --version  
```
Los cuales hacen:
+ **Update** a la máquina por si acaso tiene que actualizar librerias.
+ **Instala** software-properties-common y add-apt-repository además de ***Ansible***.
+ Pide la **versión** de ***Ansible*** para saber si se ha instalado correctamente.
![instalacion de ansible](/images/107.png)

Compruebo de nuevo que se haya instalado ***ansible*** ejecutando
```bash
ansible --version  
```
![version de ansible](/images/108.png)

---

## COMPROBAR Y CONFIGURAR SSH
**Minikube** crea una red interna automáticamente que nos va a permitir conectarnos automáticamente por *SSH* con los workers.
1. **Verificar el acceso**. Para verificar el acceso a los workers de **Minikube** debería tener acceso usando el comando:
```bash
# SSH a workers (¡funciona ya!)
minikube ssh --node m02 --profile=ansible
# exit para salir
```
En mi caso como se ve en la imágen a continuación puedo acceder dentro de ***m02***
![ssh a m02](/images/109.png)

2. Comprobación en el nodo ***m03*** y en el nodo de control usando:
```bash
minikube ssh --node m03 --profile=ansible
#exit  para Salir
minikube ssh  --profile=ansible
#exit  para Salir
```
Efectivamente también funciona.
![ssh a m03](/images/110.png)

3. Configurar ***SSH* sin contraseña**. Para ello establecemos una relación de confianza entre mi equipo y cada nodo. De esta forma podré conectar con **ansible** por *SSH* con cada uno de ellos
```bash
# 1. Generar clave SSH en TU HOST
ssh-keygen -t ed25519 -C "ansible-lab" -f ~/.ssh/minikube_key

# 2. Copiar clave a workers
# entrar en  Workers y crear carpeta .ssh
minikube ssh  --profile=ansible 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys' < ~/.ssh/minikube_key.pub
# pulsar Ctrl + C que se queda enganchado
minikube ssh --node m02 --profile=ansible 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys' < ~/.ssh/minikube_key.pub
# pulsar Ctrl + C que se queda enganchado
minikube ssh --node m03 --profile=ansible 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys' < ~/.ssh/minikube_key.pub
# pulsar Ctrl + C que se queda enganchado

# 3. Probar conexión al nodo de control y los workers
ssh -i ~/.ssh/minikube_key docker@$(minikube ip  --profile=ansible)
# escribir yes cuando haga pregunta de si queremos conectar y añadir a authorized.
# exit para salir
ssh -i ~/.ssh/minikube_key docker@$(minikube ip --node m02 --profile=ansible)
# escribir yes cuando haga pregunta de si queremos conectar y añadir a authorized.
# exit para salir
ssh -i ~/.ssh/minikube_key docker@$(minikube ip --node m03 --profile=ansible)
# escribir yes cuando haga pregunta de si queremos conectar y añadir a authorized.
# exit para salir
```
![configuracion sin contraseña](/images/111.png)
![configuracion sin contraseña 2](/images/112.png)

---

## CREAR INVENTARIO ANSIBLE
Creo el inventario para **ansible**. El inventario es un archivo en el cual está el nombre y la *ip* de un conjunto de *nodos* que están relacionados. Voy a crear el *inventario* de hosts llamado [crea_inventory.sh](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad5-SistemasDesplegadoAutomatizadoSoftware/ActividadAutomatizacionConfiguracionAnsible/files/crea_inventory.sh)
```shell
# Script que genera inventario por función
# IPs exactas: Almacenar las Ip de cada nodo en las variablesCONTROL_IP=$(minikube ip --profile=ansible)
CONTROL_IP=$(minikube ip --profile=ansible)
WORKER2_IP=$(minikube ip --node m02 --profile=ansible)
WORKER3_IP=$(minikube ip --node m03 --profile=ansible)

CONTROL_KEY=$(minikube ssh-key --profile=ansible)
WORKER2_KEY=$(minikube ssh-key --node m02 --profile=ansible)
WORKER3_KEY=$(minikube ssh-key --node m03 --profile=ansible)

cat > inventory.ini <<EOF
[control-plane]
ansible ansible_host=$CONTROL_IP ansible_user=docker ansible_ssh_private_key_file=$CONTROL_KEY

[app_nodes]
ansible-m02 ansible_host=$WORKER2_IP ansible_user=docker ansible_ssh_private_key_file=$WORKER2_KEY

[db_nodes]
ansible-m03 ansible_host=$WORKER3_IP ansible_user=docker ansible_ssh_private_key_file=$WORKER3_KEY

[all_workers:children]
db_nodes
app_nodes
EOF

# Probar ping
ansible all -i inventory.ini -m ping
```
![creacion de inventario de ansible](/images/113.png)

1. Una vez creado el archivo *crea_inventory.sh*, introduzco el contenido, le doy **permisos** y **ejecuto**, en mi caso al ya tener el archivo descargado y cargado en el paso anterior solo tengo que ejecutar:
```bash
# dar permisos de ejecución al script
chmod +x crea_inventory.sh
# ejecutar script
./crea_inventory.sh
```
**¡Los avisos de python 3.11 se pueden evitar ejecutando:!**
```
sudo su
cat > ansible.cfg << 'EOF'
[defaults]
interpreter_python = auto_silent
EOF
exit
```
![permisos y ejecucion](/images/114.png)

Y hago un cat al archivo *inventory.ini* para saber las direcciones *ip*.
```bash
cat inventory.ini
```
![cat inventory](/images/115.png)

2. Prueba: primer test **Ansible**
```bash
ansible-inventory -i inventory.ini --list
ansible all -i inventory.ini -m ping
```
![primer test](/images/116.png)

---

## EJECUTAR MÓDULOS ANSIBLE
A continuación voy a hacer algunas tareas con **Ansible**:
### Usuarios y grupos
Utilizo los módulos *group* y *user*
1. En el siguiente *playbook* creo un **usuario y un grupo* con nombre *maint* que realizará labores de mantenimiento en todos los nodos:
![creacion de usuario](/images/117.png)

2. Ejecuto el *playbook*:
```bash
ansible-playbook -i inventory.ini create_maint_user.yml
```
![ejecucion del plabook main_user](/images/118.png)

3. Compruebo que se han creado los usuarios. Se puede hacer con **Ansible** ya que aún no le he pasado la clave *ssh*.
```bash
ansible all_workers -i inventory.ini -m shell -a "id maint"
```
![comprobacion de creacion](/images/119.png)

### File & copy
Para crear directorios hay que:
1. Crear el siguiente *playbook*:
```bash 
nano manage_maint_files.yml
```
Y dentro:
```yml
---
  hosts: all_workers
  become: yes

  vars:
    maint_user: maint
    local_public_key_file: ./maintain_key.pub

  tasks:
    - name: Crear directorio de logs en nodos de aplicacion
      ansible.builtin.file:
        path: /opt/appmaint/logs
        state: directory
        owner: "{{ maint_user }}"
        group: "{{ maint_user }}"
        mode: '0755'
      when: "'app_nodes' in group_names"

    - name: Crear directorio de backups en nodos de base de datos
      ansible.builtin.file:
        path: /opt/dbmaint/backups
        state: directory
        owner: "{{ maint_user }}"
        group: "{{ maint_user }}"
        mode: '0750'
      when: "'db_nodes' in group_names"
```
![otro playbook](/images/120.png)


2. Ejecuto el *playbook*
```bash
 ansible-playbook -i inventory.ini manage_maint_files.yml
```
![ejecucion de playbook](/images/121.png)
3. Compruebo la ejecución:
```bash
# Nodos APP: solo /opt/appmaint
ansible app_nodes -i inventory.ini -m shell -a "ls -la /opt/appmaint/"

# Nodos DB: solo /opt/dbmaint  
ansible db_nodes -i inventory.ini -m shell -a "ls -la /opt/dbmaint/" -b
```
![comprobacion de la ejecucion](/images/122.png)

4. Descargo en la carpeta del proyecto el [archivo de la clave pública que tenemos aquí](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad5-SistemasDesplegadoAutomatizadoSoftware/ActividadAutomatizacionConfiguracionAnsible/files/maintain_key.pub)
![archivo clave publica](/images/123.png)
5. Creo el siguiente *playbook* para añadir la **clave pública** del usuario *maint* para que pueda conectarse a los nodos:
```bash
nano manage_public_key_maint.yml
```
```yml
---
- name: Anadir clave publica usuario maint
  hosts: all_workers
  become: yes

  vars:
    maint_user: maint
    local_public_key_file: ./maintain_key.pub

  tasks:
  
  - name: Crear carpeta .ssh del usuario
    ansible.builtin.file:
      path: /home/maint/.ssh
      state: directory
      owner: maint
      group: maint
      mode: '0700'

  - name: Copiar clave publica del usuario
    ansible.builtin.copy:
      src: ./maintain_key.pub
      dest: /home/maint/.ssh/maintain_key.pub
      owner: maint
      group: maint
      mode: '0644'

  - name: Autorizar la clave publica para acceso SSH
    ansible.posix.authorized_key:
      user: maint
      state: present
      key: "{{ lookup('file', './maintain_key.pub') }}"
      manage_dir: true
```
![nuevo playbook](/images/124.png)
6. Ejecutar el *playbook* con:
```bash
ansible-playbook -i inventory.ini manage_public_key_maint.yml
```
![ejecucion del nuevo](/images/125.png)
7. Copio la *clave privada* que tengo [aquí](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad5-SistemasDesplegadoAutomatizadoSoftware/ActividadAutomatizacionConfiguracionAnsible/files/private_maintain_key) en mi directorio personal /home/PPSnaiara/.ssh/. De esta forma cuando me intente conectar por *ss*h como susuario *maint* **me dejará.**
![calve privada](/images/126.png)
8. Compruebo las direcciones *IP* de los nodos y que me deja conectar como usuario *maint* en los nodos:
```bash
nano inventory.ini
# Comprueba que tienen las ip correctas
ssh -i ~/.ssh/maintain_key maint@192.168.49.3 "hostname; whoami; pwd"
ssh -i ~/.ssh/maintain_key maint@192.168.49.4 "hostname; whoami; pwd" 
```
![ip de nodos](/images/127.png)
9. Ejecutar directamente módulo file desde **Ansible**:
```bash
ansible all -i inventory.ini -m file -a "path=/home/maint/miDirectorio state=directory owner=maint group=maint mode='0700'--become"
```
![ejecucion de modulo desde ansible](/images/128.png)

10. Como ejecutar directamente módulo file:
```bash
ansible all -i inventory.ini -m copy -a "src=./inventory.ini dest=/opt/inventory.ini mode=0644 owner=root group=root" -b
```
![ejecucion de file](/images/129.png)

---

### PACKAGE
El módulo ***package** permite **administrar software**.
1. Crear el siguiente *playbook* para añadir la clave pública de usuario *maint* para que pueda conectarse a los nodos:
```bash
nano manage_public_key_maint.yml
``` 
```yml
---
- name: Añadir software mantenimiento en  todos los nodos
  hosts: all_workers              # Se ejecuta en app_nodes y db_nodes
  become: yes                    # Necesario para crear usuarios
  tasks:
    - name: Instalar paquetes de mantenimiento
      package:
        name:
          - htop
          - tree
          - curl
          - jq
          - net-tools
        state: present
    - name: borrar paquete vim
      package:
        name:
          - vim
        state: absent
```
![playbook nuevisimo](/images/130.png)
2. Ejecutar la tareas
```bash
ansible-playbook -i inventory.ini manage_anadir_paquetes_maint.yml
```
![ejecucion de playbook nuevisimo](/images/131.png)
3. Compruebo si se han instalado los paquetes:
```bash
ansible all -i inventory.ini -m shell -a "hostname ; dpkg -l | grep -E 'htop|curl|tree|jq|net-tools|vim' "  
```
![comprobacion de nuevisimo](/images/132.png)
4. Instalo directamente desde Ansible
```bash
ansible all -i inventory.ini -m apt -a "name=curl state=present" -b
```
![instalacion directa](/images/133.png)

### Command & shell & services
**Command y shell** se utilizan para ejecutar comandos en el nodo pero:
+ Usar shell siempre excepto cuando usamos pipelines.
+ Usar command cuando usamos pipelines.

1. Creo el siguiente *playbook* para instalar **nginx**, levantar el servicio y comprobar que está ejecutándose:
```bash
nano manage_servicio_nginx.yml
```
```yml


[`./manage_anadir_paquetes_maint.yml`](./files/manage_anadir_paquetes_maint.yml)
```yaml
---
- name: Instalar y validar Nginx
  hosts: all_workers
  become: true

  tasks:
    - name: Paso 1 - Instalar Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Paso 2A - Arrancar y habilitar Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes

    - name: Paso 2B - Comprobar estado con command (sin pipes)
      ansible.builtin.command:
        cmd: systemctl status nginx

    - name: Paso 2C - Comprobar estado con shell (con pipes)
      ansible.builtin.shell:
        cmd: systemctl status nginx | grep Active
      register: nginx_status
      changed_when: false

    - name: Mostrar salida de status
      ansible.builtin.debug:
        var: nginx_status.stdout

    - name: Comprobar HTTP con uri
      ansible.builtin.uri:
        url: http://localhost:80
        status_code: 200
      register: http_check

    - name: Mostrar resultado HTTP
      ansible.builtin.debug:
        var: http_check.status
```
![anotherone](/images/134.png)
2. Ejecuto la tarea usando
```bash
ansible-playbook -i inventory.ini manage_servicio_nginx.yml
```
![ejecucion anotherone](/images/135.png)
3. Aunque ya esté dentro de la tarea, puedo comprobarlo:
```bash
ansible all -i inventory.ini -m shell -a "curl -s -o /dev/null -w '%{http_code}\\n' localhost:80 || echo 'ERROR' "  
```
!comprobacion anotherone[](/images/136.png)

---

## ORGANIZACIÓN DE PROYECTOS EN ANSIBLE
Creo ahora estos archivos donde:
+ **store_app.ini**: es una especie de directorio donde se separan los hosts en inventarios.
+ **vars_store_app.yml**: son las variables y contienen archivos que contienen la declaración de variables que usaremos en los playbooks
![mas archivos nuevos](/images/142.png)

---

## ROLES
### Cómo crear un rol
**Ansible** tiene un comando automático para crear la estructura de rol usando:
```bash
ansible-galaxy init nginx
```
![estructura rol](/images/137.png)
Creo el siguiente *playbook* en el que instalo y configuro **nginx**
```bash
nano manage-create-ngins.yml
```
Con el contenido
```yml
---
- hosts: web
  become: yes

  tasks:
    - name: Instalar nginx
      apt:
        name: nginx
        state: present

    - name: Copiar index.html
      copy:
        src: index.html
        dest: /var/www/html/index.html

    - name: Arrancar nginx
      service:
        name: nginx
        state: started
        enabled: yes
```
![playbook ngins](/images/138.png)
Quedaría de la siguiente forma:
```bash
sudo nano site.yml
```
```yml
---
- hosts: app_nodes
  become: yes

  roles:
    - nginx
```
![forma de playbook](/images/139.png)
Creo el siguiente directorio:
```bash
mkdir roles
cd roles
mkdir nginx
cd nginx
mkdir tasks
cd tasks
nano main.yml
```
Y le añado lo siguiente:
```yml
---
- name: Instalar nginx
  apt:
    name: nginx
    state: present

- name: Copiar index.html
  copy:
    src: index.html
    dest: /var/www/html/index.html

- name: Arrancar nginx
  service:
    name: nginx
    state: started
    enabled: yes
```
![creacuib de directorios](/images/143.png)
**Ansible.cfg** es el archivo en el que se definen los parámetros de funcionamiento de **Ansible**, por ejemplo:
```bash
nano ansible.cfg
```
```bash
[defaults]
inventory = inventory/inventory.ini
roles_path = roles
# silenciar las notificaciones de python
interpreter_python = auto_silent
```
![fichero ansible.cfg](/images/144.png)
Y se ejecutaría directamente:
```bash
ansible-playbook site.yml
```
+ **Playbook** → describe QUÉ hacer sobre unos hosts.
+ **Rol** → organiza tareas relacionadas para reutilizarlas y mantener el proyecto limpio.
![ejecucion de playbook ngins](/images/145.png)

