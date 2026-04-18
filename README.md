# Jenkins + Ansible CD Repository 式样

## 1. 这一套在干什么

### 1.1 目的

本 repository 用于通过 Jenkins 调用 Ansible，对 `NonProd`、`PreProd`、`Prod` 三个环境执行 CD 基础流程。

当前流程固定为两个 stage：

1. `Exchange SSH Key`
   - 通过 Jenkins 中登记的“用户名 + 密码”凭据登录目标主机。
   - 读取 Ansible Vault 加密的共通公钥变量 `sis_pubkey`。
   - 将该公钥写入目标登录用户自己的 `~/.ssh/authorized_keys`。
   - 是否执行，由 Jenkins 参数 `RUN_KEY_EXCHANGE` 控制。

2. `Deploy`
   - 通过 Jenkins 中登记的“用户名 + 私钥”凭据登录目标主机。
   - 当前仅执行 `ping` 和 `debug msg`，用于确认 SSH 登录和 Ansible 执行路径正常。
   - 后续可在此 stage 内追加正式部署处理。

### 1.2 执行单位

Pipeline 的执行单位不是“环境”本身，而是“目标 profile”。

原因如下：

- `inventory` 负责管理主机，按环境分组。
- `env-config.yml` 负责管理 Jenkins Credentials ID。
- `NonProd` 虽然只有一台主机 `jpn00001`，但存在多个登录用户，因此仅靠环境名无法唯一确定应使用哪组凭据。

因此，Pipeline 以 `TARGET_PROFILE` 为入口。

`TARGET_PROFILE` 的定义格式为：

- `<Environment>-<LoginUser>`

本仓库的 profile 一览如下：

- `NonProd-tksis01d`
- `NonProd-tksis02d`
- `NonProd-tksis03d`
- `PreProd-tksis01d`
- `Prod-tksis01d`

### 1.3 Jenkins Pipeline 的处理顺序

1. 读取参数：
   - `TARGET_PROFILE`
   - `RUN_KEY_EXCHANGE`
   - `ANSIBLE_LIMIT`（可选）
2. 读取 `config/env-config.yml`
3. 根据 `TARGET_PROFILE` 取得：
   - 对应的 inventory group
   - 登录用户名
   - 用户名 + 密码凭据 ID
   - 用户名 + 私钥凭据 ID
4. 当 `RUN_KEY_EXCHANGE = true` 时：
   - 使用用户名 + 密码凭据执行 `playbooks/exchange-ssh-key.yml`
   - 同时使用 `vaultCredentialsId = jenkins_deploy_vault_credentials` 解密 Vault 变量 `sis_pubkey`
5. 使用用户名 + 私钥凭据执行 `playbooks/deploy.yml`

## 2. 每个成果物里面应该有什么

### 2.0 仓库结构

本 repository 的预期目录结构如下：

```text
.
├─ Jenkinsfile
├─ README.md
├─ inventory/
│  └─ all.yml
├─ config/
│  └─ env-config.yml
├─ playbooks/
│  ├─ exchange-ssh-key.yml
│  └─ deploy.yml
└─ vars/
   └─ vault/
      └─ sis_pubkey.yml
```

结构说明：

- `Jenkinsfile`
  - Jenkins Pipeline 定义文件
- `README.md`
  - 本 repository 的式样说明
- `inventory/all.yml`
  - 按环境管理目标主机
- `config/env-config.yml`
  - 管理 profile 和 Jenkins Credentials ID 的映射关系
- `playbooks/exchange-ssh-key.yml`
  - 负责交换 SSH key
- `playbooks/deploy.yml`
  - 负责 deploy 基础验证
- `vars/vault/sis_pubkey.yml`
  - 保存使用 Ansible Vault 加密的共通公钥变量

### 2.1 `Jenkinsfile`

文件作用：

- 定义 Jenkins Pipeline。
- 作为本 repository 的执行入口。

应包含的内容：

1. 参数定义
   - `TARGET_PROFILE`
   - `RUN_KEY_EXCHANGE`
   - `ANSIBLE_LIMIT`

2. 配置读取
   - 读取 `config/env-config.yml`
   - 根据 `TARGET_PROFILE` 解析对应的 profile 配置

