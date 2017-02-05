A full playbook directory structure is required to replicate the issue. The test case can be found on GitHub under [bertramn/ansible-role-defaults-bug](https://github.com/bertramn/ansible-role-defaults-bug.git).

In this test case is a role called `fancy` which defines a default variable `important_var` with the value of `role_path` which is specific to the execution context of each role. The variable is only ever defined once in the entire play environment.

The other role is called `pants` and for the intends of this test will only reference this variable in a debug statement.

Running the playbook the variable changes its value without any reassignment in the main play or any other role. It is also undefined in the main play context.

```
inventory/
  test
roles/
  fancy/
    defaults/
      main.yml  <-- variable defined in here and is never set to anything else
    tasks/
      main.yml  <-- the variable is correct here
  pants/
    tasks/
      main.yml  <-- the variable has the wrong value here
play.yml        <-- the variable is lost/undefined here
```

The variable `important_var` is only ever defined once in the `roles/fancy/defaults/main.yml` variables defaults section of the `fancy` role. However running the test one can see that although the variable is defined in the `fancy` role, it is set to the value of `role_path` in the `pants` role. 

Expected:
The default set in the `fancy` role would be available to any other role and the main playbook with the value of the `fancy` role context.

Actual:
The default variable is not available to the main play and is set to something else in any other role that references the `important_var` variable.

Example command line result of the actual behaviour:

```sh
$ ansible-playbook play.yml

PLAY [test] ********************************************************************

TASK [fancy : print the important variable value in the fancy role] ************
ok: [localhost] => {
    "important_var": "/home/fred/workspaces/ansible-role-defaults-bug/roles/fancy"
}

TASK [fancy : force to set the important variable] *****************************
skipping: [localhost]

TASK [pants : print the important variable value in the pants role] ************
ok: [localhost] => {
    "important_var": "/home/fred/workspaces/ansible-role-defaults-bug/roles/pants"
}

TASK [pants : check important variable was defined in fancy role] **************
fatal: [localhost]: FAILED! => {"changed": false, "failed": true, "msg": "expecting a fancy folder but not a pants value"}
...ignoring

TASK [print the important variable value in the main playbook] *****************
ok: [localhost] => {
    "important_var": "VARIABLE IS NOT DEFINED!"
}

PLAY RECAP *********************************************************************
localhost                  : ok=4    changed=0    unreachable=0    failed=0
```

The only workaround I found so far was setting the variable as a fact explicitely in the `fancy` role tasks section. Doing this will also ensure the value can be referenced in the main playbook.

```sh
$ ansible-playbook --extra-vars="workaround=true"  play.yml

PLAY [test] ********************************************************************

TASK [fancy : print the important variable value in the fancy role] ************
ok: [localhost] => {
    "important_var": "/home/fred/workspaces/ansible-role-defaults-bug/roles/fancy"
}

TASK [fancy : force to set the important variable] *****************************
ok: [localhost]

TASK [pants : print the important variable value in the pants role] ************
ok: [localhost] => {
    "important_var": "/home/fred/workspaces/ansible-role-defaults-bug/roles/fancy"
}

TASK [pants : check important variable was defined in fancy role] **************
skipping: [localhost]

TASK [print the important variable value in the main playbook] *****************
ok: [localhost] => {
    "important_var": "/home/fred/workspaces/ansible-role-defaults-bug/roles/fancy"
}

PLAY RECAP *********************************************************************
localhost                  : ok=4    changed=0    unreachable=0    failed=0
```

Bug [#14472](https://github.com/ansible/ansible/issues/14472) describes a very simiilar issue but I believe this ones is slightly different to the one discussed here.
