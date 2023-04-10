# Install MariaDB Galera Cluster in Kubernetes Cluster

This repository will show you how to install MariaDB Galera cluster in Kubernetes cluster.

## Prepare for installation

You need to prepare persistent voluem for your database cluster. To do this, follow the steps below:

1. Create directory by issuing command below

    > This guide using home directory as persistent volume target folder, you can change path to what you want.

    ```console
    $ mkdir -p ~/.production/mariadb-galera
    ```

2. Make folder for every mariadb node.

    ```console
    $ cd mariadb-galera
    $ mkdir node1 node2 node3
    ```

3. Change the owner of directory to `1001`

    > We only change the owner of `mariadb-galera` folder but the `.production` folder.

    ```console
    # chown -R 1001:1001 mariadb-galera/
    ```

## Install

1. Change kubernetes yaml contents and apply it to kubernetes cluster

    > Don't forget to change folder path in yaml file.

    ```console
    $ cp deployment.yaml deployment.production.yaml
    $ vim deployment.production.yaml
    $ kubectl apply -f deployment.production.yaml
    ```

2. Apply SELinux policy

    ```console
    # semodule -i allowregistrypolicy.pp
    ```

    > If you create your folder in other directory but not your home directory, you need to change mariadbgalerapolicy.te to meet your situation.

    1. Your SELinux type will be shown after you issue `ls <YOUR_PATH> -alZ` command, the result will be shown between owner and size.
    2. Modify `mariadbgalerapolicy.te` file, add the type to `require` section.
    3. Modify `container_t` section below, change `user_home_t` into your type.
    4. Issue the commands below:

        > If you failed to apply these command, check your te file is using `LF` but `CRLF`.

        ```console
        # checkmodule -M -m -o mariadbgalerapolicy.mod mariadbgalerapolicy.te
        # semodule_package -o mariadbgalerapolicy.pp -m mariadbgalerapolicy.mod
        # semodule -i mariadbgalerapolicy.pp
        ```

3. Install MariaDB Galera by using Helm charts created by Bitnami

    > We install this charts to `database-system` namespace in Kubernetes, you can change it to what you want.

    > Change `<YOUR_ROOT_PASSWORD>` to what you want.

    ```console
    $ helm repo add bitnami https://charts.bitnami.com/bitnami
    $ helm install mariadb-galera bitnami/mariadb-galera \
        --namespace database-system --create-namespace \
        --set rootUser.password=<YOUR_ROOT_PASSWORD>
    ```

4. Waiting for all nodes up and done. You can connect your database by port forward the 3306 port in Kubernetes cluster service.

## Reference

- [MariaDB Galera packaged by Bitnami](https://github.com/bitnami/charts/tree/main/bitnami/mariadb-galera)
