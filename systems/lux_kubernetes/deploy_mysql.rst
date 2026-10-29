..
   DRAFT for contribution to the OLCF Lux documentation, as a use-case page under the
   Lux Kubernetes User Guide. Structured after the SLATE "Deploy MongoDB" page
   (services_and_applications/slate/use_cases/mongodb_service.rst).

   Sections marked TODO(verify) depend on questions that cannot be answered without a
   deployment on Lux. Do not submit this upstream until they are resolved -- see
   lux/README.md, "Open questions for User Assistance".

.. _lux-kube-deploy-mysql:

***************
Deploying MySQL
***************

This page walks through deploying a single-instance MySQL 8.0 server on Lux Kubernetes, backed by
persistent storage, and connecting to it from a client. It is suitable for a workflow that needs a
database to coordinate tasks or record results. It is **not** a highly available configuration:
there is one replica, no replication, and no automatic failover.

Prerequisites
=============

* A project namespace on Lux Kubernetes.
* ``kubectl`` configured with a kubeconfig copied from the Rancher dashboard. See
  :ref:`lux-kube-using-kubectl`. Note that these credentials expire every 24 hours, and that
  ``kubectl`` must be run from ``home.ccs.ornl.gov``.
* Optionally ``helm``, if you prefer to deploy the chart rather than the raw manifests.

Set your default namespace so the commands below do not each need ``-n``:

.. code-block:: bash

   kubectl config set-context --current --namespace=<proj>

Deploying MySQL
===============

MySQL is deployed as a StatefulSet rather than a Deployment. A StatefulSet gives the pod a stable
name (``mysql-0``), and a PersistentVolumeClaim that is bound to that
pod and survives restarts.

1. Create the Secret
--------------------

Credentials are passed at creation time rather than committed to a file. ``stringData`` lets you
supply plain text; Kubernetes stores the values base64-encoded.

.. code-block:: bash

   kubectl create secret generic mysql-secret \
     --from-literal=MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PW" \
     --from-literal=MYSQL_DATABASE=pocdb \
     --from-literal=MYSQL_USER=pocuser \
     --from-literal=MYSQL_PASSWORD="$MYSQL_USER_PW"

.. note::

   Kubernetes Secrets are encoded, not encrypted, and anyone with read access to the namespace can
   read them. TODO(verify): whether Secrets are encrypted at rest on Lux, and what RBAC applies within
   a project namespace.

2. Create the ConfigMap
-----------------------

Non-secret server settings go in a ConfigMap, so they can be changed without touching credentials:

.. code-block:: yaml

   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: mysql-config
   data:
     MYSQL_CHARSET: "utf8mb4"
     MYSQL_COLLATION: "utf8mb4_unicode_ci"
     MYSQL_MAX_CONNECTIONS: "100"

.. code-block:: bash

   kubectl apply -f mysql-configmap.yaml

3. Create the StatefulSet
-------------------------

.. code-block:: yaml

   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: mysql
   spec:
     serviceName: mysql
     replicas: 1
     selector:
       matchLabels:
         app: mysql
     template:
       metadata:
         labels:
           app: mysql
       spec:
         securityContext:
           fsGroup: 0
         containers:
           - name: mysql
             image: quay.io/sclorg/mysql-80-c9s:latest
             ports:
               - containerPort: 3306
                 name: mysql
             envFrom:
               - secretRef:
                   name: mysql-secret
               - configMapRef:
                   name: mysql-config
             volumeMounts:
               - name: mysql-data
                 mountPath: /var/lib/mysql/data
             resources:
               requests:
                 cpu: "250m"
                 memory: "512Mi"
               limits:
                 cpu: "1"
                 memory: "1Gi"
             livenessProbe:
               exec:
                 command: ["sh", "-c", "mysqladmin ping -h 127.0.0.1 -u root -p\"$MYSQL_ROOT_PASSWORD\""]
               initialDelaySeconds: 30
               periodSeconds: 10
               timeoutSeconds: 5
             readinessProbe:
               exec:
                 command: ["sh", "-c", "mysql -h 127.0.0.1 -u root -p\"$MYSQL_ROOT_PASSWORD\" -e \"SELECT 1\""]
               initialDelaySeconds: 15
               periodSeconds: 5
               timeoutSeconds: 5
     volumeClaimTemplates:
       - metadata:
           name: mysql-data
         spec:
           accessModes: ["ReadWriteOnce"]
           storageClassName: netapp-block
           resources:
             requests:
               storage: 2Gi

.. code-block:: bash

   kubectl apply -f mysql-statefulset.yaml
   kubectl get pods -w

Three details are worth calling out:

* ``storageClassName: netapp-block``: Lux offers ``netapp-file`` (the default) and ``netapp-block``.
  Use ``netapp-block`` for a database: it provides a block device rather than an NFS share, which is
  what InnoDB's locking and ``fsync`` behaviour expects. See :ref:`lux-kube-storage`.
* ``securityContext.fsGroup: 0``: The ``sclorg`` image is built for an arbitrary-UID model and
  relies on group permissions on the data directory. TODO(verify) whether this is required on Lux
  or whether the CSI driver already sets ownership appropriately.
* **The probes use the root password from the environment**, so it is never written into the
  manifest. The readiness probe gates the Service: traffic is only sent to the pod once MySQL
  answers a query.

.. note::

   Lux Kubernetes nodes do not have direct internet access. TODO(verify) whether ``quay.io`` is
   reachable for image pulls. If it is not, mirror the image to Docker Hub and attach an
   ``imagePullSecrets`` entry - see :ref:`pull-secret-kubernetes`.

4. Create the Service
---------------------

A headless Service (``clusterIP: None``) gives the pod a stable DNS name within the cluster. Other
pods in the namespace reach MySQL at ``mysql:3306``, or at
``mysql.<proj>.svc.cluster.local:3306`` from another namespace.

.. code-block:: yaml

   apiVersion: v1
   kind: Service
   metadata:
     name: mysql
     labels:
       app: mysql
   spec:
     clusterIP: None
     selector:
       app: mysql
     ports:
       - port: 3306
         targetPort: 3306
         name: mysql

.. code-block:: bash

   kubectl apply -f mysql-service.yaml

5. Verify
---------

.. code-block:: bash

   kubectl exec -it mysql-0 -- sh -c 'mysql -u pocuser -p"$MYSQL_PASSWORD" pocdb -e "SELECT VERSION();"'

Confirm the storage is actually persistent by writing a row, deleting the pod, and reading it back
after the StatefulSet recreates it:

.. code-block:: bash

   kubectl exec -i mysql-0 -- sh -c 'mysql -u pocuser -p"$MYSQL_PASSWORD" pocdb' <<'SQL'
   CREATE TABLE persistence_test (id INT AUTO_INCREMENT PRIMARY KEY, note VARCHAR(64));
   INSERT INTO persistence_test (note) VALUES ('written before pod deletion');
   SQL

   kubectl delete pod mysql-0
   kubectl wait --for=condition=ready pod/mysql-0 --timeout=300s
   kubectl exec -it mysql-0 -- sh -c 'mysql -u pocuser -p"$MYSQL_PASSWORD" pocdb -e "SELECT * FROM persistence_test;"'

Accessing MySQL from outside the cluster
========================================

Port forwarding
---------------

For development and testing, forward a local port to the Service. This works for any TCP protocol:

.. code-block:: bash

   kubectl port-forward svc/mysql 3307:3306

Then, from a second terminal on the same host:

.. code-block:: bash

   mysql -h 127.0.0.1 -P 3307 -u pocuser -p pocdb

The forward lasts as long as the command runs, and is bound by the 24-hour lifetime of your
kubeconfig credentials.

A durable endpoint
------------------

.. warning::

   TODO(verify). :ref:`HTTPRoute <lux-kube-httproutes>`, the exposure mechanism documented for Lux,
   routes HTTP by host and path and cannot carry the MySQL wire protocol. This section needs the
   supported answer for a long-lived non-HTTP TCP service - ``TCPRoute`` on the Istio gateway, a
   NodePort, a ``LoadBalancer`` Service, or a recommendation to keep such services in-cluster only.
   Until then, clients should run as pods in the same namespace and connect to ``mysql:3306``.

Whatever mechanism is used, restrict the source ranges that can reach port 3306 rather than
accepting any client, and remember that the proof of concept relies on the image's self-signed TLS certificate
with no certificate management.

Teardown
========

.. code-block:: bash

   kubectl delete statefulset mysql
   kubectl delete service mysql
   kubectl delete configmap mysql-config
   kubectl delete secret mysql-secret

Deleting a StatefulSet does **not** delete the PersistentVolumeClaims it created. Remove the data
explicitly once you are sure you no longer need it:

.. code-block:: bash

   kubectl get pvc
   kubectl delete pvc mysql-data-mysql-0

Limitations
===========

* Single instance, no replication, no automatic failover. A node failure or eviction means downtime
  for as long as the pod takes to reschedule and recover.
* TLS is the image's self-signed default; no certificates are managed or enforced.
* The image tag is unpinned. Pin a digest for anything beyond a proof of concept.
* Backups are not configured.