3. `Exchange SSH Key` stage
   - 执行条件：`RUN_KEY_EXCHANGE == true`
   - 使用 profile 中的 `password_credential_id`
   - 使用 `jenkins_deploy_vault_credentials` 作为 `vaultCredentialsId`
   - 调用 `playbooks/exchange-ssh-key.yml`

4. `Deploy` stage
   - 使用 profile 中的 `ssh_key_credential_id`
   - 调用 `playbooks/deploy.yml`

5. `ANSIBLE_LIMIT` 支持
   - 当该参数不为空时，追加到 Ansible 执行参数中

Jenkinsfile 中的参数式样如下：

```groovy
parameters {
  choice(name: 'TARGET_PROFILE', choices: [
    'NonProd-tksis01d',
    'NonProd-tksis02d',
    'NonProd-tksis03d',
    'PreProd-tksis01d',
    'Prod-tksis01d'
  ])
  booleanParam(name: 'RUN_KEY_EXCHANGE', defaultValue: false)
  string(name: 'ANSIBLE_LIMIT', defaultValue: '')
}
```

### 2.2 `inventory/all.yml`

文件作用：

- 管理目标主机。
- 仅按环境分组，不按用户分组。

应包含的内容：

```yaml
all:
  children:
    NonProd:
      hosts:
        jpn00001:

    PreProd:
      hosts:
        jpn10001:
        jpn10002:

    Prod:
      hosts:
        jpn20001:
        jpn20002:
```

约束：

- `NonProd` 固定包含 `jpn00001`
- `PreProd` 固定包含 `jpn10001`, `jpn10002`
- `Prod` 固定包含 `jpn20001`, `jpn20002`

### 2.3 `config/env-config.yml`

文件作用：

- 管理 Jenkins Credentials ID 和 profile 映射关系。
- 文件中只保存 Jenkins Credentials 的 ID，不保存密码、私钥、公钥正文。

应包含的内容：

```yaml
vault:
  vault_credentials_id: jenkins_deploy_vault_credentials
  vaulted_var_file: vars/vault/sis_pubkey.yml
  vaulted_var_name: sis_pubkey

profiles:
  NonProd-tksis01d:
    inventory_group: NonProd
    login_user: tksis01d
    password_credential_id: sis_tksis01d_password_nonprod
    ssh_key_credential_id: sis_tksis01d_ed25519_nonprod

  NonProd-tksis02d:
    inventory_group: NonProd
    login_user: tksis02d
    password_credential_id: sis_tksis02d_password_nonprod
    ssh_key_credential_id: sis_tksis02d_ed25519_nonprod

  NonProd-tksis03d:
    inventory_group: NonProd
    login_user: tksis03d
    password_credential_id: sis_tksis03d_password_nonprod
    ssh_key_credential_id: sis_tksis03d_ed25519_nonprod

  PreProd-tksis01d:
    inventory_group: PreProd
    login_user: tksis01d
    password_credential_id: sis_tksis01d_password_preprod
    ssh_key_credential_id: sis_tksis01d_ed25519_preprod

  Prod-tksis01d:
    inventory_group: Prod
    login_user: tksis01d
    password_credential_id: sis_tksis01d_password_prod
    ssh_key_credential_id: sis_tksis01d_ed25519_prod
```

字段定义：

- `vault.vault_credentials_id`
  - Jenkins 中登记的 Ansible Vault 解密用 credential ID
- `vault.vaulted_var_file`
  - 保存加密公钥变量的 yml 文件路径
- `vault.vaulted_var_name`
  - 加密公钥变量名，本仓库固定为 `sis_pubkey`
- `profiles.<PROFILE>.inventory_group`
  - 该 profile 对应的 inventory group
- `profiles.<PROFILE>.login_user`
  - 该 profile 使用的登录用户
- `profiles.<PROFILE>.password_credential_id`
  - `Exchange SSH Key` stage 使用的用户名 + 密码 credential ID
- `profiles.<PROFILE>.ssh_key_credential_id`
  - `Deploy` stage 使用的用户名 + 私钥 credential ID

### 2.4 `vars/vault/sis_pubkey.yml`

文件作用：

- 保存共通公钥。
- 文件内容使用 Ansible Vault 加密。

应包含的内容：

```yaml
sis_pubkey: !vault |
  xxxxx
```

约束：

- 变量名固定为 `sis_pubkey`
- 文件内容为 Vault 加密后的 YAML
- 该变量在 `Exchange SSH Key` stage 中被读取并写入 `authorized_keys`

