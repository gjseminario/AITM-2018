# Práctica de Ansible

Vamos a conocer ANSIBLE; herramienta para el despliegue de aplicaciones y gestión de la configuración - http://www.ansible.com/

Sigue las instrucciones paso a paso con la ayuda del instructor. Las prácticas de realizarán en una máquina Ubuntu de Vagrant.

## Requisitos previos: Entorno local

* Instalar Ansible - http://docs.ansible.com/ansible/intro_installation.html

> NOTA: Windows no es compatible como máquina de control

* Abrir Git Bash (Windows) o Terminal (Linux/MacOSX)

## Configuración

```shell
vagrant up && vagrant ssh zape
cd /vagrant && mkdir -p ansible/hosts && cd ansible
sudo cp /etc/ansible/ansible.cfg .
vim ansible.cfg
```

> Tambien podemos descargar la configuración del respositorio oficial: https://github.com/ansible/ansible/blob/devel/examples/ansible.cfg

* Aseguramos que tenemos la siguiente configuración

```python
remote_user = ubuntu
host_key_checking = False
```

* Salvamos y salimos con *:x*

> NOTA: Orden de prioridad del fichero de configuración:
1. ANSIBLE_CONFIG (variable de entorno POSIX)
2. ansible.cfg (en carpeta actual)
3. ~/.ansible.cfg (en el directorio home del usuario ejecutor)
4. /etc/ansible/ansible.cfg

## "hello world"

```shell
vim hosts/all
```

* Añadimos...

```ini
[localhost]
127.0.0.1

[zape]
192.168.32.11

[base]
192.168.32.12
```

* Salvamos y salimos con *:x*
* Ejecutamos...

```shell
ansible localhost -i hosts/all -m ping
ansible zape -i hosts/all -m ping -k
```

* Introducimos la contraseña del usuario vagrant que es `c02b644df7ddd7348994b896`.

> NOTA: Contraseña Ubuntu Vagrant en Mac OSX: `cat ~/.vagrant.d/boxes/ubuntu-VAGRANTSLASH-xenial64/20170224.0.0/virtualbox/Vagrantfile`

* ¿Que sucede?

## Inventario (opcional)

```shell
vim hosts/all
```

* Actualizamos...

```ini
[localhost]
127.0.0.1

[base]
192.168.32.12

[base:vars]
ntp_server=es.pool.ntp.org
ansible_ssh_user=vagrant

[zape]
192.168.32.11 ansible_ssh_user=vagrant ansible_ssh_pass=vagrant

[entorno:children]
base
zape

[all:vars]
ansible_connection=ssh
ansible_ssh_user=ubuntu
;ansible_ssh_pass=c02b644df7ddd7348994b896
```

* Salvamos y salimos con *:x*

```shell
ansible all -i hosts/all -m setup --tree /tmp/facts -k
```

* Introducimos la contraseña del usuario vagrant que es *vagrant*.
* ¿Que sucede?

> También podemos obtener la información de inventorios dinámicos: http://docs.ansible.com/ansible/intro_dynamic_inventory.html

## Playbook

### Consultas

```shell
vim request.yml
```

* Introducimos el siguiente texto

```yaml
---
- hosts: zape
  tasks:
    - name: que sistema eres?
      command: uname -a
      register: info
    - name: imprimir variable
      debug: var=info
    - name: imprimir campo de variable
      debug: var=info.stdout

    - name: como te llamas?
      command: hostname
      register: info
    - name: dame tu nombre
      debug: var=info.stdout
```

* Ejecutamos

```shell
ansible-playbook request.yml -i hosts/all --list-tasks --list-hosts
```

* ¿Que sucede?

```shell
ansible-playbook request.yml -i hosts/all -k
```

* ¿Que sucede?
* Abrimos un navegador y vamos a http://docs.ansible.com/ansible/YAMLSyntax.html

### Aprovisionamiento

```shell
vim install.yml
```

* Introducimos el siguiente texto

```yaml
---
- hosts: zape
  tasks:
    - name: instalamos nginx
      become: true
      apt: name={{ package }} state=latest update_cache=yes cache_valid_time=3600

  vars:
    package: nginx

# Equivalente de "update apt-get", si la última actualización es de hace más de 3600 segundos
```

* Ejecutamos

```shell
ansible-playbook install.yml -i hosts/all -k
```

* Abrimos un navegador y vamos a http://localhost:8080/
* ¿Que sucede?


```shell
vim install.yml
```

* Introducimos el siguiente texto

