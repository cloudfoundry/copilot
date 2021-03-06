# Copilot

To help Istio Pilot work with Cloud Foundry

You probably want to deploy this using [Istio Release](https://github.com/cloudfoundry/istio-release).

## Development

To get started:

```sh
git clone https://github.com/cloudfoundry/copilot.git
cd copilot
go get github.com/onsi/ginkgo/ginkgo
go get github.com/golang/dep/cmd/dep
dep ensure
```

To run the tests:

```sh
ginkgo -r -p -race
```

To compile the server:

```sh
go build code.cloudfoundry.org/copilot/cmd/copilot-server
```

## Using the Server

We are using a generic grpc client to interact with cloud controller grpc service (installation instructions below)

### Setup GRPC Client

If you are developing locally, you can install `grpcurl`
```sh
go get -u github.com/fullstorydev/grpcurl
```

If you are using a cloudfoundry
- bosh ssh to the istio-control vm and `sudo su`
- grpcurl is at `/var/vcap/packages/grpcurl/bin/grpcurl`
- the certs you need are in `/var/vcap/jobs/pilot-discovery/config/certs/`


### Push an App

```sh
cf push ...
```

### Find Diego Process GUID
##### Get the CAPI Process GUID:
The following example assumes the "web" process type, but you can replace that with another type if you know what you're doing.

```sh
export CAPI_PROCESS_GUID=$(cf curl "/v3/apps/$(cf curl "/v3/apps" | jq -r '.resources[] | select(.name == "<app-name>") | .guid')/processes" | jq -r '.resources[] | select(.type == "web") | .guid')
```

##### Get the CAPI Process Version:
The CAPI Process GUID is not sufficient for routing. If you want to map/delete a route, you'll need the entire `<capi-process-guid>-<version>` concatenation (the "Diego Process GUID"):

```sh
export APP_GUID=$(cf app <my-app> --guid) # to obtain the application guid
export CAPI_PROCESS_VERSION=$(cf curl /v2/apps/$APP_GUID | jq -r .entity.version) # to obtain the version
```

##### Construct the Diego Process GUID
```sh
export DIEGO_PROCESS_GUID="$CAPI_PROCESS_GUID-$CAPI_PROCESS_VERSION"
```

##### Get the Route Guid used by Cloud Controller
Given an existing route in cloud controller...

```sh
export CAPI_ROUTE_GUID=$(cf curl /v2/routes | jq -r '.resources[] | select(.entity.host == "<hostname-of-existing-route>").metadata.guid')
```

### As Cloud Controller, Add a Route

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  -d '{"route": {"host": "example.com", "guid": "route-guid-a"}}' \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/UpsertRoute
```

### As Cloud Controller, Map a Route

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  -d '{"route_mapping": {"route_guid": "route-guid-a", "capi_process_guid": "capi_guid_1"}}' \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/MapRoute
```

### As Cloud Controller, Associate a CAPI Process with a Diego Process

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  -d '{"capi_diego_process_association": {"capi_process_guid": "capi_guid_1", "diego_process_guids": ["diego_guid_1"]}}' \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/UpsertCapiDiegoProcessAssociation
```

### As Cloud Controller, Delete an Association between a CAPI Process and a Diego Process

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  -d '{"capi_process_guid": "capi_guid_1"}' \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/DeleteCapiDiegoProcessAssociation
```

### As Cloud Controller, Unmap a Route

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  -d '{"route_mapping": {"capi_process_guid": "capi_guid_1", "route_guid": "route-guid-a"}}' \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/UnmapRoute
```

### As Cloud Controller, Delete a Route

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  -d '{"guid": "route-guid-a"}' \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/DeleteRoute
```

### As Cloud Controller, Bulk Sync Routes Between Cloud Controller and Istio

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  -d '{"route_mappings": [{"route_guid": "route-guid-1", "capi_process_guid": "capi-guid-1"}, \
  "routes": [{"host": "example.org", "guid": "route-guid-1"}], \
  "capi_diego_process_associations": [{"capi_process_guid": "capi-guid-1", "diego_process_guids": ["diego-guid-1", "diego-guid-2"]}]}' \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/BulkSync
```

## The following endpoints are only used for debugging. They expose Copilot's internal state

### List the CF Routes that Copilot knows about

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/ListCfRoutes
```

### List the CF Route Mappings that Copilot knows about

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/ListCfRouteMappings
```

### List the associations between CAPI Process GUIDs and Diego Process GUIDs that Copilot knows about

(running from `/var/vcap/jobs/pilot-discovery/config/certs`)
```sh
/var/vcap/packages/grpcurl/bin/grpcurl -cacert ./ca.crt \
  -key ./client.key \
  -cert ./client.crt \
  copilot.service.cf.internal:9001 \
  api.CloudControllerCopilot/ListCapiDiegoProcessAssociations
```

