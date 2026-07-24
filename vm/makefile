create-key:
	ssh-keygen -f key

copy-key:
	ssh-copy-id -i ./key.pub cloud_user@${HOSTNAME}

ping:
	ansible -m ping all -i inventory.yaml

install:
	ansible-playbook install.yaml -i inventory.yaml --ask-become-pass

install-grafana:
	ansible-playbook install-grafana-stack.yaml -i inventory.yaml --ask-become-pass