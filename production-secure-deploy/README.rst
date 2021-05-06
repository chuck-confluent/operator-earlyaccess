Production recommended secure setup
===================================

Confluent recommends these security mechanisms for a production deployment:

- Enable Kafka client authentication. Choose one of:

  - SASL/Plain or mTLS

  - For SASL/Plain, the identity can come from LDAP server

- Enable Confluent Role Based Access Control for authorization, with user/group identity coming from LDAP server

- Enable TLS for network encryption - both internal (between CP components) and external (Clients to CP components)

In this deployment scenario, we will set this up, choosing SASL/Plain for authentication, using user provided custom certificates.

==================================
Set the current tutorial directory
==================================

Set the tutorial directory for this tutorial under the directory you downloaded
the tutorial files:

::
   
  export TUTORIAL_HOME=<Tutorial directory>/production-secure-deploy
  
=========================
Deploy Confluent Operator
=========================

The assumption is that you’ve set up Early Access credentials following `the
instruction
<https://github.com/confluentinc/operator-earlyaccess/blob/master/README.rst>`__.

#. Install Confluent Operator using Helm:

   ::

     helm upgrade --install operator confluentinc_earlyaccess/confluent-for-kubernetes --set image.registry=confluent-docker-internal-early-access-operator-2.jfrog.io
  
#. Check that the Confluent Operator pod comes up and is running:

   ::
     
     kubectl get pods

===============
Deploy OpenLDAP
===============

This repo includes a Helm chart for `OpenLdap
<https://github.com/osixia/docker-openldap>`__. The chart ``values.yaml``
includes the set of principal definitions that Confluent Platform needs for
RBAC.

#. Deploy OpenLdap

   ::

     helm upgrade --install -f $TUTORIAL_HOME/../assets/openldap/ldaps-rbac.yaml test-ldap $TUTORIAL_HOME/../assets/openldap --namespace confluent

#. Validate that OpenLDAP is running:  
   
   ::

     kubectl get pods

#. Log in to the LDAP pod:

   ::

     kubectl exec -it ldap-0 -- bash

#. Run the LDAP search command:

   ::

     ldapsearch -LLL -x -H ldap://ldap.confluent.svc.cluster.local:389 -b 'dc=test,dc=com' -D "cn=mds,dc=test,dc=com" -w 'Developer!'

#. Exit out of the LDAP pod:

   ::
   
     exit 
     
============================
Deploy configuration secrets
============================

You'll use Kubernetes secrets to provide credential configurations.

With Kubernetes secrets, credential management (defining, configuring, updating)
can be done outside of the Confluent Operator. You define the configuration
secret, and then tell Confluent Operator where to find the configuration.
   
To support the above deployment scenario, you need to provide the following
credentials:

* Component TLS Certificates

* Authentication credentials for Zookeeper, Kafka, Control Center, remaining CP components

* RBAC principal credentials
  
You can either provide your own certificates, or generate test certificates. Follow instructions
in the below "Appendix: Create your own certificates" section to see how to generate certificates
and set the appropriate SANs. 

Provide component TLS certificates
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

::
   
     kubectl create secret generic tls-group1 \
   --from-file=fullchain.pem=$TUTORIAL_HOME/../assets/certs/generated/server.pem \
   --from-file=cacerts.pem=$TUTORIAL_HOME/../assets/certs/generated/ca.pem \
      --from-file=privkey.pem=$TUTORIAL_HOME/../assets/certs/generated/server-key.pem


Provide authentication credentials
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Create a Kubernetes secret object for Zookeeper, Kafka, and Control Center.

   This secret object contains file based properties. These files are in the
   format that each respective Confluent component requires for authentication
   credentials.

   ::
   
     kubectl create secret generic credential \
      --from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
      --from-file=digest-users.json=$TUTORIAL_HOME/creds-zookeeper-sasl-digest-users.json \
      --from-file=digest.txt=$TUTORIAL_HOME/creds-kafka-zookeeper-credentials.txt \
      --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
      --from-file=basic.txt=$TUTORIAL_HOME/creds-control-center-users.txt \
      --from-file=ldap.txt=$TUTORIAL_HOME/ldap.txt

   In this tutorial, we use one credential for authenticating all client and
   server communication to Kafka brokers. In production scenarios, you'll want
   to specify different credentials for each of them.

Provide RBAC principal credentials
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Create a Kubernetes secret object for MDS:

   ::
   
     kubectl create secret generic mds-token \
       --from-file=mdsPublicKey.pem=$TUTORIAL_HOME/../assets/certs/mds-publickey.txt \
       --from-file=mdsTokenKeyPair.pem=$TUTORIAL_HOME/../assets/certs/mds-tokenkeypair.txt
   
   ::
   
     # Kafka RBAC credential
     kubectl create secret generic mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/bearer.txt
     # Control Center RBAC credential
     kubectl create secret generic c3-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/c3-mds-client.txt
     # Connect RBAC credential
     kubectl create secret generic connect-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/connect-mds-client.txt
     # Schema Registry RBAC credential
     kubectl create secret generic sr-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/sr-mds-client.txt
     # ksqlDB RBAC credential
     kubectl create secret generic ksqldb-mds-client \
       --from-file=bearer.txt=$TUTORIAL_HOME/ksqldb-mds-client.txt
     # Kafka REST credential
     kubectl create secret generic rest-credential \
       --from-file=bearer.txt=$TUTORIAL_HOME/bearer.txt \
       --from-file=basic.txt=$TUTORIAL_HOME/bearer.txt

