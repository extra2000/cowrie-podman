Production Deployment
=====================

Example how to deploy Cowrie for single-instance production environment.

Clone repositories
------------------

.. code-block:: bash

    mkdir ~/extra2000
    cd ~/extra2000
    git clone https://github.com/extra2000/cowrie-podman.git
    git clone https://github.com/extra2000/cowrie.git cowrie-podman/src/cowrie

Then, ``cd`` into project root directory:

.. code-block:: bash

    cd cowrie-podman

Build Cowrie Image
------------------

From the project root directory, ``cd`` into ``src/cowrie/`` and then build:

.. code-block:: bash

    cd src/cowrie/
    podman build -t extra2000/cowrie -f docker/Dockerfile .

Deploy Cowrie
-------------

From the project root directory, ``cd`` into ``deployment/production/cowrie``:

.. code-block:: bash

    cd deployment/production/cowrie

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/cowrie.yaml{.example,}
    cp -v configs/cowrie.cfg{.example,}
    cp -v configs/userdb.txt{.example,}
    chmod og+r ./configs/*

Create SSH keys without passphrase:

.. code-block:: bash

    ssh-keygen -f ./secrets/ssh_host_rsa_key -t rsa -b 4096
    ssh-keygen -f ./secrets/ssh_host_ed25519_key -t ed25519
    ssh-keygen -f ./secrets/ssh_host_ecdsa_key -t ecdsa -b 521
    chcon -v -t container_file_t ./secrets/*
    chmod og+r ./secrets/*

Create pod file:

.. code-block:: bash

    cp -v cowrie-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Create SELinux security policy:

.. code-block:: bash

    cp -v selinux/cowrie_podman.cil{.example,}

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/cowrie_podman.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "cowrie_podman"

Deploy Cowrie:

.. code-block:: bash

    podman play kube --configmap configmaps/cowrie.yaml --seccomp-profile-root ./seccomp cowrie-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name cowrie-pod-srv01
    systemctl --user enable container-cowrie-pod-srv01.service

Testing
-------

Try ``ssh`` login:

.. code-block:: bash

    ssh -p 22 root@127.0.0.1

After finished testing, clean up by removing the ``127.0.0.1:22`` from known host:

.. code-block:: bash

    ssh-keygen -R 127.0.0.1

Try ``scp`` a file:

.. code-block:: bash

    scp -P 22 /path/to/myfile root@127.0.0.1:

The file can be found in volume ``cowrie-var`` with path ``lib/cowrie/downloads/``. To list all uploaded files:

.. code-block:: bash

    podman run -it --rm -v cowrie-var:/opt/foo:ro docker.io/library/alpine ls /opt/foo/lib/cowrie/downloads/
