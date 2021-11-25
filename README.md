# ROSA with LDAP

The target LDAP directory for this example is based on AWS Simple AD without TLS. It was configured with a directory DNS name of corp.example.com (corresponding NetBIOS name is corp) along with a private hosted zone in Route 53 (example.com) and an A record (corp.example.com) that is mapped to two directory DNS addresses that Simple AD generates in the ROSA private subnets.

Use OCM to configure the LDAP Identity Provider integration for ROSA. The generated configuration is stored in the openshift-authentication namespace as an OAuth object shown below. Note the preferredUsername setting (changed from uid to sAMAccountName) as well as the bindDN (prefixed with the NetBIOS name).

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
		bindDN: corp\Administrator
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

With this configuration in place and assuming a user entry has been created within Simple AD and password verfication confirmed from a Windows Server 2019 instance as per:

https://itpro-tips.com/2019/test-ad-authentication-via-powershell/

Then it should be possible to login using the sAMAccountName (e.g., johndoe) via the OpenShift console.

To assign this user to the cluster-admins group this can be done via OCM. The effect of all of this can be validated using the following:

	oc get users
	oc get groups
	oc get identities
	
To troubleshoot login issue check each of the logs of the oauth-openshift pods in the openshift-authentication namespace.
