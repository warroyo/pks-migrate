# Migrate PKS to a new VSphere Cluster

this doc describes how to migrate a pks installation and any other bosh managed vms to a new vsphere cluster. 

**Warning: migrating clusters can cuase issues if the new cluster does not have the same network and storage attached, be sure to validate before doing this.you can vmotion a test vm to validate** 

**Note:** if you are planning on delete the old cluster afterward make sure you move the opsmanager over to the new cluster using vmotion. 

## Setup 

* download the om [om cli](https://github.com/pivotal-cf/om/releases)
* set OM cli env variables
  
```
export OM_TARGET=<opsman fqdn>
export OM_USERNAME=admin
export OM_SKIP_SSL_VALIDATION=true
export OM_PASSWORD=<password>
```

## Add your new cluster

first we need to add our new cluster(s) to the AZs and apply changes so that bosh is aware of the new clusters.

1. login to opsman and go into the director tile
2. click on "Create Availability Zones" 
3. for each AZ that you want to change the underlying vsphere cluster for add a new cluster to that az by clicking the "add cluster" button and filling in your details.
4. Apply changes.


## Put the opsman into Advanced Mode

We need to use the api to put opsman into advanced mode. the easiest way to do this is to use the om cli and use `om curl` . 

if you have not done the setup above do that now.


1. run the `om curl` command to change into advanced mode.

```bash
om curl -p /api/v0/staged/infrastructure/locked -x PUT --data '{"locked" : "false"}' -H "Content-Type: application/json"
```

2. check the status, it should be false

```bash
om curl -p /api/v0/staged/infrastructure/locked
```

## Update the AZ(s) to remove the old cluster

this can only be done via the api when in advanced mode. 

1. get a list of all of azs and clusters

```bash
om curl -p /api/v0/staged/director/availability_zones
```

you should recieve an output that looks something like this. 


```json
{
  "availability_zones": [
    {
      "name": "az1",
      "guid": "b64536b7d146fb18042e",
      "iaas_configuration_guid": "c7d392865fad1fa8168b",
      "clusters": [
        {
          "guid": "23a52683cec52225ff05",
          "cluster": "old-cluster",
          "resource_pool": "old",
          "host_group": null,
          "drs_rule": "MUST"
        },
        {
          "guid": "53a5283cef52226ft05",
          "cluster": "new-cluster",
          "resource_pool": "new",
          "host_group": null,
          "drs_rule": "MUST"
        }
      ]
    }
  ]
}
```

2. modify the json that was output to remove the old cluster from the list

**be sure to keep the GUID the same on the az and cluster**

you should end up with something like this:

```json
{
  "availability_zones": [
    {
      "name": "az1",
      "guid": "b64536b7d146fb18042e",
      "iaas_configuration_guid": "c7d392865fad1fa8168b",
      "clusters": [
        {
          "guid": "53a5283cef52226ft05",
          "cluster": "new-cluster",
          "resource_pool": "new",
          "host_group": null,
          "drs_rule": "MUST"
        }
      ]
    }
  ]
}
```

3. update opsman with the api to overwrite the azs

you will need to replace the json after the `-d` below with your modified json from the previous step.

```bash
om curl -p "/api/v0/staged/director/availability_zones" \
    -x PUT \
    -H "Content-Type: application/json" \
    -d '{
  "availability_zones": [
    {
      "name": "az1",
      "guid": "b64536b7d146fb18042e",
      "iaas_configuration_guid": "c7d392865fad1fa8168b",
      "clusters": [
        {
          "guid": "53a5283cef52226ft05",
          "cluster": "new-cluster",
          "resource_pool": "new",
          "host_group": null,
          "drs_rule": "MUST"
        }
      ]
    }
  ]
}'
```

## apply the changes

1. go into the director tile under "director config" and check the "recreate all VMs" box. we need to do this so we are certain bosh will move everything"
2. if you are using PKS be sure the "upgrade all clusters" errand is checked. 
3. Click "apply changes"



at this point all the VMs should start recreating in the new cluster.