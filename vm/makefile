create-key:
	ssh-keygen -f acg

copy-key:
	ssh-copy-id -i ./acg.pub cloud_user@${SERVER_IP}

ping:
	ansible -m ping all -i inventory.yaml

install:
	ansible-playbook docker-install.yaml -i inventory.yaml --ask-become-pass