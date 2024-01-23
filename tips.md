1. 运行 playbook 时指定 ssh key  
    `ansible-playbook reimage_host.yml --key-file ~/.ssh/id_rsa.infra`

2. 使用galaxy创建role.
    ```
    $ ansible-galaxy init role_A
    ```



