# Prepare the CAP API sample for Deployment

Once the proxy via SAP BTP, AI Core is running ([see steps here](/01-ai-core-azure-openai-proxy/README.md)), you are able to prepare the CAP API sample in order to deploy it to a SAP BTP, Cloud Foundry Runtime in your Subaccount.

In the following parts, it's explained how the CAP boilerplate code needs to be extended to consume the inference service (proxy) of SAP BTP AI Core and expose it as an [action](https://cap.cloud.sap/docs/guides/providing-services#actions-and-functions) via an endpoint.

A destination service instance as well as an UAA service instance are attached to your CAP application in the `mta.yaml`. During the initial deployment, there is a destination with the neccessary information created which needs to get completed in the cockpit itself by entering the Client ID and Client Secret of your AI Core service key (see [Register general artifacts on SAP BTP, AI Core and inspect in SAP BTP, AI Launchpad](/documentation/01-ai-core-azure-openai-proxy/03-register-general-artifacts.md)):

```yaml
---
# -------------------- DESTINATION SERVICE -------------------
- name: cap-aicore-dest
  # ------------------------------------------------------------
  type: org.cloudfoundry.managed-service
  parameters:
    service: destination
    service-plan: lite
    config:
      init_data:
        instance:
          existing_destinations_policy: ignore
          destinations:
            - Name: openai-aicore-api
              Description: SAP AI Core deployed service
              URL: https://api.ai.prod.eu-central-1.aws.ml.hana.ondemand.com
              URL.headers.AI-Resource-Group: azure-openai-aicore # your resource group
              URL.headers.Content-Type: application/json
              Type: HTTP
              ProxyType: Internet
              Authentication: OAuth2ClientCredentials
              tokenServiceURL: https://sceaiml.authentication.eu10.hana.ondemand.com/oauth/token # your token service url of the SAP AI Core instance
              clientId: DUMMY_ID # enter in cockpit
              clientSecret: DUMMY_SECRET # enter in cockpit
              HTML5.DynamicDestination: true
```

1. Access the Destination Service on the SAP BTP Cockpit
   ![Destination Service](resources/destination-service.png)

2. Enter client ID and Client Secret
   ![Destination Service](resources/destination.png)

On the CAP application, the destination pointing to your proxy is defined in the `package.json` (including the path to your deployed proxy with the deployment id as shown in [Deploy the Inference Service on SAP BTP, AI Core as Proxy for Azure OpenAI Services](/documentation/01-ai-core-azure-openai-proxy/04-setup-deployment-inference-service.md)):

```json
{
    "name": "cap",
    "cds": {
        "requires": {
            "AICoreAzureOpenAIDestination": {
                "kind": "rest",
                "credentials": {
                    "destination": "openai-aicore-api",
                    "path": "/v2/inference/deployments/<YOUR_AICORE_DEPLOYMENT_ID>"
                }
            }
        }
    },
    ...
}
```

In the `/srv` directoy of the CAP API sample, there is an action defined in the service `ai-service.cds` which takes a string (prompt) as payload:

```typescript
@requires : 'authenticated-user'
service AIService {

    type GPTTextResponse {
        text : String;
    }

    action aiProxy(prompt : String) returns GPTTextResponse;
}
```

The Javascript (`ai-service.js`) / TypeScript handler (`ai-service.ts`) further processes the action by hooking the function `aiProxyAction`. Please specify your engine in the attribute `const ENGINE` which can be found after deploying a OpenAI model on Azure:

```typescript
import { ApplicationService } from "@sap/cds";
import { Request, ResultsHandler } from "@sap/cds/apis/services";

// PARAMETERS FOR AZURE OPENAI SERVICES
const ENGINE = "YOUR_ENGINE_OF_AZURE_OPENAI_SERVICES"
const MAX_TOKENS = 500;
const TEMPERATURE = 0.8;
const FREQUENCY_PENALTY = 0;
const PRESENCE_PENALTY = 0;
const TOP_P = 0.5;
const BEST_OF = 1;
const STOP_SEQUENCE = null;

const GPT_PARAMS = {
    engine: ENGINE,
    max_tokens: MAX_TOKENS,
    temperature: TEMPERATURE,
    frequency_penalty: FREQUENCY_PENALTY,
    presence_penalty: PRESENCE_PENALTY,
    top_p: TOP_P,
    best_of: BEST_OF,
    stop: STOP_SEQUENCE
}

// handler for ai-service.cds
class AIService extends ApplicationService {

    /**
     * Define handlers for CAP actions
     */
    async init(): Promise<void> {
        await super.init();
        this.on("aiProxy", this.aiProxyAction);
    }


    /**
    * Action forwarding prompt to through AI Core provided proxy
    */
    private aiProxyAction = async (req: Request): Promise<any | undefined> => {
        const { prompt } = req.data;
        const response = this.callAIProxy(prompt);
        return { text: response["choices"][0].text };
    };


    /**
     * Forwards prompt of the payload via a destination (mapped as AICoreAzureOpenAIDestination) through an AI Core deployed service to Azure OpenAI services
     */
    private callAIProxy = (prompt: string): Promise<any | undefined> => {
        const openai = await cds.connect.to("AICoreAzureOpenAIDestination");
        const payload: any = {
            ...GPT_PARAMS
            prompt: prompt,
        };

        // @ts-ignore
        const response: any = await openai.send({
            // @ts-ignore
            query: "POST /v2/completion",
            data: payload
        });
        return response;
    }
}
```