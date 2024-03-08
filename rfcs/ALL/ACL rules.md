# Allow/Deny Rule Priority in ACL

If the ACL is enabled in the config, we have two fields that will set the rules to be checked for authorization:

1. *default_permission*: Allow/Deny 
2. *rules*: [ vector of rules ] here explicit Allow and Deny permissions are granted for access to key-expressions

For each subject+action combination (example: `localhost0`+`test/demo/a`) in the rules vector, we construct both an *Allow* and *Deny* KeTree (key-expression trie). The *Allow* KeTree contains all the key-expressions on which the subject is allowed to do that action. On receiving an authorization request, the key-expression in the request will be matched against the KeTrees to confirm authorization. The priority of rules is as follows: 

explicit *Deny* rule > explicit *Allow* rule > *default_permission* rule

1. Explicit *Deny* rules are given topmost priority. This means that if the key-expression in the  authorization request matches against anything in the *Deny* KeTree, it will be denied. 
2. Explicit *Allow* permissions come next, i.e., if request matches anything in the *Allow* tree then it will be allowed (unless denied in the previous step).
3. The *default_permission* value has least priority. It is applied only if there are no matches in the previous two steps.
    
    Note: if *default_permission* is set to *Allow*, then there is no need for checking against explicit *Allow* rules.
    

The decision logic looks as follows:

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

An important thing to note here is how our matching works since it has an effect on the behavior of the authorization logic. In matching a Key-expression against a KeTree, it will match as a positive if it is *equal to*, *subset of* or *superset of* anything in the KeTree. For example: key-expression `test/demo/a` will match if the KeTree contains any of these values: `test/demo/a` , `test/demo/**` , `test/demo/a/b`. Since explicit *Deny* is given preference over explicit *Allow*, this means that if the incoming request on `test/demo/a` is denied if any of  `test/demo/a` , `test/demo/**` or `test/demo/a/b` are present in the *Deny* list of rules specified in the ACL.