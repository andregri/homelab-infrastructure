create-key:
	ssh-keygen -f ssh/key

copy-key:
	ssh-copy-id -i ./ssh/key.pub cloud_user@${HOSTNAME}

ssh:
	ssh -i ./ssh/key cloud_user@${HOSTNAME}