# lab-devspaces-ansible-exercise4

ansible-builder build -t quay.io/adelahoz/ee_aap_2.5_filetree_dispatch:latest --build-arg ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_CERTIFIED_TOKEN="<YOUR_SECRET_TOKEN_HERE>"
podman push quay.io/adelahoz/ee_aap_2.5_filetree_dispatch:latest