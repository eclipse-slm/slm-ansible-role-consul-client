# slm-ansible-role-consul-client

Role Name
=========

An Ansible role for interacting with HashiCorp Consul, providing functionality for:
- Registering services with Consul agent
- Creating and managing Consul roles
- Storing and retrieving application configurations in Consul KV store
- Managing Consul configuration for applications

Requirements
------------

- Ansible 2.2 or higher
- Python 3.x
- `py-consul` Python package (installed automatically by the role)
- `community.general` collection (installed automatically by the role)
- Access to a running Consul instance
- Consul ACL token with appropriate permissions

Role Variables
--------------

### Required Global Variables

These variables must be defined in your playbook or inventory:

- `ENV.CONSUL.SCHEME.INTERNAL` - Scheme for Consul connection (http or https)
- `ENV.CONSUL.HOST.INTERNAL` - Hostname or IP address of Consul server
- `ENV.CONSUL.PORT.INTERNAL` - Port number of Consul HTTP API
- `_consul_token` - ACL token for authenticating with Consul

### Task-Specific Variables

#### register_service.yml
Registers a service with Consul agent.

**Required:**
- `_service_name` - Name of the service to register
- `_service_port` - Port number of the service
- `_service_address` - IP address of the service

**Optional:**
- `_additional_tags` - List of additional tags to add to the service (default: `["slm", "backend"]`)
- `_meta` - Dictionary of metadata to attach to the service
- `_http` - HTTP endpoint URL for HTTP health check
- `_tcp` - TCP address for TCP health check

#### create_or_add_config.yml
Creates or updates application configuration in Consul KV store.

**Required:**
- `_app_name` - Name of the application
- `_config` - Dictionary containing configuration values to store

**Behavior:**
- Retrieves existing config from Consul at `config/{{ _app_name }}/data`
- Combines existing config with new config (recursive merge)
- Stores combined config back to Consul
- Updates `app_configs` variable with the new configuration

#### store_app_config.yml
Stores application configuration from memory to Consul KV store.

**Required:**
- `_app_name` - Name of the application whose config should be stored

**Behavior:**
- Takes configuration from `app_configs[_app_name]` variable
- Stores it in Consul at `config/{{ _app_name }}/data`

#### get_app_config.yml
Retrieves application configuration from Consul KV store.

**Required:**
- `_app_name` - Name of the application whose config should be retrieved

**Behavior:**
- Retrieves config from Consul at `config/{{ _app_name }}/data`
- Stores it in `app_configs[_app_name]` variable

#### add_consul_config_in_consul.yml
Adds common Consul connection configuration to an application's config.

**Required:**
- `_app_name` - Name of the application

**Behavior:**
- Includes `create_or_add_config.yml` with Consul connection details
- Adds `consul.scheme`, `consul.host`, and `consul.port` to app config

#### create_role.yml
Creates Consul roles, policies, and binding rules.

**Required:**
- `_role_name` - Name of the role to create
- `_policies` - List of policies to create and assign to the role
  - Each policy should have `name` and `rules` fields
- `_binding_rule` - Binding rule configuration
  - `name` - Name of the binding rule
  - `description` - Description of the binding rule
  - `selector` - Selector expression for the binding rule

**Behavior:**
- Creates each policy in the `_policies` list
- Creates the role with the specified policies
- Creates a binding rule for the role with the keycloak auth method

Dependencies
------------

- `community.general` - Ansible collection for additional modules (consul_kv, consul_agent_service, consul_agent_check, consul_policy, consul_role, consul_binding_rule)

Example Playbook
----------------

### Basic Service Registration

```yaml
- hosts: servers
  vars:
    ENV:
      CONSUL:
        SCHEME:
          INTERNAL: "http"
        HOST:
          INTERNAL: "consul.example.com"
        PORT:
          INTERNAL: 8500
    _consul_token: "your-acl-token"
    
  roles:
    - role: consul-client
      tasks_from: register_service
      vars:
        _service_name: "my-service"
        _service_port: 8080
        _service_address: "192.168.1.100"
        _http: "http://192.168.1.100:8080/health"
```