### 2.5 `playbooks/exchange-ssh-key.yml`

文件作用：

- 将 `sis_pubkey` 写入目标登录用户的 `authorized_keys`

应包含的内容：

1. `hosts` 使用由 Jenkins 传入的目标 inventory group
2. `remote_user` 使用由 Jenkins 传入的登录用户
3. 读取 `vars/vault/sis_pubkey.yml`
4. 使用 `ansible.posix.authorized_key`
5. 写入对象为登录用户本人的 `~/.ssh/authorized_keys`

式样示例如下：

```yaml
---
- name: Exchange SSH key
  hosts: "{{ target_group }}"
  gather_facts: false
  remote_user: "{{ login_user }}"
  vars_files:
    - "{{ vaulted_var_file }}"
  tasks:
    - name: Install common public key
      ansible.posix.authorized_key:
        user: "{{ login_user }}"
        state: present
        key: "{{ sis_pubkey }}"
```

### 2.6 `playbooks/deploy.yml`

文件作用：

- 通过 SSH key 登录目标主机并进行基础验证

应包含的内容：

1. `hosts` 使用由 Jenkins 传入的目标 inventory group
2. `remote_user` 使用由 Jenkins 传入的登录用户
3. 执行 `ansible.builtin.ping`
4. 输出 `debug msg`

式样示例如下：

```yaml
---
- name: Deploy
  hosts: "{{ target_group }}"
  gather_facts: false
  remote_user: "{{ login_user }}"
  tasks:
    - name: Ping target
      ansible.builtin.ping:

    - name: Print deploy message
      ansible.builtin.debug:
        msg: "Deploy connectivity check succeeded on {{ inventory_hostname }} as {{ login_user }}"
```

### 2.7 Jenkins Credentials

成果物作用：

- 保存 Pipeline 执行时使用的全部认证信息。

Jenkins 中应登记的内容如下。

#### A. Vault 解密用 credential

- ID: `jenkins_deploy_vault_credentials`
- 用途：作为 `vaultCredentialsId`，用于解密 `vars/vault/sis_pubkey.yml` 中的 `sis_pubkey`

#### B. 用户名 + 私钥 credential

- `sis_tksis01d_ed25519_nonprod`
- `sis_tksis02d_ed25519_nonprod`
- `sis_tksis03d_ed25519_nonprod`
- `sis_tksis01d_ed25519_preprod`
- `sis_tksis01d_ed25519_prod`

#### C. 用户名 + 密码 credential

- `sis_tksis01d_password_nonprod`
- `sis_tksis02d_password_nonprod`
- `sis_tksis03d_password_nonprod`
- `sis_tksis01d_password_preprod`
- `sis_tksis01d_password_prod`

约束：

- `env-config.yml` 中只写这些 credential 的 ID
- 实际密码、私钥、Vault 解密信息只保存在 Jenkins 中

### 2.8 Jenkins Job / Pipeline 参数

成果物作用：

- 控制一次 Pipeline 执行的目标和行为。

应包含的内容：

- `TARGET_PROFILE`
  - 本次执行选择的 profile
- `RUN_KEY_EXCHANGE`
  - 是否执行 `Exchange SSH Key`
- `ANSIBLE_LIMIT`
  - 可选的主机限制条件

## 3. 整体对应关系

### 3.1 主机分组

- `NonProd` -> `jpn00001`
- `PreProd` -> `jpn10001`, `jpn10002`
- `Prod` -> `jpn20001`, `jpn20002`

### 3.2 Profile 与凭据的对应关系

- `NonProd-tksis01d`
  - password: `sis_tksis01d_password_nonprod`
  - ssh key: `sis_tksis01d_ed25519_nonprod`
- `NonProd-tksis02d`
  - password: `sis_tksis02d_password_nonprod`
  - ssh key: `sis_tksis02d_ed25519_nonprod`
- `NonProd-tksis03d`
  - password: `sis_tksis03d_password_nonprod`
  - ssh key: `sis_tksis03d_ed25519_nonprod`
- `PreProd-tksis01d`
  - password: `sis_tksis01d_password_preprod`
  - ssh key: `sis_tksis01d_ed25519_preprod`
- `Prod-tksis01d`
  - password: `sis_tksis01d_password_prod`
  - ssh key: `sis_tksis01d_ed25519_prod`
