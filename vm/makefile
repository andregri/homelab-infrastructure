create-key:
	ssh-keygen -f acg

copy-key:
	ssh-copy-id -i ./acg.pub cloud_user@${SERVER_IP}

ping:
	ansible -m ping all -i inventory.yaml

install:
	ansible-playbook install.yaml -i inventory.yaml --ask-become-pass

install-grafana:
	ansible-playbook install-grafana-stack.yaml -i inventory.yaml --ask-become-pass