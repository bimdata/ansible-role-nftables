Ansible role Nftables
=========

This role installs and configures nftables.

Requirements
------------
* This role is tested with ansible-core 2.20 to 2.22.
* This role is tested on Debian 12, Debian 13, Ubuntu 24.04 and Ubuntu 26.04 (resolute).
* This role needs the `community.general` collection (>= 9.1.0).

Role Variables
--------------

You can find more informations about each nftables option in the [Nftables documentation](
https://wiki.nftables.org/wiki-nftables/index.php/Main_Page).

Variables used for the installation:

| Variables                          | Default value | Description                                            |
|------------------------------------|---------------|--------------------------------------------------------|
| nftables_enable_service_management | true          | Whether the role will manage nftables service or not.  |
| nftables_pkg_name                  | nftables      | The nftables APT package name.                         |
| nftables_svc_name                  | nftables      | The nftables service name to start/stop the daemon.    |
| nftables_svc_state                 | started       | The nftables service state in which nftable should be. |
| nftables_svc_enabled               | true          | If the nftables service should be enabled or not .     |


Variables used for the configuration:

| Variables                  | Default value      | Description                                                                           |
|----------------------------|--------------------|---------------------------------------------------------------------------------------|
| nftables_nft_bin           | /usr/sbin/nft      | Path for the nft executable, needed to run rule validation.                           |
| nftables_config_file       | /etc/nftables.conf | Path for the nftables configuration file.                                             |
| nftables_config_validation | true               | If the role need to verify the rules with nft -c -f before trying to apply the rules. |
| nftables_flush_ruleset     | true               | If the role should flush all the ruleset or just the table managed by it.             |

**nftables_flush_ruleset** can be usefull to help to not brake other tools managing their rules in nftables.
If `false`, instead of adding `flush ruleset` it will add `table <family> <name>` followed by `delete table <family> <name>` for each defined tables in `_nftables_tables` (equivalent to `destroy table`, which requires nftables >= 1.0.8).
So if you remove a table from your variables and use `nftables_flush_ruleset: false`, the removed table will not be destroy and you will need to destroy it manually.

Variables used for rules definition:

| Variables                     | Default value                                                                                                      | Description                                                                   |
|-------------------------------|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| nftables_table_use_by_default | `"{{ ((_nftables_tables \| length) == 1) \| ternary(_nftables_tables[0].table \| default(''), '') }}"`               | Define the default table to use when the value is not defined on each object. |
| _nftables_tables              | `"{{ lookup('template', 'get_vars.j2', template_vars=dict(app_name='nftables', var_type='table')) \| from_json }}"`  | Agregate table definitions.                                                   |
| _nftables_chains              | `"{{ lookup('template', 'get_vars.j2', template_vars=dict(app_name='nftables', var_type='chain')) \| from_json }}"`  | Agregate chain definitions                                                    |
| _nftables_defines             | `"{{ lookup('template', 'get_vars.j2', template_vars=dict(app_name='nftables', var_type='define')) \| from_json }}"` | Agregate define definitions.                                                  |
| _nftables_sets                | `"{{ lookup('template', 'get_vars.j2', template_vars=dict(app_name='nftables', var_type='set')) \| from_json }}"`    | Agregate set definitions.                                                     |
| _nftables_maps                | `"{{ lookup('template', 'get_vars.j2', template_vars=dict(app_name='nftables', var_type='map')) \| from_json }}"`    | Agregate map definitions.                                                     |
| _nftables_objects             | `"{{ lookup('template', 'get_vars.j2', template_vars=dict(app_name='nftables', var_type='object')) \| from_json }}"` | Agregate other objects definitions.                                           |
| _nftables_rules               | `"{{ lookup('template', 'get_vars.j2', template_vars=dict(app_name='nftables', var_type='rule')) \| from_json }}"`   | Agregate rule definitions.                                                    |


**In most cases, you should not modify the variables that start with an '_'.**
Templating is used to build these lists with other variables.
* `_nftables_tables` will aggregate all the variable whose name matches this regex: `^nftables_.+_table(s)?$`
* `_nftables_chains` will aggregate all the variable whose name matches this regex: `^nftables_.+_chain(s)?$`
* `_nftables_defines` will aggregate all the variable whose name matches this regex: `^nftables_.+_define(s)?$`
* `_nftables_sets` will aggregate all the variable whose name matches this regex: `^nftables_.+_set(s)?$`
* `_nftables_maps` will aggregate all the variable whose name matches this regex: `^nftables_.+_map(s)?$`
* `_nftables_objects` will aggregate all the variable whose name matches this regex: `^nftables_.+_object(s)?$`
* `_nftables_rules` will aggregate all the variable whose name matches this regex: `^nftables_.+_rule(s)?$`


Each variable matching these regexes must be:
  - a dictionnary defining one table/chain/rule/define/set/map/object **or**
  - a list of dictionnarty defining one or more table/chain/rule/define/set/map/object.

It allows you to define variables in multiple groups and cumulate them for hosts in multiples groups without the need to rewrite a complete list.

**By default, the role define no table nor chain. The ruleset is empty.**

**By default, `nftables_table_use_by_default` is set to the table only if you have only one table. You can override it if you have multiples tables.**

### Table definitions
Table can be define like this:
```
nftables_main_tables:
  table: inet main
```
It **must** have a `table` in the form `<family> <name>`.
It **can** have a `comment` and some `flags` like `dormant`.

```
nftables_main_tables:
  table: inet main
  comment: Main firewall
  flags:
    - dormant
```

This exact same string is used everywhere a table is referenced like in `nftables_table_use_by_default`. For example:
```
nftables_filter_tables:
  - table: ip filter
  - table: ip6 filter

nftables_filter_chains:
  - name: forward
    table: ip filter
    rules:
      00_policy: type filter hook forward priority filter; policy drop;
  - name: forward
    table: ip6 filter
    rules:
      00_policy: type filter hook forward priority filter; policy drop;
```

### Define definitions
Definition can be define like this:
```
nftables_admin_lan_defines:
  admin_lan_ipv4: 192.0.2.0/25
  admin_lan_ipv6:
    - 2001:db8:ac:12::/64
    - 2001:db8:ac:13::/64

nftables_admin_vpn_defines:
  admin_vpn_ipv4: 198.51.100.0/24
  admin_vpn_ipv6: 2001:db8:ff::/64
```

Each **key** will be use as the name of a definition. Each **value** has the corresponding value.
The **key** is the exact same string to use in your rule when you want to reference it.
The **value** can be a **string** or a **list**.

### Set definitions
Set can be define like this:
```
nftables_admin_sets:
  admin_ipv4: |-
    type ipv4_addr
    flags interval
    auto-merge
    elements = {
        $admin_lan_ipv4,
        $admin_vpn_ipv4
    }
  admin_ipv6: |-
    type ipv6_addr
    flags interval
    auto-merge
    elements = {
        $admin_lan_ipv6,
        $admin_vpn_ipv6
    }
```

Each **key** will be use as the name of a definition. Each **value** has the corresponding value.
The **key** is the exact same string to use in your rule when you want to reference it.
The **value** is a **string**. It can be multilines, it will be properly indent in the ruleset.
The `table` is **optionnal** only if the rule go in the table define by `nftables_table_use_by_default`.


### Map definitions
Map can be define like this:
```
nftables_web_forward_map:
  web_forward_map: |-
    type inet_service : ipv4_addr
    elements = {
      80 : 192.0.2.1,
      443 : 192.0.2.1,
      8080 : 192.0.2.2,
      8443 : 192.0.2.2
    }
```

### Object definitions
Object are not really and nftables concept. This can be use to define almost anything in a table.
For example, this can be use to define `counter`, `quota`, or even `chain` if you really want.

It can be define like this:
```
nftables_ftp_ct_helper_object:
  type: ct helper
  name: ftp-standard
  content: |-
    type "ftp" protocol tcp;
    comment "Coontrack helper to allow FTP data"
```

It **must** have a `type` and an `name`.
It **can** have a `content`.
The `table` is **optionnal** only if the rule go in the table define by `nftables_table_use_by_default`.

In general, this is render has:
```
table inet my_table {
    type name { value }
}
```
This is why it can be use to define a lot of different things.

### Chain definitions
Chain can be define like this:
```
nftables_base_chains:
  - name: input
    table: inet main
    rules:
      00_policy: type filter hook input priority filter; policy drop;
      01_localhost: iif lo accept
      02_state: "ct state vmap { established : accept, related : accept, invalid : drop }"

  - name: output
    rules:
      00_policy: type filter hook output priority filter; policy accept;

nftables_prerouting_chain:
  - name: prerouting
    rules:
      00_policy: type filter hook prerouting priority filter;
```

It **must** have a `name`.
It **can** have a list of `rules` directly attach to it. This is usefull for policy defining for example.
The `table` is **optionnal** only if the rule go in the table define by `nftables_table_use_by_default`.

### Rule definitions
Has we previously see, `rules` can be define directly on a `chain`. But you can also define them like this:

```
nftables_admin_rules:
  table: inet main
  chain: input
  rules:
    10_admin_ssh_accept: |-
      # SSH
      ip saddr @admin_ipv4 tcp dport 22 accept
      ip6 saddr @admin_ipv6 tcp dport 22 accept
    11_admin_snmp_accept: |
      # SNMP
      ip saddr @admin_ipv4 udp dport 161 accept
      ip6 saddr @admin_ipv6 udp dport 161 accept

nftables_ftp_rules:
  - chain: prerouting
    rules:
      10_ftp_helper: tcp dport 21 ct helper set "ftp-standard"
  - chain: input
    rules:
      10_ftp_control: tcp dport 21 ct state new,established accept
      11_ftp_data: ct helper "ftp" accept
```

The `table` is **optionnal** only if the rule go in the table define by `nftables_table_use_by_default`.
It **must** have a `chain` name to reference in which `chain` the rule is set.
It **must** have a `rules` **dictionnary**.

#### Rule precedence
When you define rules in multiples places, all the `rules` dictionnary are merged into one per `chain`.

First, each variables is retrieve in the list `_nftables_rules` by alphabetical order. Then, all the dictionnaries of this list are merged with `combine`. So, if two variables define the same rule, it can be override: the last variable wins.

For example, `nftables_01_rule` will take precedence for the rule `10_precedence_illustration`.
```
nftables_00_precedence_rule:
  chain: input
  rules:
    10_precedence_illustration: "# This one does not apply"

nftables_01_precedence_rule:
  chain: input
  rules:
    10_precedence_illustration: "# This on does apply"
```

Then we combine the `rules` define directly in the chain with this resulting dictionnary. The rules defined is their own variables wins against rules from `chain.rules`.

For example
```
nftables_base_chains:
  - name: input
    rules:
      10_precedence_illustration: "This one does not apply"

nftables_precedence_rules:
  - chain: input
    rules:
      10_precedence_illustration: "# This on does apply"
```

Finally, variable than explictly define `table` will take precedence over the variable the doesn't. Even when they are before in alphabetical order.

For example:
```
nftables_00_precedence_rule:
  chain: input
  table: inet main
  rules:
    10_precedence_illustration: "# This one does apply"

nftables_01_precedence_rule:
  chain: input
  rules:
    10_precedence_illustration: "# This on does not apply"
```

Then when building the final configuration, the `rules` are written in the alphabetical order of key.

For example:
```
nftables_00_precedence_rule:
  chain: input
  table: inet main
  rules:
    10_precedence_illustration_2: "# This will be the second rule"
    05_precedence_illustration_1: "# This will be the first rule"
```
