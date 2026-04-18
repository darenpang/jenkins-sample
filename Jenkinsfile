pipeline {
  agent any

  options {
    disableConcurrentBuilds()
  }

  parameters {
    choice(
      name: 'TARGET_PROFILE',
      choices: [
        'NonProd-tksis01d',
        'NonProd-tksis02d',
        'NonProd-tksis03d',
        'PreProd-tksis01d',
        'Prod-tksis01d'
      ],
      description: 'Select the target environment and login user profile.'
    )
    booleanParam(
      name: 'RUN_KEY_EXCHANGE',
      defaultValue: false,
      description: 'When enabled, Exchange SSH Key runs before Deploy.'
    )
    string(
      name: 'ANSIBLE_LIMIT',
      defaultValue: '',
      description: 'Optional Ansible --limit value. Leave empty to target the full profile group.'
    )
  }

  environment {
    ANSIBLE_INSTALLATION = 'Ansible 2'
    CONFIG_FILE = 'config/env-config.yml'
    INVENTORY_FILE = 'inventory/all.yml'
    EXCHANGE_PLAYBOOK = 'playbooks/exchange-ssh-key.yml'
    DEPLOY_PLAYBOOK = 'playbooks/deploy.yml'
  }

  stages {
    stage('Prepare Config') {
      steps {
        script {
          currentBuild.displayName = "#${env.BUILD_NUMBER} ${params.TARGET_PROFILE}"

          if (!fileExists(env.CONFIG_FILE)) {
            error("Missing pipeline config file: ${env.CONFIG_FILE}")
          }
          def config = readYaml file: env.CONFIG_FILE
          if (!(config instanceof Map)) {
            error("Invalid YAML structure in ${env.CONFIG_FILE}")
          }

          def vaultConfig = config.vault ?: [:]
          def profileConfig = config.profiles?."${params.TARGET_PROFILE}"

          if (!(profileConfig instanceof Map)) {
            error("Unknown TARGET_PROFILE: ${params.TARGET_PROFILE}")
          }

          def requiredVaultKeys = ['vault_credentials_id', 'vaulted_var_file', 'vaulted_var_name']
          def requiredProfileKeys = ['inventory_group', 'login_user', 'password_credential_id', 'ssh_key_credential_id']

          requiredVaultKeys.each { key ->
            if (!vaultConfig[key]?.toString()?.trim()) {
              error("Missing vault.${key} in ${env.CONFIG_FILE}")
            }
          }

          requiredProfileKeys.each { key ->
            if (!profileConfig[key]?.toString()?.trim()) {
              error("Missing profiles.${params.TARGET_PROFILE}.${key} in ${env.CONFIG_FILE}")
            }
          }

          env.TARGET_GROUP = profileConfig.inventory_group.toString()
          env.LOGIN_USER = profileConfig.login_user.toString()
          env.EXCHANGE_CREDENTIAL_ID = profileConfig.password_credential_id.toString()
          env.DEPLOY_CREDENTIAL_ID = profileConfig.ssh_key_credential_id.toString()
          env.VAULT_CREDENTIAL_ID = vaultConfig.vault_credentials_id.toString()
          env.VAULTED_VAR_FILE = vaultConfig.vaulted_var_file.toString()
          env.VAULTED_VAR_NAME = vaultConfig.vaulted_var_name.toString()
          env.ANSIBLE_LIMIT_VALUE = params.ANSIBLE_LIMIT?.trim() ?: ''

          echo "Resolved TARGET_PROFILE=${params.TARGET_PROFILE}"
          echo "Resolved inventory_group=${env.TARGET_GROUP}"
          echo "Resolved login_user=${env.LOGIN_USER}"
          echo "Resolved exchange_credential_id=${env.EXCHANGE_CREDENTIAL_ID}"
          echo "Resolved deploy_credential_id=${env.DEPLOY_CREDENTIAL_ID}"
          echo "Resolved vault_credential_id=${env.VAULT_CREDENTIAL_ID}"
        }
      }
    }

    stage('Exchange SSH Key') {
      when {
        expression { params.RUN_KEY_EXCHANGE }
      }
      steps {
        script {
          if (!fileExists(env.INVENTORY_FILE)) {
            error("Missing Ansible inventory file: ${env.INVENTORY_FILE}")
          }
          if (!fileExists(env.EXCHANGE_PLAYBOOK)) {
            error("Missing Exchange SSH Key playbook: ${env.EXCHANGE_PLAYBOOK}")
          }
          if (!fileExists(env.VAULTED_VAR_FILE)) {
            error("Missing vaulted variable file: ${env.VAULTED_VAR_FILE}")
          }

          def playbookArgs = [
            installation      : env.ANSIBLE_INSTALLATION,
            inventory         : env.INVENTORY_FILE,
            playbook          : env.EXCHANGE_PLAYBOOK,
            credentialsId     : env.EXCHANGE_CREDENTIAL_ID,
            vaultCredentialsId: env.VAULT_CREDENTIAL_ID,
            colorized         : true,
            extraVars         : [
              target_group    : env.TARGET_GROUP,
              login_user      : env.LOGIN_USER,
              vaulted_var_file: env.VAULTED_VAR_FILE,
              vaulted_var_name: env.VAULTED_VAR_NAME
            ]
          ]

          if (env.ANSIBLE_LIMIT_VALUE) {
            playbookArgs.limit = env.ANSIBLE_LIMIT_VALUE
          }

          ansiblePlaybook(playbookArgs)
        }
      }
    }

    stage('Deploy') {
      steps {
        echo "TODO: implement Deploy stage for ${params.TARGET_PROFILE} using ${env.DEPLOY_CREDENTIAL_ID}"
      }
    }
  }
}
