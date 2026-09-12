**ansible-galaxy collection install community.crypto**


**ansible-galaxy collection install ansible.posix**

1 - sudo apt install whois -y

2 - mkpasswd --method=sha-512

3 - ansible-vault encrypt_string '$6$abc123...hash...' --name 'user_password_hash'

4 - put the result of previous command to the host_vars/vpc.yml

5 - ansible-playbook 1-ssh-access/bootstrap.yml --vault-password-file ~/.vault_pass