=========================
Deploy Confluent Platform
=========================

#. Deploy Confluent Platform:

   ::

     kubectl apply -f $TUTORIAL_HOME/confluent-platform-production.yaml

#. Check that all Confluent Platform resources are deployed:

   ::
   
     kubectl get pods

Note: The default required RoleBindings for each Confluent component are created
automatically, and maintained as `confluentrolebinding` custom resources.

   ::

     kubectl get confluentrolebinding
   
     

=================================================
Create RBAC Rolebindings for Control Center admin
=================================================

Create Control Center Role Binding for a Control Center ``testadmin`` user.

   ::

     kubectl apply -f $TUTORIAL_HOME/controlcenter-testadmin-rolebindings.yaml

========
Validate
========

Validate in Control Center
^^^^^^^^^^^^^^^^^^^^^^^^^^

Use Control Center to monitor the Confluent Platform, and see the created topic
and data. You can visit the external URL you set up for Control Center, or visit the URL
through a local port forwarding like below:

#. Set up port forwarding to Control Center web UI from local machine:

   ::

     kubectl port-forward controlcenter-0 9021:9021

#. Browse to Control Center. You will log in as the ``testadmin`` user, with ``testadmin`` password.

   ::
   
     https://localhost:9021

The ``testadmin`` user (``testadmin`` password) has the ``SystemAdmin`` role granted and will have access to the
cluster and broker information.
  

======================================
Appendix: Create your own certificates
======================================

When testing, it's often helpful to generate your own certificates to validate the architecture and deployment.

You'll want both these to be represented in the certificate SAN:

- external domain names
- internal Kubernetes domain names

The internal Kubernetes domain name depends on the namespace you deploy to. If you deploy to `confluent` namespace, 
then the internal domain names will be: 

- *.kafka.confluent.svc.cluster.local
- *.zookeeper.confluent.svc.cluster.local
- *.confluent.svc.cluster.local

::

  # Install libraries on Mac OS
  brew install cfssl

::
  
  # Create Certificate Authority
  cfssl gencert -initca $TUTORIAL_HOME/../assets/certs/ca-csr.json | cfssljson -bare $TUTORIAL_HOME/../assets/certs/generated/ca -

::

  # Validate Certificate Authority
  openssl x509 -in $TUTORIAL_HOME/../assets/certs/generated/ca.pem -text -noout

::

  # Create server certificates with the appropriate SANs (SANs listed in server-domain.json)
  cfssl gencert -ca=$TUTORIAL_HOME/../assets/certs/generated/ca.pem \
  -ca-key=$TUTORIAL_HOME/../assets/certs/generated/ca-key.pem \
  -config=$TUTORIAL_HOME/../assets/certs/ca-config.json \
  -profile=server $TUTORIAL_HOME/../assets/certs/server-domain.json | cfssljson -bare $TUTORIAL_HOME/../assets/certs/generated/server

  # Validate server certificate and SANs
  openssl x509 -in $TUTORIAL_HOME/../assets/certs/generated/server.pem -text -noout

=====================================
Appendix: Update authentication users
=====================================

In order to add users to the authenticated users list, you'll need to update the list in the following files:

- For Kafka users, update the list in ``creds-kafka-sasl-users.json``.
- For Control Center users, update the list in ``creds-control-center-users.txt``.

After updating the list of users, you'll update the Kubernetes secret.

::

  kubectl create secret generic credential \
      --from-file=plain-users.json=$TUTORIAL_HOME/creds-kafka-sasl-users.json \
      --from-file=digest-users.json=$TUTORIAL_HOME/creds-zookeeper-sasl-digest-users.json \
      --from-file=digest.txt=$TUTORIAL_HOME/creds-kafka-zookeeper-credentials.txt \
      --from-file=plain.txt=$TUTORIAL_HOME/creds-client-kafka-sasl-user.txt \
      --from-file=basic.txt=$TUTORIAL_HOME/creds-control-center-users.txt \
      --from-file=ldap.txt=$TUTORIAL_HOME/ldap.txt \ 
      --save-config --dry-run=client -oyaml | k apply -f -

In this above CLI command, you are generating the YAML for the secret, and applying it as an update to the existing secret ``credential``.

There's no need to restart the Kafka brokers or Control Center. The updates users list is picked up by the services.

=======================================
Appendix: Configure mTLS authentication
=======================================

Kafka supports mutual TLS (mTLS) authentication for client applications. With mTLS, principals are taken from the 
Common Name of the certificate used by the client application.

This example deployment spec ($TUTORIAL_HOME/confluent-platform-production-mtls.yaml) configures the Kafka external listener 
for mTLS authentication.

When using mTLS, you'll need to provide a different certificate for each component, so that each component
has the principal in the Common Name. In the example deployment spec, each component refers to a different
TLS certificate secret.

=========================
Appendix: Troubleshooting
=========================

Gather data
^^^^^^^^^^^

::

  # Check for any error messages in events
  kubectl get events -n confluent

  # Check for any pod failures
  kubectl get pods

  # For pod failures, check logs
  kubectl logs <pod-name>
