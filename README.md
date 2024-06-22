Задание выглядит так:

> Напиши ansible роль, устанавливающую пароль заданного пользователя (пусть будет с именем test) и флаг необходимости смены пароля при первом входе в систему.

Заметим:

-   В задании не сказано, что пользователя надо создать. Поэтому предположим, что он уже существует, и если не существует, то создавать его не будем во избежание дополнительных усложнений.
-   В задании не сказано, какая система на целевой машине. Поскольку у меня оказалась под рукой виртуальная машина с Windows, сделаем и под Linux, и под Windows, но без Active Directory и прочего.
-   В задании сказано \`\`роль'', а не \`\`плейбук''. Поэтому мы делаем роль, хотя и простую.


# Роль

Роль называется `user-password`. Структура каталогов:

```
.
├── bastion-testtask.yaml
├── inventory
│   └── inventory.yaml
├── roles
│   └── user-password
│       └── tasks
│           └── main.yaml
```

Основной плейбук `bastion-testtask.yaml`:

```yaml
---
- hosts: all
  roles:
    - user-password
```

В `inventory.yaml` предусмотрены группы `linux` и `win`, можно задавать отдельно `hosts: linux` или `hosts: win`. В `inventory.yaml` также заданы параметры для подключения к хостам.

Имя пользователя и новый пароль берутся из переменных, заданных в командной строке:

```bash
ansible-playbook ./bastion-testtask.yaml \
		 -i inventory/inventory.yaml \
		 --vault-password-file ../vault_pass \
		 -e user="test" \
		 -e password="hunter2"
```


## Задачи

```yaml
---
- name: Check connectivity Windows
  ansible.windows.win_ping:
  when: ansible_os_family == "Windows"
- name: Check connectivity Linux
  ansible.builtin.ping:
  when: ansible_system == "Linux"

- block:
    - name: Проверить, существует ли пользователь Linux
      become: true
      ansible.builtin.getent:
	database: passwd
	key: "{{ user }}"
      ignore_errors: true
      register: user_exists_linux
    - name: Сообщить, если пользователь Linux не существует
      ansible.builtin.fail:
	msg: "Пользователь Linux {{ user }} не существует."
      when: user_exists_linux.failed

    - name: Сменить пароль пользователя Linux
      become: true
      ansible.builtin.user:
	name: '{{ user }}'
	update_password: always
	password: "{{ password | password_hash }}"
	shell: /bin/bash
      no_log: true
      when: not user_exists_linux.failed

    - name: Установить флаг "пароль устарел"
      become: true
      ansible.builtin.command:
	cmd: passwd --expire "{{ user }}"
      when: not user_exists_linux.failed
  when: ansible_system == "Linux"

- block:  
    - name: Проверить, существует ли пользователь Windows
      ansible.windows.win_command:
	cmd: net user "{{ user }}"
      ignore_errors: true
      register: user_exists_win
    - name: Сообщить, если пользователь Windows не существует
      ansible.builtin.fail:
	msg: "Пользователь Windows {{ user }} не существует."
      when: user_exists_win.failed
    - name: Сменить пароль пользователя Windows
      ansible.windows.win_user:
	name: '{{ user }}'
	update_password: always
	password: "{{ password }}"
	password_expired: true
      no_log: true
      when: not user_exists_win.failed
  when: ansible_os_family == "Windows"
```


# Тестирование


## Linux ВМ

Для тестирования нам потребуется машина. Создадим ее с помощью [multipass](https://ubuntu.com/blog/using-cloud-init-with-multipass):

```bash
multipass launch mantic -n mantic --cloud-init ./cloud-config.yaml
```

Почему `mantic`, а не более поздние? Потому что `24 (noble)` более свежий Python, и с ним у `Ansible` проблемы. Файл `cloud-config.yaml`:

```yaml
#cloud-config
users:
  - default
  - name: ansible
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: true
    shell: /bin/bash
    ssh_authorized_keys:
      # Мой публичный ключ.
      - "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCzGFkQSRFoZus7piWi5CU32sao/2ST/DqOmLbAWmRKg58icjH+ae01tdQX5uqUSs+i4qsTiQQmDI/K7m4sPrrX1pZqoxKsUUADq49l6uSdbtr89s1YkuVjFObud3TWLhINCSkcd0j0EBPNodRUI3DtovFSWT6/0Xr+ZL2okdHDifozfS+TorT80mdQaSF65GzdZnmbVSFQcyUsPUCG2W1cv48TvG86gFoz3+7EdxIH6wa2Qa3rQgxjsr/y8jv58lXIai3BMnjfniKDvrX3l/QxNxysaFXdu44HmP3v5g0UDczS3LsfoJMkC4keoJylfVrJYdU1jKwi409ZvZTygqLg5U9ANRC+6a5K0EZAP0nkxZcmz5PSrP7xJ5wo58duLLU12IR/HFX3c6DvaK+LZjJMC231TE858UEANZHqVBH0i4Vj8VHoKYyAaGiHJR73kfnVu6UhTOoP1DtnU2KiJoToUbRlbKCoyngFZNtEkjwCz2aK4bn5lJmmp4qoFjq1Yu8= ivg@DESKTOP-U11D1U8"
```

```bash
multipass list --format yaml
```

```yaml
mantic:
  - state: Running
    ipv4:
      - 192.168.112.83
    release: Ubuntu 23.10
```


## Windows ВМ

Создание виртуальной машины с Windows и настройка ее для удаленного управления с помощью Ansible здесь не рассматривается.


## Inventory и параметры подключения

К хостам на базе Linux мы подключаемся с ключом ssh. Это не требует дополнительной настройки, так как публичный ключ задан при создании ВМ. К Windows мы подключаемся с логином и паролем, поэтому в файле пароль нужно скрыть. Используем Ansible Vault с очень сложным паролем (хранится в `../vault_pass`, чтобы случайно не пустить его в `git`) и запускаем следующим образом:

```bash
ansible-playbook -v ./test_ssh.yaml -i inventory/inventory.yaml --vault-password-file ../vault_pass
```

```yaml
linux:
  hosts:
    192.168.112.83:
      ansible_user: ansible
win:
  hosts:
    192.168.112.178:
      ansible_user: ansible
      ansible_password: !vault |
	$ANSIBLE_VAULT;1.1;AES256
	38326635333030323437653564646361323232386331616366326236656535303564333861613238
	6237343231336136616363663364623463356332626233310a633939323036333833656262393538
	36636264333264386362343837363463363231383337396466633862653163383163653333396439
	3038656662643931310a386564373961333766346635346466616234393138353038353962396437
	6661
      ansible_connection: winrm
      ansible_winrm_transport: ntlm
      ansible_port: 5985
```

```yaml
---
- name: Check connections to Windows targets.
  hosts: win
  tasks:
  - action: ansible.windows.win_ping
- name: Check connections to Linux targets.
  hosts: linux
  tasks:
  - action: ansible.builtin.ping
```