```yaml
---
- hosts: zape
  tasks:
    - name: instalamos nginx
      become: true
      apt: name={{ package }} state=latest update_cache=yes cache_valid_time=3600
      when: ansible_os_family == "Debian"

# Equivalente de "update apt-get", si la última actualización es de hace más de 3600 segundos

    - name: despliegue de la aplicacion
      become: true
      template: src={{ template_name }} dest={{ destination }} owner=root group=root mode=0644 backup=yes

    - name: ls /usr/share/nginx/html/
      command: ls /usr/share/nginx/html/
      register: contents

    - name: mostrar variable
      debug: var=contents.stdout_lines

  vars:
    package: nginx
    template_name: "index.html.j2"
    destination: /usr/share/nginx/html/index.html
    url: "http://www.chiquitoipsum.com/"
    url_title: "chiquitoipsum"
    condicion: True
    demo: Ansible
```

* Salvamos y salimos con *:x*

```shell
vim index.html.j2
```

* Introducimos el siguiente texto

```html
<!DOCTYPE html>
<html>
<head>
<title>{{ demo }} test</title>
</head>
<body>
<h1>Estamos viendo el poder de {{ demo }}</h1>
<p>Lorem ipsum dolor sit amet, consectetur adipiscing elit. Mauris gravida.</p>

{% if condicion %}
    <p>Visita: <a href="{{ url }}">{{ url_title }}</a>
{% else %}
    <p>Visita: <a href="http://carlessanagustin.com/">carlessanagustin.com</a>
{% endif %}

<p><img src="http://www.socallinuxexpo.org/scale12x-supporting/default/files/logos/AnsibleLogo_transparent_web.png" width="497" height="393"></p>
</body>
</html>
```

* Salvamos y salimos con *:x*
* Ejecutamos...

```shell
ansible-playbook install.yml -i hosts/all -k
```

* ¿Que sucede?
* Abrimos un navegador y vamos a http://docs.ansible.com/ansible/template_module.html
* Abrimos un navegador y vamos a http://jinja.pocoo.org/docs/dev/

## Roles

```shell
mkdir roles && cd roles
ansible-galaxy init nginx
tree nginx
```

* ¿Que sucede?

```shell
vim nginx/tasks/main.yml
```

* Introducimos el siguiente texto

```yaml
---
- name: instalamos nginx
  become: true
  apt: name={{ item }} state=latest update_cache=yes cache_valid_time=3600
  when: ansible_os_family == "Debian"
  with_items: {{ packages }}

# Equivalente de "update apt-get", si la última actualización es de hace más de 3600 segundos

- name: despliegue de la aplicacion
  become: true
  template: src={{ template_name }} dest={{ destination }} owner=root group=root mode=0644 backup=yes

- name: ls /usr/share/nginx/html/
  command: ls /usr/share/nginx/html/
  register: contents

- name: mostrar variable
  debug: var=contents.stdout_lines      
```

* Salvamos y salimos con *:x*

```
vim nginx/defaults/main.yml
```

* Introducimos el siguiente texto

```yaml
---
packages:
  - nginx
  - vim
  - curl
template_name: "index.html.j2"
destination: /usr/share/nginx/html/index.html
url: "http://www.chiquitoipsum.com/"
url_title: "chiquitoipsum"
condicion: True
demo: Ansible     
```

* Salvamos y salimos con *:x*

```shell
cp ../index.html.j2 nginx/templates/
vim ../install2.yml
```

* Introducimos el siguiente texto

```yaml
---
- hosts: zape

  roles:
    - nginx
```

* Salvamos y salimos con *:x*

```
cd ..
ansible-playbook install2.yml -i hosts/all -k
```

## Comandos

* Los ya conocidos...

```shell
ansible <host-pattern> [options]
ansible-playbook playbook.yml
```

* Otros comandos incluidos en la instalación...

```shell
ansible-doc [options] [module...]
ansible-galaxy [init|info|install|list|remove] [--help] [options] ...
ansible-pull [options] [playbook.yml]
ansible-vault [create|decrypt|edit|encrypt|rekey|view] [--help] [options] file_name
```

# Preguntas y respuestas

# Eliminamos las instancias Vagrant

```
vagrant destroy -f
```

------

# Información extra

## Generar claves de autenticación de SSH

```
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa): /home/vagrant/.ssh/id_rsa_ansible
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/vagrant/.ssh/id_rsa_ansible.
Your public key has been saved in /home/vagrant/.ssh/id_rsa_ansible.pub.
The key fingerprint is:
50:e6:96:7b:5b:e7:70:bf:1e:82:b8:b9:75:60:dc:23 your_email@example.com
The key's randomart image is:
+--[ RSA 4096]----+
|        o        |
|       + .       |
|      . +        |
|       o .. .    |
|        S .Eooo  |
|         .oo+=.. |
|         ..o o...|
|          + . . o|
|         +.   .o |
+-----------------+
ssh-copy-id -i /home/vagrant/.ssh/id_rsa_ansible.pub vagrant@192.168.32.11
```

---

Creado por [carlessanagustin.com](http://www.carlessanagustin.com)