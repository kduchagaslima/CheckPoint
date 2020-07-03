# Ansible Module - check_point_mgmt by Check Point®

Note
-----
This repository is being deprecated.  
Starting with Ansible 2.9 release, users are encouraged to use the official Check Point integration.  
For more details, see:  
https://www.ansible.com/cybersecurity-automation-and-prevention-with-check-point-and-ansible<br>
There are two ways to obtain the official Check Point integration:
1. Download the official Check Point Management API modules collection from the Ansible Galaxy:      https://galaxy.ansible.com/check_point/mgmt (recommended)
2. Use the Check Point Management API modules included in the core of Ansible 2.9.



Installation instructions
-------------------------
1. Clone the repository with this command:
```git
git clone https://github.com/CheckPointSW/cpAnsible
```
or by clicking the Download ZIP button.   
2. Download and install the [Check Point API Python SDK](https://github.com/CheckPointSW/cp_mgmt_api_python_sdk) repository, follow the instructions in the SDK repository.<br>
3. [Install Ansible.](http://docs.ansible.com/ansible/intro_installation.html)  
4. [Install a Python interpreter with version >= 2.7.9, but not Python 3 or above.](https://www.python.org/downloads/)  
5. In your `/etc/ansible/ansible.cfg` file, there is a `library` line, uncomment it and set it to be whatever you want, this will be your library folder for modules.  
6. Move the `check_point_mgmt` folder to your library folder.    
7. Edit `/etc/ansible/hosts` so that it would contain a section similar to this one:  

```sh
[localhost]
127.0.0.1
[localhost:vars]
ansible_httpapi_use_ssl: False
FINGERPRINT: "46:E5:0E:1C:D7:34:2E:00:44:6A:06:FD:F5:22:04:FD:54:DA:22:FB" --> basta executar o seguinte comando na Manager ([Expert@servermgmt:0]# api fingerprint)
VPN_COMUNITY: "VPN_SCDE_API" --> Comunidade de VPN utilizada no processo de automação
GRP_BLOCK_WAN: "GRP_NO_BLOCK_WAN_II" --> Grupo de objetos utilizado para bloqueio ao ambiente da CCEE
MAILSERVER: "pxim01" --> servidor de email
```

Usage
-----

Run a playbook:
```sh
ansible-playbook your_ansible_playbook.yml
```
```

Before connecting to a Check Point management server for the first time, follow the instructions in step #4.
Otherwise, an "unverified server" error could appear.


Description
-----------
This Ansible module provides control over a Check Point management server using
Check Point's web-services APIs.
The web-services API reference can be found here:
https://sc1.checkpoint.com/documents/latest/APIs/#introduction~v1.6%20

A typical ansible command in a playbook should look like this:
  ```yaml
  - name: "adding a demo host"
    check_point_mgmt:
      command: add-host                     # web-service command name.
      parameters:                           # The web-service request arguments.
                                            # Note that the API web-services samples use JSON format.
                                            # Ansible requires these arguments to use YAML format.
        name: "host_demo2"
        ip-address: "1.2.3.5"
      session-data: "{{ login_response }}"  # where {{ login_response }} is received from
                                            # the login command that was called previously.
                                            # This replaces the need for the HTTP
                                            # headers that are mentioned in the API reference.
  ```
###  Notes:

  1. Como este módulo Ansible está controlando o servidor de gerenciamento remotamente por meio da API da web,
     o servidor ansible precisa ter acesso ao servidor da API Check Point. Abra o `SmartConsole`, navegue até 
     "Manage & Settings > Blades > Management API > Advanced settings" e verifique as configurações de 
     acessibilidade do servidor API.

  2. O campo "host" do playbook a ser executado deve se referir à máquina do Ansible (ou seja, 127.0.0.1),
     não há necessidade de ssh para o servidor de gerenciamento, os comandos são enviados para o servidor de 
     gerenciamento como solicitações de serviço da web da API.

  3. Comandos assíncronos - Alguns comandos da API Check Point APIs rodam de forma assíncrona, 
     por exmplo: publish, install-policy and run-script.
     Por padrão, todos os comandos em um playbook ansible serão executados de forma síncrona.
     Em outras palavras, o playbook ansible aguardará o término do comando, antes de passar para o próximo comando.
     Se você precisar substituir esse comportamento padrão e executar o comando de maneira assíncrona,
     use a opção "async: <timeout_in_seconds>" do Ansible, para obter mais informações:
     http://docs.ansible.com/ansible/playbooks_async.html

  4. O Ansible possui um recurso chamado "Check Mode" que permite testar a alteração sem realmente mudar nada.
     Ao executar no Ansible's "check mode", chamando a API "publicar" do Check Point
     onde chamaria a API de "discard" da Check Point.
     Isso permitiria testar seu script sem fazer alterações no banco de dados do produto Check Point
     ou manter objetos bloqueados.
     Nesse modo, as APIs "install-policy", "run-script" e "add-domain" serão ignoradas.
  5. Logout - A última tarefa do seu playbook deve ser `logout ', para fechar a sessão de maneira limpa. 
  Exemplo simples:
  ```yaml
  - name: logout
    check_point_mgmt:
      command: logout
      session-data: '{{login_response}}'
  ```



Toda comunicação com o servidor de gerenciamento Check Point deve começar com um comando "login".
O comando login usa alguns parâmetros que não estão listados na referência da API,

A lista dos parâmetros utilizados pelo login pode ser vista na seção [options] (# options):
```
Example of a login task:
```yaml
- name: "login task"
  check_point_mgmt:
    command: login
    parameters:
      username: user1
      password: pass1
      management: 192.168.1.193
    fingerprint: "7D:FE:DE:EE:C7:B9:D0:67:35:E4:C4:16:EC:7A:03:13:48:CD:CA:8D"
  register: login_response    # Register the output from the login command so
                              # it can later be used by subsequent commands.
```

Você deve adicionar o campo "register" do Ansible à tarefa Login, para que outras APIs de gerenciamento
possam continuar a sessão.

```yaml
  - name: "adding a demo host"
    check_point_mgmt:
      command: add-host                     # web-service command name.
      parameters:                           # The web-service request arguments.
                                            # Note that the API web-services samples use JSON format.
                                            # Ansible requires these arguments to use YAML format.
        name: "host_demo2"
        ip-address: "1.2.3.5"
      session-data: "{{ login_response }}"  # The session data we received from
                                            # the login command is used here.
                                            # This replaces the need for the HTTP
                                            # headers that are mentioned in the API reference.
```

Options
-------
**command** - o comando a ser excutado na Manager (Required)  
**parameters** - Os parâmetros do comando (dado como um dicionário - chave: valor)  
**fingerprint** - Impressão digital para verificar a identidade do servidor (Utilizado apenas para a tas de login).  
**session-data** - Após o login, use essa variável para fornecer a saída do comando login, incluindo detalhes como o ID da sessão, o IP e a porta do servidor de gerenciamento, o domínio no qual efetuar login e a fingerprint.

### Caso especial para o comando 'login':

### parameters:
  management - IP da Manager CheckPoint  
  domain - Domínio da Manager (utilizado em ambientes mult-domain)  
  port - Porta de acesso a Web Console da Manager (Default: 443).  
  username - Check Point api username  
  password - Check Point api password  


Example playbook
----------------
```yaml
---
- hosts: "localhost"                        # Note #2 in the Description section
  tasks:
  - name: "login"                           # You have to login to the management
                                            # server before running any commands
    check_point_mgmt:
      command: login
      parameters:
        username: "{{mgmt_user}}"           # Variables set in /etc/ansible/hosts, to avoid needing
        password: "{{mgmt_password}}"       # to type your login details in every playbook.
        management: "{{mgmt_server}}"
      fingerprint: "C3:B4:54:8D:0E:C0:7E:D4:31:64:94:13:F5:14:CD:93:68:6A:4F:4D"
    register: login_response                # Register the output from the login
                                            # command so we can use it later to run commands.
  - name: "add host"
    check_point_mgmt:
      command: add-host                     # Name of the command
      parameters:                           #  The parameters for it, in dictionary form
        name: "host_demo"
        ip-address: "1.2.3.5"
      session-data: "{{ login_response }}"  # The session data we received from
                                            # the login command is used here to run 'add-host'
  - name: "add group"
    check_point_mgmt:
      command: add-group
      parameters:
        name: "group_demo"
        members:
          - "host_demo"
      session-data: "{{ login_response }}"
  - name: "publish"                         # Publishing is important if you want
                                            # your changes to be saved.
    check_point_mgmt:                       # This will actually 'discard' when
                                            # check mode is enabled (ansible-playbook -C)
                                            # unless you add 'always_run: yes' to the task.
      command: publish
      session-data: "{{login_response}}"
```
