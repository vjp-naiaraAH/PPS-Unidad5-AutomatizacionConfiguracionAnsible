# Actividad Unidad 5 - Automatización de la Configuración: Ansible
## PREPARANDO EL LABORATORIO
Creo una carpeta para la actividad y al haber realizado la de kubernetes puedo copiar la carpeta con los archivos:
```bash
# crea carpeta donde vas a realizar 
mkdir -p Actividad-Ansible
# copiar los archivos de la actividad de kubernetes
cp -rp Actividad_despliegue_app_store/* Actividad-Ansible/
# colocarse en la carpeta
cd Actividad-Ansible``` 
Además al coger los archivos de la actividad de kubernetes NO NECESITO descomprimir nada 
![](/images/100.png)

La descarga de los archivos no necesito hacerla porque ya se usaron en la actividad de kubernetes, aún así adjunto captura de la ejecución del comando:
```bash
ls
```
![](/images/101.png)

Compruebo que tengo instalado kubectl en el equipo con el comando
```bash
kubectl version --client=true
```
![](/images/102.png)

Inicio minikube con los comandos
```bash
# Iniciar un clúster de 3 nodos
minikube start --nodes 3 -p ansible
# -p ansible es el nombre del perfil (puedes elegir otro).
# Verifica los nodos creados
kubectl get nodes
```
Esto lo que ahce es crear un clúster con 
+ 1 nodo de control (ansible)
+ 2 nodos workers (ansible-m02, ansible-m03)
![](/images/103.png)

Etiqueto los nodos. Etiqueto los nodos para que los pods store-app se despliegue en modo ansible-m02 y el store-db en el nodo ansible-m03
```bash
# Etiquetar worker2 para aplicación
kubectl label nodes ansible-m02 app-role=frontend

# Etiquetar worker3 para base de datos  
kubectl label nodes ansible-m03 app-role=backend

# Verificar etiquetas
kubectl get nodes --show-labels
```
![](/images/104.png)

Me muevo a 
```bash
cd store-app
```
Para configurar el docker de `minikube, construir imágen y aplicar el despliegue:
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
![](/images/105.png)

Compruebo que la aplicación funciona navegando a http://192.168.76.2:30583 como indica al terminar los comandos que ejecutamos anteriormente los comandos.
![](/images/106.png)


## INSTALAR ANSIBLE EN EQUIPO LOCAL
Para la isntalación de Ansible ejecutaré los siguientes comandos:
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
+ Update a la máquina por si acaso tiene que actualizar librerias
+ Instala software-properties-common y add-apt-repository además de Ansible
+ Pide la versión de Ansible para saber si se ha instalado correctamente.
![](/images/107.png)

Compruebo de nuevo que se haya instalado ansible ejecutando
```bash
ansible --version  
```
![](/images/108.png)

## COMPROBAR Y CONFIGURAR SSH
Minikube crea una red interna automáticamente que nos va a permitir conectarnos automáticamente por SSH con los workers.
1. Verificar el acceso. Para verificar el acceso a los workers de Minikube debería tener acceso usando el comando:
```bash
# SSH a workers (¡funciona ya!)
minikube ssh --node m02 --profile=ansible
# exit para salir
```
En mi caso como se ve en la imágen a continuación puedo acceder dentro de m02
![](/images/109.png)

2. Comprobación en el nodo m03 y en el nodo de control usando:
```bash
minikube ssh --node m03 --profile=ansible
#exit  para Salir
minikube ssh  --profile=ansible
#exit  para Salir
```
Efectivamente también funciona.
![](/images/110.png)

3. Configurar SSH sin contraseña. Para ello establecemos una relación de confianza entre mi equipo y cada nodo. De esta forma podré conectar con ansible por SSH con cada uno de ellos
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
![](/images/111.png)
![](/images/112.png)

## CREAR INVENTARIO ANSIBLE
Creo el inventario para ansible. El inventario es un archivo en el cual está el nombre y la ip de un conjunto de nodos que están relacionados. Voy a crear el inventario de hosts llamado [crea_inventory.sh](https://github.com/jmmedinac03vjp/PuestaProduccionSegura/blob/main/Unidad5-SistemasDesplegadoAutomatizadoSoftware/ActividadAutomatizacionConfiguracionAnsible/files/crea_inventory.sh)
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
![](/images/113.png)

1. Una vez creado el archivo crea_inventory.sh, introduzco el contenido, le doy permisos y ejecuto, en mi caso al ya tener el archivo descargado y cargado en el paso anterior slo tengo que ejecutar:
```bash
# dar permisos de ejecución al script
chmod +x crea_inventory.sh
# ejecutar script
./crea_inventory.sh
```
Los avisos de python 3.11 se pueden evitar ejecutando:
```
sudo su
cat > ansible.cfg << 'EOF'
[defaults]
interpreter_python = auto_silent
EOF
exit
```
![](/images/114.png)

115 -> Y hago un cat al archivo inventory.ini para saber las direcciones ip.
```bash
cat inventory.ini
```
![](/images/115.png)

116 -> 2. Prueba: primer test Ansible
```bash
ansible-inventory -i inventory.ini --list
ansible all -i inventory.ini -m ping
```
![](/images/116.png)

## EJECUTAR MÓDULOS ANSIBLE
A continuación voy a hacer algunas tareas con Ansible:
### Usuarios y grupos
Utilizo los módulos group y user
1. En el siguiente playbook creo un usuario y un grupo con nombre maint que realizará labores de mantenimiento en todos los nodos:
![](/images/117.png)

2. Ejecuto el playbook:
```bash
ansible-playbook -i inventory.ini create_maint_user.yml
```
![](/images/118.png)

3. Compruebo que se han creado los usuarios. Se puede hacer con Ansible ya que aún no le he pasado la clave ssh.
```bash
ansible all_workers -i inventory.ini -m shell -a "id maint"
```
![](/images/119.png)

### File & copy
Para crear directorios hay que:
1. Crear el siguiente playbook:
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
![](/images/120.png)


2. Ejecuto el playbook
```bash
 ansible-playbook -i inventory.ini manage_maint_files.yml
```
![](/images/121.png)
