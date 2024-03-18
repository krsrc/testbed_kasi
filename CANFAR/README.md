# CANFAR Deployment

> [!NOTE]
>
> - CANFAR reference:
>   - <https://github.com/opencadc/science-platform/tree/SP-3544/deployment/helm>
> - OIDC TOKEN reference:
>   - <https://confluence.skatelescope.org/display/SRCSC/RED-10+Using+oidc-agent+to+authenticate+to+OpenCADC+services>

## Access check using OIDC

Just for getting temporary access to test each pods - posixmapper, skaha, and so on.

Prepare oidc : running oidc agent

```bash
eval $(oidc-agent)
eval $(oidc-agent-service use) > /dev/null
```

create a new device (time limited)

```bash
oidc-gen --iss=https://ska-iam.stfc.ac.uk --scope max --flow=device <id>
```

check id

```bash
oidc-gen --print <id>
```

when time passed long, it is needed to load again

```bash
oidc-add <id>
```

it is verification with host ID

```bash
curl -s -H "authorization: bearer $SKA_TOKEN" https://ska-iam.stfc.ac.uk/userinfo | jq
```

To test each process, SKA_TOKEN SHOULD BE SET BY BELOW (SKA_TOKEN):

```bash
export SKA_TOKEN=$(oidc-token <id>)
```
