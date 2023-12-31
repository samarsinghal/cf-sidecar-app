# Cloud Foundry Sidecar

This is a proof of concept to implement sidecars with Cloud Foundry application to intercept and redirect traffic to the application on TAS and prevent any direct access to the main application. 

Requires cf CLI v7 or greater.

## Procedure 

1. `cf push --var app-domain=<APP DOMAIN GOES HERE>`
1. Find the cf-sidecar-app route guid: `cf curl '/v3/routes?hosts=cf-sidecar-app' | jq '.resources[].guid'`
1. Find the app guid: `cf curl '/v3/apps?names=cf-sidecar-app' | jq '.resources[].guid'`
1. Update the cf-sidecar-app route to point to app port 8081, instead of 8080: `cf curl -X PATCH /v3/routes/<ROUTE GUID GOES HERE>/destinations -d '{"destinations": [{"app": { "guid": "<APP GUID GOES HERE>", "process": { "type": "web" } }, "weight": null, "port": 8081, "protocol": "http1" }]}'`
1. Authorize c2c networking between the app and itself: `cf add-network-policy cf-sidecar-app cf-sidecar-app`
1. Restart the app to update port configuration for the container: `cf restart cf-sidecar-app`
1. Curl the server: `curl cf-sidecar-app.<APP DOMAIN GOES HERE>`

## How This Works

The app's web process and sidecar are invocations of the `nc` utility already present in the application container. This is why this "app" doesn't have any source code, only start commands configured in the app manifest.

The web process receives traffic via the external route (`cf-sidecar-app.<app domain>`) and then curls the main app. The main app responds to the sidecar request, and the sidecar responds to the original request, including the main app response.

There is only one port exposed on the C2C overlay network per container. This port always maps to the `$PORT` environment variable
inside the container (default `:8080`). Thus, whatever we want to connect to
via C2C has to be bound to`:8080` in the container. This demo moves the
"cf-sidecar-app.<app domain>" route to another port (`:8081`, but this could be
anything), so the sidecar process can bind to `:8080` and be exposed
on the C2C network.

Relevant documentation:
- https://docs.cloudfoundry.org/devguide/deploy-apps/cf-networking.html
- https://v3-apidocs.cloudfoundry.org/version/release-candidate/#replace-all-destinations-for-a-route
- https://docs.cloudfoundry.org/devguide/deploy-apps/manifest-attributes.html
- https://docs.cloudfoundry.org/devguide/custom-ports.html
- https://docs.cloudfoundry.org/devguide/sidecars.html

Reference: 
- https://github.com/Gerg/cf-sidecar-networking

