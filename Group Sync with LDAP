== Creating group sync

Create a new project
----
# oc new-project ldap-sync
----

Create yaml file
ldap-group-sync.yaml
[source,yaml]
----
apiVersion: v1
kind: LDAPSyncConfig
url: ldap://<ldap server address>
bindDN: ""
bindPassword: "1234@abcd"
insecure: true
rfc2307:
    groupsQuery:
        baseDN: "OU=EXAMPLE ENV OU,OU=EXAMPLE OCP OU,DC=example,DC=com"
        scope: sub
        filter: "(|(CN=GROUP-1)(CN=GROUP-2)(CN=GROUP-3))"
        derefAliases: never
        pageSize: 0
    groupUIDAttribute: dn 
    groupNameAttributes: [ cn ] 
    groupMembershipAttributes: [ member ] 
    usersQuery:
        baseDN: "DC=example,DC=com"
        scope: sub
        derefAliases: never
        pageSize: 0
    userUIDAttribute: dn 
    userNameAttributes: [ sAMAccountName ] 
    tolerateMemberNotFoundErrors: false
    tolerateMemberOutOfScopeErrors: false
----

Create generic secret

----
# oc create secret generic ldap-sync --from-file=ldap-sync.yaml=ldap-group-sync.yaml
----

Sync the service account

----
# oc create sa ldap-sync
----

Sync cluster rols for ldap

----
# oc create clusterrole ldap-group-sync \
    --verb=create,update,patch,delete,get,list \
    --resource=groups.user.openshift.io
----

Create role binding for ldap sync
----
# oc adm policy add-cluster-role-to-user ldap-group-sync -z ldap-sync -n ldap-sync
----

Create yaml file for cronjob
ldap-group-sync.yaml
[source,yaml]
----
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: ldap-group-sync
spec:
  # Format: https://en.wikipedia.org/wiki/Cron
  # schedule: '@hourly'
  schedule: '*/02 * * * *'
  suspend: false
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: ldap-sync
          restartPolicy: Never
          containers:
            - name: oc-cli
              command:
                - /bin/oc
                - adm
                - groups
                - sync
                - --sync-config=/ldap-sync/ldap-sync.yaml
                - --confirm
              image: registry.redhat.io/openshift4/ose-cli
              imagePullPolicy: Always
              volumeMounts:
              - mountPath: /ldap-sync/
                name: config
                readOnly: true
          volumes:
          - name: config
            secret:
              defaultMode: 420
              secretName: ldap-sync
----

Apply the yaml to create the cronjob
----
# oc create -f ldap-group-sync.yaml
----

== Configuring roles to the user groups

Assign roles to groups by running the following command:

----
# NOT IN USE FOR ROSA oc adm policy add-cluster-role-to-group cluster-admin <group name>
oc adm policy add-cluster-role-to-group cluster-status ocp-dev-view
oc adm policy add-cluster-role-to-group admin ocp-dev-devops
oc adm policy add-cluster-role-to-group self-provisioner ocp-dev-devops
oc adm policy add-cluster-role-to-group edit ocp-dev-dev-digital
oc adm policy add-cluster-role-to-group edit ocp-dev-qa-digital
----

== Configuring Disable self-provision

View the self-provisioners cluster role binding usage by running the following command:

----
# oc describe clusterrolebinding.rbac self-provisioners

Name:         self-provisioners
Labels:       <none>
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  self-provisioner
Subjects:
  Kind   Name                        Namespace
  ----   ----                        ---------
  Group  system:authenticated:oauth
----

Remove the self-provisioner cluster role from the group system:authenticated:oauth.

----
# oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects": null}'
----

Edit the self-provisioners cluster role binding to prevent automatic updates to the role. Automatic updates reset the cluster roles to the default state.

Run the following command:
----
# oc edit clusterrolebinding.rbac self-provisioners
----

In the displayed role binding, set the rbac.authorization.kubernetes.io/autoupdate parameter value to false, as shown in the following example:

[source,yaml]
----
apiVersion: authorization.openshift.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "false"
  ...
----

To update the role binding by using a single command:

----
# oc patch clusterrolebinding.rbac self-provisioners -p '{ "metadata": { "annotations": { "rbac.authorization.kubernetes.io/autoupdate": "false" } } }'
----

Login as an authenticated user and verify that it can no longer self-provision a project:

----
# oc new-project test
Error from server (Forbidden): You may not request a new project via this API.
----
