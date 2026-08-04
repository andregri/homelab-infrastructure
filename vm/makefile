create-role:
	ansible-galaxy init roles/${ROLE}

create-key:
	ssh-keygen -f key

copy-key:
	ssh-copy-id -i ./key.pub cloud_user@${HOSTNAME}

ping:
	ansible -m ping all -i inventory.yaml

ssh:
	ssh -i ./key cloud_user@${HOSTNAME}

install-minikube:
	ansible-playbook install-minikube.yaml -i inventory.yaml --ask-become-pass

install-grafana:
	ansible-playbook install-grafana-stack.yaml -i inventory.yaml --ask-become-pass

install-cluster-api-operator:
	ansible-playbook install-cluster-api-operator.yaml -i inventory.yaml --ask-become-pass