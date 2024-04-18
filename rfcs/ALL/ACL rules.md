# ACL rules

Starting from 0.11.0 release, zenoh provides the option of controlling access to key-expressions (eg: `test/demo` ) based on network interface (eg: `lo`). The access control is managed by filtering messages (denoted as **actions** in the acl config): `put`, `get`, `declare_subscriber`, `declare_queryable`. This filteration can be applied on both sides of the interceptor: the incoming messages (`ingress`) and outgoing messages (`egress`).

 The typical ACL configuration in the config.json5 file looks like this:

```json5
 acl: {
  ///[true/false] acl will be activated only if this is set to true
  "enabled": false,
  ///[deny/allow] default permission is deny (even if this is left empty or not specified)
  "default_permission": "deny",
  ///rule set for permissions allowing or denying access to key-expressions
  "rules": 
  [
    {
      "actions": [
        "put","get","declare_subscriber","declare_queryable"
      ],
      "flows":["egress","ingress"],
      "permission": "allow",
      "key_exprs": [
        "test/thr"
      ],
      "interfaces": [
        "lo0"
      ]
    },
 ]
}
```


The configuration has 3 fields:

1. **enabled**: true/false
2. **default_permission**: allow/deny 
3. **rules**: [ vector of rules ] This is where explicit Allow and Deny permissions are granted for access to key-expressions

The **enabled** field decides if the ACL is enabled or not. If it is set to `false`, no filtering of messages takes place and everything following that in the config is ignored.
The **default_permission** field provides the implicit permission for the filtering, i.e., this rule applies if no other matching rule is found for an **action**. If set to `allow`, it will allow all messages to pass through unless explicitly denied in the **rules** field. If set to `deny`, it blocks all messages and only allows those that are allowed by explicit rules provided in the **rules** field. The **default_permission** always has lower priority than explicit rules provided in the **rules** section.

The **rules** section itself has 5 inner fields: **actions**, **flows**, **permission**, **key_exprs**, **interfaces**

**actions**: supports four types of messsges - `put`, `get`, `declare_subscriber`, `declare_queryable`
**flows**: supports two values - `egress` and `ingress`
**permission**: supports `allow` and `deny`
**interfaces**: supports all possible value for network interfaces, eg: `lo`, `lo0` etc
**key_exprs**: supports values of any key type or key-expression (set of keys) type, eg: `temp/room_1`, `temp/**` etc


For each combination of **interface** + **flow** + **action**  (example:`l0`+`egress`+`put`) in the rules vector, we construct *allow* and *deny* KeTrees (key-expression tries). The *allow* KeTree is built using all the key-expressions provided in the config on which that **interface** is allowed to do that **action** on a particular **flow**. On receiving an authorization request, the key-expression in the request is matched against the appropriate KeTrees to confirm authorization. The priority of rules is as follows: 

*explicit deny* rule > *explicit allow* rule > *default_permission* rule

1. *Explicit deny* rules are given topmost priority. This means that if the key-expression in the  authorization request matches against anything in the *deny* KeTree, it will be denied immediately. If there is no match, then the *explicit allow* rules are checked.
2. *Explicit allow* permissions come next, i.e., if request matches anything in the *allow* tree then it will be allowed.
3. The **default_permission** value has least priority. It is applied only if there are no matches in the previous two steps.
    
    Note: if **default_permission** is set to `allow`, then there is no need for checking against *explicit allow* rules.
    

The decision logic is as follows:

```rust
fn is_allowed(key_expr) -> decision {
   if default_permission == DENY {
      if !deny_ketree.match(key_expr) && allow_ketree.match(key_expr) {
            return true;
      } else {
          return false;
      }
   } else { //default_permission = ALLOW
      if deny_ketree.match(key_expr) {
         return false;
      } 
			//no need to check for allow_ketree here
			return true; 
   }
}
```

An important thing to note here is how our key-expression matching works since it ultimately decides the behavior of the access control logic. In matching a key-expression against a KeTree, it will match as a positive if it is *equal to*, *subset of* or *superset of* anything in the KeTree. For example: key-expression `test/demo/a` will match if the KeTree contains any of these values: `test/demo/a` , `test/demo/**` , `test/demo/a/b`. Since *explicit deny* is always given preference over *explicit allow*, this means that if the incoming request on `test/demo/a` is denied if any of `test/demo/a`, `test/demo/**` or `test/demo/a/b` are present in the *deny* set of rules specified in the ACL.

The following table gives a better idea of of how the filtering will work on a key-expression (in Request column) and in the ruleset, you have explictly denied that action on another key-epression in the rules in the acl config:


|   Request    | In Deny Ruleset |  Result   |
|--------------|-----------------|-----------|
| test/demo/a  |  test/demo/**   | denied    |
| test/demo/** |  test/demo/a    | denied    |
| test/demo/a  |  test/demo/a/b  | denied    |
| test/demo/a/b|  test/demo/a    | denied    |
| test/demo/a  |  test/demo/**   | denied    |