---
id: ansible
title: Ansible
---

# Ansible

## Tips

- 系统要求
  - 控制节点 - python \*nix
  - 管理节点 - python sftp/scp
- 注意 ⚠️
  - 分组名包含 `-` 会告警
    - `force_valid_group_names=ignore` 可关闭
  - docker_container 模块网络有所调整 - 之后默认不会添加 default 网络 - 与 docker 保持一直
    - 建议 `networks_cli_compatible=yes` 提前与 docker 网络保持一致
  - 建议使用 yaml 写 inventory - 比 ini 的模式好管理 - 结构也更加清晰
  - 相同 set_fact 不能互相依赖 [#40239](https://github.com/ansible/ansible/issues/40239)
- 参考
  - [Ansible tutorial](https://github.com/leucos/ansible-tuto)
  - [List all modules](http://docs.ansible.com/ansible/list_of_all_modules.html)
  - [ansible.cfg](https://raw.githubusercontent.com/ansible/ansible/devel/examples/ansible.cfg) 可用的 ansible.cfg 配置
  - 可用环境变量[列表](https://github.com/ansible/ansible/blob/devel/lib/ansible/constants.py)
  - [YAML Syntax](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)
- 学习
  - [geerlingguy/ansible-for-devops](https://github.com/geerlingguy/ansible-for-devops)
- 界面
  - [ansible-semaphore/semaphore](https://github.com/ansible-semaphore/semaphore)
    - Go 实现
    - 有 Web界面 和 [API](https://ansible-semaphore.github.io/semaphore/)
  - [ansible/awx](https://github.com/ansible/awx) - Ansible Tower 上游开源项目
    - 可 Docker Compose 启动
- 环境变量
  - `ANSIBLE_INVENTORY` - 逗号分隔的仓库源
  - `DEFAULT_HOST_LIST` - 默认仓库源 - `inventory` 配置

```bash
# ping 所有节点
ansible all -m ping
# -i 指定仓库
ansible all -m ping -i hosts
# ping 本地 - 指定解释器
ansible localhost -m ping -e 'ansible_python_interpreter=/usr/bin/python3'
# 执行命令
ansible all -a date -i hosts

# ansible_facts 内容
ansible localhost -m setup

# 节点上本地设置的内容 - 文件为 ini 格式
# /etc/ansible/facts.d/preferences.fact
ansible hostname -m setup -a "filter=ansible_local"
```

## ansible.cfg

- [Ansible Configuration Settings](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)
- 查找顺序
  - ANSIBLE_CONFIG
  - ansible.cfg
  - ~/.ansible.cfg
  - /etc/ansible/ansible.cfg

```ini
# 缓存 facts
[defaults]
gathering = smart
# 缓存时间 - 秒
fact_caching_timeout = 86400
# 缓存到 redis
# pip install redis
fact_caching = redis
# 缓存到 json 文件
fact_caching = jsonfile
fact_caching_connection = /path/to/cachedir

# 兼容 docker network - 如果指定了网络不添加默认网络
networks_cli_compatible=yes
# 不校验分组名字 允许包含 `-'
force_valid_group_names=ignore
```

## 变量查找路径

* [Variable precedence: Where should I put a variable?](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)

## 最佳实践

- [Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)

### 常用目录结构

```
production                # 正式环境仓库
staging                   # 预发环境仓库

group_vars/
    all.yml               # 全局变量
    group1.yml            # 分组变量
    group2.yml
host_vars/
    hostname1.yml          # 主机变量
    hostname2.yml

library/                  # 自定义模块
module_utils/             # 用于支持模块的模块工具
filter_plugins/           # 自定义过滤插件

site.yml                  # 主 playbook
webservers.yml            # playbook for webserver tier
dbservers.yml             # playbook for dbserver tier

roles/
    common/               # 角色结构
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies
        library/          # roles can also include custom modules
        module_utils/     # roles can also include custom module_utils
        lookup_plugins/   # or other types of plugins, like lookup in this case

    webtier/              # same kind of structure as "common" was above, done for the webtier role
    monitoring/           # ""
    fooapp/               # ""
```

### 独立仓库目录结构

如果仓库区别较大，可采用

```
inventories/
   production/
      hosts               # production 仓库主机定义
      group_vars/
          all.yml         # production 全局变量
          group1.yml      # 分组变量
          group2.yml
      host_vars/
          hostname1.yml   # 主机变量
          hostname2.yml

   staging/
      hosts               # staging 仓库主机定义
      group_vars/
          group1.yml       # here we assign variables to particular groups
          group2.yml
      host_vars/
          stagehost1.yml   # here we assign variables to particular systems
          stagehost2.yml

library/
module_utils/
filter_plugins/

site.yml
webservers.yml
dbservers.yml

roles/
    common/
    webtier/
    monitoring/
    fooapp/
```

## Generate ansible directories

```
mkdir gourp_vars host_vars
for ROLE in "apache" "web";do mkdir -p roles/${ROLE}/{files,handlers,tasks}; done
```

## 变量加载顺序

- role defaults
- inventory vars
- inventory group_vars
- inventory host_vars
- playbook group_vars
- playbook host_vars
- host facts
- registered vars
- set_facts
- play vars
- play vars_prompt
- play vars_files
- role and include vars
- block vars (only for tasks in block)
- task vars (only for the task)
- extra vars (always win precedence)

## Name Default Description

| Inventory Vars               | Description                                                          |
| ---------------------------- | -------------------------------------------------------------------- |
| ansible_ssh_host             | name of host Hostname or IP address to SSH to                        |
| ansible_ssh_port             | 22 Port to SSH to                                                    |
| ansible_ssh_user             | root User to SSH as                                                  |
| ansible_ssh_pass             | none Password to use for SSH authentication                          |
| ansible_connection           | smart How Ansible will connect to host (see below)                   |
| ansible_ssh_private_key_file | none SSH private key to use for SSH authentication                   |
| ansible_shell_type           | sh Shell to use for commands (see below)                             |
| ansible_python_interpreter   | /usr/bin/python Python interpreter on host (see below)               |
| `ansible_*_interpreter`      | none Like ansible_python_interpreter for other languages (see below) |

> 在新版本中,变量中的 `ssh` 已经去掉了

| Behavioral inventory parameter | ansible.cfg option                       |
| ------------------------------ | ---------------------------------------- |
| ansible_ssh_port               | remote_port                              |
| ansible_ssh_user               | remote_user                              |
| ansible_ssh_private_key_file   | private_key_file                         |
| ansible_shell_type             | executable (see the following paragraph) |

### ansible-vault 命令

| Command                        | Description                                       |
| ------------------------------ | ------------------------------------------------- |
| ansible-vault encrypt file.yml | Encrypt the plaintext file.yml file               |
| ansible-vault decrypt file.yml | Decrypt the encrypted file.yml file               |
| ansible-vault view file.yml    | Print the contents of the encrypted file.yml file |
| ansible-vault create file.yml  | Create a new encrypted file.yml file              |
| ansible-vault edit file.yml    | Edit an encrypted file.yml file                   |
| ansible-vault rekey file.yml   | Change the password on an encrypted file.yml file |

### SSH 多播选项

| Option         | Value                                   |
| -------------- | --------------------------------------- |
| ControlMaster  | auto                                    |
| ControlPath    | \$HOME/.ansible/cp/ansible-ssh-%h-%p-%r |
| ControlPersist | 60s                                     |

### AnsibleModule 参数属性

| Option   | Description                                                                  |
| -------- | ---------------------------------------------------------------------------- |
| required | If True, argument is required                                                |
| default  | Default value if argument is not required                                    |
| choices  | A list of possible values for the argument                                   |
| aliases  | Other names you can use as an alias for this argument                        |
| type     | Argument type. Allowed values: 'str', 'list', 'dict', 'bool', 'int', 'float' |

### AnsibleModule 初始化参数

| Parameter Default       | Description                                                   |
| ----------------------- | ------------------------------------------------------------- |
| argument_spec (none)    | Dictionary that contains information about arguments          |
| bypass_checks           | False If true, don’t check any of the parameter constrains    |
| no_log                  | False If true, don’t log the behavior of this module          |
| check_invalid_arguments | True If true, return error if user passed an unknown argument |
| mutually_exclusive      | None List of mutually exclusive arguments                     |
| required_together       | None List of arguments that must appear together              |
| required_one_of         | None List of arguments where at least one must be present     |
| add_file_common_args    | False Supports the arguments of the file module               |
| supports_check_mode     | False If true, indicates module supports check mode           |

### 调用外部命令

在 Ansible 模块中使用 `run_command` 调用外部命令

| Argument         | Type                      | Default                                                       | Description                                                                           |
| ---------------- | ------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| args (default)   | string or list of strings | (none) The command to be executed (see the following section) |
| check_rc         | Boolean                   | False                                                         | If true, will call fail_json if command returns a non-zero value.                     |
| close_fds        | Boolean                   | True                                                          | Passes as close_fds argument to subprocess.Popen                                      |
| executable       | string (path to program)  | None                                                          | Passes as executable argument to subprocess.Popen                                     |
| data             | string                    | None                                                          | Send to stdin if child process                                                        |
| binary_data      | Boolean                   | False                                                         | If false and data is present, Ansible will send a newline to stdin after sending data |
| path_prefix      | string (list of paths)    | None                                                          | Colon-delimited list of paths to prepend to PATH environment variable                 |
| cwd              | string (directory path)   | None                                                          | If specified, Ansible will change to this directory before executing                  |
| use_unsafe_shell | Boolean                   | False                                                         | See the following section                                                             |

### Documentation markup

| Type           | Syntax with example       | When to use           |
| -------------- | ------------------------- | --------------------- |
| URL            | U(http://www.example.com) | URLs                  |
| Module         | M(apt)                    | Module names          |
| Italics        | I(port)                   | Parameter names       |
| Constant-width | C(/bin/bash)              | File and option names |

## Tips

- 使用 merge [hash_behaviour](http://docs.ansible.com/ansible/intro_configuration.html#hash-behaviour) 可合并对象配置
