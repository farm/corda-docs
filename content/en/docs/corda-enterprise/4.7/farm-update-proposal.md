NOTE: Please take my snippets as a general direction only, I know you can phrase them better :)

# Suggested changes 

## Create a central place for remote management docs

Move the Gateway, Auth, and SSO pages to the Corda Enterprise docs:
	- Gateway Svc
	- Auth Svc
	- SSO support (https://seel.corda.net/docs/cenm/1.5/azure-ad-sso.html)

### Gateway Svc page changes

1) Replace
```
The Gateway Service provides a transfer layer between front-end Corda Enterprise Network Manager (CENM) interfaces, and the Auth Service that underpins authentication and authorisation in CENM.

Once installed and configured, users can connect with the Gateway Service via the CENM CLI Tool to manage CENM service tasks. Administrators can use the Gateway Service address plus /admin to access the (CENM User Admin Tool)[user-admin] via a web browser.
```
with a product-agnostic version:
```
The Gateway Service acts as common entry point for remote management of Corda Nodes as well as networks using CENM - either using the available commandline tools or via the web applications hosted by the Gateway.
```

2) Replace the following section in the config example:
```
# CENM zone-service address
cenm {
    zoneHost: "zone-service"
    # Admin listener port of the zone service
    zonePort: 5063
}
```

with a placeholder:
```
# application-specific configuration should go here
```

Add a new section:
```
#### Installing applications onto the Gateway Service
Auth service needs to be set up with baseline permission data for each application.

* CENM <-- link this to the installation section of https://seel.corda.net/docs/cenm/1.5/cenm-console.html
* Node Management Console <-- link this to the installation section of https://seel.corda.net/docs/corda-enterprise/4.7/node/management-console.html
* Flow Management <-- link this to the installation section of https://seel.corda.net/docs/corda-enterprise/4.7/node/node-flow-management-console.html
```


### Auth svc page changes

Rework the first section to be generic:

```
The Auth service is the user authentication and authorization service for managing Corda Nodes and networks (CENM). It stores and controls secure user-access to network services, such as:

* Nodes
* Identity manager
* Zone service
* Signing service
* Network map (and associated network configurations and node info)

Whenever you use the [User admin tool](user-admin) to create new users, groups or roles, the Auth service is updated to authenticate those users and their permissions. When using the remote management tools such as the [CENM Command Line Interface](cenm-cli-tool) or the web GUIs hosted on the Gateway Service, the auth service verifies your identity and security clearance as needed.

You do not need to interact directly with the Auth Service once it has been installed and configured. To protect the integrity of this secure service, there is no direct API contact with the Auth Service - all front-end communications go via the Gateway service.
```

Add a link to the SSO page (https://seel.corda.net/docs/cenm/1.5/azure-ad-sso.html)

Add a section:
```
## Setting up applications
Auth service needs to be set up with baseline permission data for each application.

* CENM <-- link this to the installation section of https://seel.corda.net/docs/cenm/1.5/cenm-console.html
* Node Management Console <-- link this to the installation section of https://seel.corda.net/docs/corda-enterprise/4.7/node/management-console.html
* Flow Management Console <-- link this to the installation section of https://seel.corda.net/docs/corda-enterprise/4.7/node/node-flow-management-console.html
```

### Node MGMT console
Based on https://seel.corda.net/docs/corda-enterprise/4.7/node/management-console.html

Just before the Configuration section, reword the installation instructions and add a new heading so that the section can be linked from the Auth and Gateway pages.

Change 
```
node-management-plugin-<release>.jar should be put into the plugins directory in the Gateway service, and auth-baseline-node-management-<release>.jar should be put into the plugins directory in the Auth service.
```
to
```
#### Installing the Node Management Console

1) place node-management-plugin-<release>.jar into the plugins directory in the Gateway service, 
2) place auth-baseline-node-management-<release>.jar into the plugins directory in the Auth service.
```

### Flow Management Console
Based on https://seel.corda.net/docs/corda-enterprise/4.7/node/node-flow-management-console.html

Same as the Node Management Console, create a new heading with the installation instructions.

### CENM Management Console

Add another item to the Requirements section:
* Install the Gateway Service (link this to the new gateway page)

Remove the gateway installation steps from the installation instructions 

Reword step 6 of the installation instructions:
```Launch the Gateway Service.``` to ```Launch or restart the Gateway Service.```