### Managing Application Configuration

```yaml
- hosts: servers
  vars:
    ENV:
      CONSUL:
        SCHEME:
          INTERNAL: "http"
        HOST:
          INTERNAL: "consul.example.com"
        PORT:
          INTERNAL: 8500
    _consul_token: "your-acl-token"
    
  tasks:
    - name: Create initial config for app
      ansible.builtin.include_role:
        name: consul-client
        tasks_from: create_or_add_config
      vars:
        _app_name: "my-app"
        _config:
          database:
            host: "db.example.com"
            port: 5432
          logging:
            level: "info"
    
    - name: Add more config to existing
      ansible.builtin.include_role:
        name: consul-client
        tasks_from: create_or_add_config
      vars:
        _app_name: "my-app"
        _config:
          cache:
            enabled: true
            ttl: 3600
    
    - name: Get config for app
      ansible.builtin.include_role:
        name: consul-client
        tasks_from: get_app_config
      vars:
        _app_name: "my-app"
    
    - name: Print app config
      ansible.builtin.debug:
        var: app_configs
```

### Creating Consul Roles and Policies

```yaml
- hosts: servers
  vars:
    ENV:
      CONSUL:
        SCHEME:
          INTERNAL: "http"
        HOST:
          INTERNAL: "consul.example.com"
        PORT:
          INTERNAL: 8500
    _consul_token: "your-acl-token"
    
  tasks:
    - name: Create Consul role
      ansible.builtin.include_role:
        name: consul-client
        tasks_from: create_role
      vars:
        _role_name: "app-reader"
        _policies:
          - name: "app-read-policy"
            rules: |
              service "app" {
                policy = "read"
              }
        _binding_rule:
          name: "app-reader-binding"
          description: "Bind app-reader role to keycloak"
          selector: "datacenter == 'dc1'"
```

### Complete Example with Multiple Tasks

```yaml
- hosts: servers
  vars:
    ENV:
      CONSUL:
        SCHEME:
          INTERNAL: "http"
        HOST:
          INTERNAL: "consul.example.com"
        PORT:
          INTERNAL: 8500
    _consul_token: "your-acl-token"
    consul_service_name: "my-app"
    consul_service_port: 8080
    consul_service_address: "192.168.1.100"
    consul_service_config:
      database:
        host: "db.example.com"
        port: 5432
      redis:
        host: "redis.example.com"
        port: 6379
    consule_service_new_config_entry:
      cache:
        enabled: true
    
  pre_tasks:
    - name: Add Consul connection config
      ansible.builtin.include_role:
        name: consul-client
        tasks_from: add_consul_config_in_consul
      vars:
        _app_name: "{{ consul_service_name }}"
    
    - name: Register service with Consul
      ansible.builtin.include_role:
        name: consul-client
        tasks_from: register_service
      vars:
        _service_name: "{{ consul_service_name }}"
        _service_port: "{{ consul_service_port }}"
        _service_address: "{{ consul_service_address }}"
        _http: "http://{{ consul_service_address }}:{{ consul_service_port }}/health"
    
    - name: Store application config
      ansible.builtin.include_role:
        name: consul-client
        tasks_from: create_or_add_config
      vars:
        _app_name: "{{ consul_service_name }}"
        _config: "{{ consul_service_config }}"
    
    - name: Update config and store
      ansible.builtin.set_fact:
        app_configs: "{{ app_configs | combine({consul_service_name: app_configs[consul_service_name] | combine(consule_service_new_config_entry)}) }}"
    
    - name: Store updated config
      ansible.builtin.include_role:
        name: consul-client
        tasks_from: store_app_config
      vars:
        _app_name: "{{ consul_service_name }}"
    
  post_tasks:
    - name: Print all app configs
      ansible.builtin.debug:
        var: app_configs
```

License
-------

MIT

Author Information
------------------

- **Author**: Benjamin Goetz
- **Company**: Fraunhofer IPA
- **Namespace**: fabos
- **Role Name**: consul_client
