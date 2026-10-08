# helper_merge_kv

Merges two lists of key-value mappings into a dictionary. Entries from the
`override` list are processed after `default` entries, so the last value wins.
A configurable sentinel can remove a key from the result.

The role is intended for string-based key-value data, such as environment
variables. All keys and values are converted to strings.

Entries from `helper_merge_kv_default` are processed before entries from
`helper_merge_kv_override`. If a key appears more than once, the last value
wins. The removal token removes a key only when it exactly matches the value.
The output is a host-scoped fact created with `set_fact`.

## Usage

```yaml
- name: Merge environment variables
  ansible.builtin.include_role:
    name: system4u.infra.helper_merge_kv
  vars:
    helper_merge_kv_default:
      - key: APP_ENV
        value: production
      - key: LOG_LEVEL
        value: info

    helper_merge_kv_override:
      - key: LOG_LEVEL
        value: debug
      - key: DEBUG
        value: "true"

    helper_merge_kv_output: merged_environment
```

The result is:

```yaml
merged_environment:
  APP_ENV: production
  LOG_LEVEL: debug
  DEBUG: "true"
```

## Removing a key

Use the default removal token:

```yaml
helper_merge_kv_override:
  - key: LOG_LEVEL
    value: "__remove__"
```

Or configure a different token:

```yaml
helper_merge_kv_remove_token: "__delete__"
```

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `helper_merge_kv_default` | required | Base list of mappings containing `key` and `value` |
| `helper_merge_kv_override` | `[]` | Values applied after the defaults |
| `helper_merge_kv_output` | required | Name of the output fact variable |
| `helper_merge_kv_remove_token` | `__remove__` | Value that removes a key |
| `helper_merge_kv_no_log` | `true` | Hide merge values from Ansible output |

The output is stored as a host fact using `set_fact`. Choose an output name
that does not conflict with another variable. The role is intentionally
`no_log` by default because values may contain credentials or tokens.
