# OpenStack control plane services deployment

## Prerequisites

* Previous Adoption steps completed. Notably, the service databases
  must already be imported into the podified MariaDB.

## Variables

Define the shell variables used in the steps below. The values are
just illustrative, use values which are correct for your environment:

```
ADMIN_PASSWORD=SomePassword
KEYSTONE_DATABASE_PASSWORD=SomePassword
```

## Pre-checks

## Procedure - OpenStack control plane services deployment

* Set Keystone password and database user creation password to match
  the original deployment:

  ```
  oc set data secret/osp-secret "AdminPassword=$ADMIN_PASSWORD"
  oc set data secret/osp-secret "DatabasePassword=$KEYSTONE_DATABASE_PASSWORD"
  oc set data secret/osp-secret "KeystoneDatabasePassword=$KEYSTONE_DATABASE_PASSWORD"
  ```

  > Note: The `DatabasePassword` is currently common, affects creation
  > of all database users. This should be fixed in podified control
  > plane after the MariaDB Operator is replaced with Galera Operator.

* Patch OpenStackControlPlane to deploy Keystone:

  ```
  oc patch openstackcontrolplane openstack --type=merge --patch '
  spec:
    keystone:
      enabled: true
      template:
        secret: osp-secret
        containerImage: quay.io/tripleozedcentos9/openstack-keystone:current-tripleo
        databaseInstance: openstack
  '
  ```

## Post-checks

* Test that `openstack user list` works.

  > Note: This used to work, but after recent changes to endpoint
  > management mechanism in the Keystone operator, the operator
  > actually removes all existing endpoints. This needs to be
  > addressed further.

  ```
  cat > clouds-adopted.yaml <<EOF
  clouds:
    adopted:
      auth:
        auth_url: http://keystone-public-openstack.apps-crc.testing
        password: $ADMIN_PASSWORD
        project_domain_name: Default
        project_name: admin
        user_domain_name: Default
        username: admin
      cacert: ''
      identity_api_version: '3'
      region_name: regionOne
      volume_api_version: '3'
  EOF

  export OS_CLIENT_CONFIG_FILE=clouds-adopted.yaml
  export OS_CLOUD=adopted

  openstack user list
  ```
