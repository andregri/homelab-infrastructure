# Multi node lab

## acloudguru specifications
Create 3 servers on acloudguru with the following specifications:
- OS: Ubuntu 22.04 - Jammy Jellyfish
- size: 3
- regione: Europe

## prepare env variables
Export env vars:
```
export NODE1=3.8.236.195
export NODE2=18.133.238.248
export NODE3=3.8.123.132
export ANSIBLE_HOST_KEY_CHECKING=False
```

### login into each server and change the temporary password
```
ssh cloud_user@$NODE1
ssh cloud_user@$NODE2
ssh cloud_user@$NODE3
```

### cd into vm/ folder
```
cd vm
```

### create the SSH key pair
```
ssh-keygen -f key
```

### copy the key to the server (it will ask the password):
```
ssh-copy-id -i ./key.pub cloud_user@${NODE1}
ssh-copy-id -i ./key.pub cloud_user@${NODE2}
ssh-copy-id -i ./key.pub cloud_user@${NODE3}
```

To login to the machines:
```
ssh -i ./key cloud_user@${NODE1}
ssh -i ./key cloud_user@${NODE2}
ssh -i ./key cloud_user@${NODE3}
```

### test the connection with the servers
```
ansible -m ping all -i inventory-multi-node.yaml
```