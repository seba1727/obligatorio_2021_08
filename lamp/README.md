# Obligatorio 2021 - 08 Roles Stack LAMP
-------------------------------------------
# Modificaiones realizadas

# Creacion de un archivo "ansible.cfg" con las siguientes directivas
[defaults]
inventory = hosts/inventario.ini
deprecation_warnings = False
remote_user = ansible
become_method = sudo

# Creacion de una carptea llamada "hosts". Dentro de ella se creo una archivo llamado ".ini" con la configuracion de los host.

En donde "dbservers" son hosts que distribuciones SO CentOS y donde "webservers" son hosts que distribuciones SO Ubuntu
[dbservers]
192.168.0.110

[webservers]
192.168.0.111

# Roles - common

# El rol "common" se crearon dos playbooks llamados "deploy-on-centos" y "deploy-on-ubuntu" para dezplegar las configuraciones a las dos distribuciones ya configuradas.

# En ambos se configuro para que se instale el servicio "chrony" asi como sus configuraciones respectivas para "firewalld" y "ufw".
           CentOS                              
 - chrony
      - python3-libselinux             
      - python3-libsemanage
      - git
      
            Ubuntu 
        
        - chrony
        - ufw
        - git

# En ambos casos se configuro una tarea para copiar el archivo de configuracion a ambas distribuciones .La carpeta "templates" fue movida a la carpeta de "tasks" y el archivo "ntp.conf.j2" fue renombrado a "chrony.conf.j2" y configurado de la siguiente manera:
keyfile /etc/chrony/chrony.keys


driftfile /var/lib/chrony/chrony.drift

logdir /var/log/chrony

maxupdateskew 100.0

makestep 1 3

# En centos al final del playbook se agrego un handler con una tarea a cumplir: handlers:
            - name: restart chronyd
              service: 
                      name: chronyd
                      state: restarted


# En el archivo "main.yml" se configuro para que ambas distribuciones se desplieguen.
  - name: Deploy on dbservers
    import_playbook: deploy-on-centos.yml
    

  - name: Deploy on webservers
    import_playbook: deploy-on-ubuntu.yml

# Roles - DB
El archivo "main.yml" se cambio de nombre a "install_db.yml"

La carpeta "templates" y le archivo de variables "dbservers" paso a estar dentro de la carpeta "tasks"

Se modifico la tarea de "install MariadDb package" en su sintaxis y paquetes a instalar.
   yum: 
      name: "{{ item }}" 
      state: present
    loop:
      - mariadb-server
      - python3
# Se arego una tarea para que se verifique que "pmysql" este instalado
- name: Make sure pymysql is present
    pip:
      name: pymysql
      state: present

# Se agrego una regla para el "Firewall"

- name: Firewall reload
    shell: firewall-cmd --reload
    notify: restart mariadb

# Y al final del archivo se agreo un handlers

handlers:
  - name: restart mariadb
    service: name=mariadb state=restarted 

# ·Roles - WEB

# EN el rol web para la instalacion de apache se configuro desde cero para adaptarlo para la distribucion "Ubuntu". Ademas de la tarea de installar httpd, tambien se agregaron dependencias.

apt: 
      name: '{{ item }}'
      state: present
  loop:
        - apache2
        - php-mysql
        - git
        - php
        - libapache2-mod-security2
        - libapache2-mod-php

# Se agrego una configuracion para "ufw" con un puerto.
- name: Allow all access to tcp port 80
  community.general.ufw:
      rule: allow
      port: '80'
      proto: tcp   

# Y agregamos un par de tareas para asegurarse que el servicio corra correctamente:

- name: Apache2 start
  ansible.builtin.systemd:
      name: apache2
      state: started
      enabled: yes

- name: Make sure apache2 is running
  ansible.builtin.systemd:
      state: started
      name: apache2

# Tanto el playbook "copy-code.yml como "install_httpd.yml" van a ser corridos con el playbook "main.yml" con la siguiente configuracion.
  
- include: install_httpd.yml
- include: copy_code.yml  

# Todo el stack de roles va a ser corrido por el playbook "site.yml"
  # Se modifico la los hosts a los que cada playbook iba destinado.






