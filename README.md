# ROSA with LDAP

The target LDAP directory for this example is based on AWS Simple AD without TLS. It was configured with a directory DNS name of corp.example.com along with a private hosted zone in Route 53 (example.com) and an A record (corp.example.com) that is mapped to two directory DNS addresses that Simple AD provisions in the ROSA VPC.

Use OCM to configure the LDAP Identity Provider integration for ROSA. The generated configuration is stored in the openshift-authentication namespace as an OAuth object shown below for reference purposes. Note that the preferredUsername setting was changed from from uid to sAMAccountName and the format for the bindDN includes the NetBIOS name of the domain as a prefix (alternatively enter Adminstrator@corp.example.com for the binDN if you so prefer).

Do NOT apply this configuration manually (the only supported option for configuring an Identity Provider for ROSA is via OCM).

	apiVersion: v1
	items:
	- apiVersion: config.openshift.io/v1
	  kind: OAuth
	  metadata:
	    annotations:
	      include.release.openshift.io/ibm-cloud-managed: "true"
	      include.release.openshift.io/self-managed-high-availability: "true"
	      include.release.openshift.io/single-node-developer: "true"
	      release.openshift.io/create-only: "true"
	    name: cluster
	  spec:
	    identityProviders:
	    - htpasswd:
		fileData:
		  name: htpasswd-secret
	      mappingMethod: claim
	      name: Cluster-Admin
	      type: HTPasswd
	    - ldap:
		attributes:
		  email:
		  - mail
		  id:
		  - dn
		  name:
		  - cn
		  preferredUsername:
		  - sAMAccountName
		bindDN: CORP\Administrator
		bindPassword:
		  name: idp-bind-password-1om8usdpgkn7l6it6ft9lv89saf0l64q
		ca:
		  name: ""
		insecure: true
		url: ldap://corp.example.com/DC=corp,DC=example,DC=com?sAMAccountName?sub
	      mappingMethod: claim
	      name: SimpleAD
	      type: LDAP
	    templates:
	      error:
		name: rosa-oauth-templates-errors
	      login:
		name: rosa-oauth-templates-login
	      providerSelection:
		name: rosa-oauth-templates-providers
	    tokenConfig:
	      accessTokenMaxAgeSeconds: 0
	kind: List
	metadata:
	  resourceVersion: ""
	  selfLink: ""

To populate Simple AD with a user carefully follow the instructions in the AWS documentation. This will require setting a Windows Server 2019 instance.

https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_simple_ad.html

With this configuration in place and assuming a user entry has been created in Simple AD, it is possible to verify the user password via the following instructions:

https://itpro-tips.com/2019/test-ad-authentication-via-powershell/

If password verification was sucessful the next step is to attempt this from the OpenShift console. Use the sAMAccountName entry for the user.

To assign this user to the cluster-admins group this can be done via OCM. The effect of all of this can be validated using the following:

	oc get users
	oc get groups
	oc get identities
	
To troubleshoot login issue check each of the logs of the oauth-openshift pods in the openshift-authentication namespace. Note that any changes made to the identity provider configuration via OCM result in pods being restarted and this may take a few minutes.

TBD: ROSA integration using LDAP with TLS enabled.

Synchronising of users and groups was achieved using the Group Sync Operator from RedHat Communities of Practice. Instructions for installing it can be found here:

https://github.com/redhat-cop/group-sync-operator

Other than creating the secret (remember to prefix the user name with either the NetBIOS name or use the email format) the critical piece of configuration that needs to be configured is the GroupSync custom resource in the group-sync-operator namespace.

	apiVersion: redhatcop.redhat.io/v1alpha1
	kind: GroupSync
	metadata:
	  name: ldap-groupsync
	spec:
	  providers:
	  - ldap:
	      credentialsSecret:
		name: ldap-group-sync
		namespace: group-sync-operator
	      insecure: true
	      groupUIDNameMapping:
		"CN=OpenShift,CN=Users,DC=corp,DC=example,DC=com" : openshift-users
	      activeDirectory:
		usersQuery:
		  baseDN: "DC=corp,DC=example,DC=com"
		  derefAliases: never
		  filter: (&(objectClass=person)(memberOf=CN=OpenShift,CN=Users,DC=corp,DC=example,DC=com))
		  scope: sub
		userNameAttributes: [ sAMAccountName ]
		groupMembershipAttributes: [ memberOf ]
	      url: ldap://corp.example.com:389
	    name: SimpleAD

This listing assumes that only users defined in the group CN=OpenShift,CN=Users,DC=corp,DC=example,DC=com should be synchronised. Note that SimpleAD stores both users and groups in CN=Users. In a real-world directory it is expected that these would be stored somewhere separate (e.g., CN=Groups or OU=Groups).

In either case use commands like this to figure out the correct filter format for user groups:

ldapsearch -x -H ldap://corp.example.com -D "CORP\Administrator" -b "DC=corp,DC=example,DC=com" -W '(&(objectClass=person)(memberOf=CN=OpenShift,CN=Users,DC=corp,DC=example,DC=com))'

Verify the result of the group sync operation using the following commands:

	oc get groups
	oc describe group/openshift-users
