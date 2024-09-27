# Verify image with Notation in GitHub Actions

This document walks you through how to create a GitHub Actions workflow using Notation to verify a signed image stored in Azure Container Registry (ACR).

## Prerequisites

- You have a signed image stored in your ACR. 
- As a verifier, you can access the signed image stored in the ACR registry. If you didn't create a GitHub repository secret for ACR in your signing process, you can follow the step [Create ACR_PASSWORD](sign-action.md#create-acr_password) to create a GitHub repository secret. For demonstration convenience, we will use the same GitHub repository secret `secrets.ACR_PASSWORD` that we used in the [signing process](sign-action.md).
- You have a GitHub repository to store trusted certificates and trust policy for Notation verification. For demonstration convenience, we will use the same GitHub repository that we used in the signing process.

## Prepare Notation trust policy and trust store 

To verify the image signature, we need to create a Notation [trust policy](https://github.com/notaryproject/specifications/blob/main/specs/trust-store-trust-policy.md#trust-policy) file and a Notation [trust store](https://github.com/notaryproject/specifications/blob/main/specs/trust-store-trust-policy.md#trust-store) directory on the GitHub runner machine. Follow the steps below:

- In your GitHub repository, create directory `.github/trustpolicy`. In the directory, create a Notation trust policy file `trustpolicy.json`. You can copy the [trust policy template](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/template/.github/trustpolicy/trustpolicy.json) from the collapsed section below into your `trustpolicy.json` file. 

<details>

<summary>See the trust policy template (Click here).</summary>

```JSON
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "<yourPolicyName>",
            "registryScopes": [ "<yourRegistry>.azurecr.io/<yourArtifact>" ],
            "signatureVerification": {
                "level" : "strict" 
            },
            "trustStores": [ "ca:<yourTrustStore>"],
            "trustedIdentities": [
                "*"
            ]
        }
    ]
}
```

</details>

- In the `trustpolicy.json`, update the value of `registryScopes` with your registry `${registry-name}/${namespace}/${repository-name}`, where the signed image is stored.

- In your GitHub repository, create directory `.github/truststore/x509/ca/<yourTrustStore>`. 

- Sign in Azure with Azure CLI. If you followed the document [Sign an image with Notation in GitHub Actions](sign-action.md) to sign your image, use the commands below to download your self-signed certificate from Azure Key Vault (AKV). 

  ```
  KEY_NAME=<key_name_you_picked_when_creating_the_key>
  AKV_NAME=<akv_name_where_certificate_is_stored>
  CERT_ID=$(az keyvault certificate show -n $KEY_NAME --vault-name $AKV_NAME --query 'id' -o tsv)
  CERT_PATH=./${KEY_NAME}.pem
  az keyvault certificate download --file $CERT_PATH --id $CERT_ID --encoding PEM
  ```

- Save the downloaded certificate to `.github/truststore/x509/ca/<yourTrustStore>` created above.

### Create the GitHub Actions workflow

- In the `.github/workflows` directory, create a file named `<your_verify_workflow>.yml`. 

- You can copy the [verification template workflow](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/template/verify-template.yml) from the collapsed section below into your `<your_verify_workflow>.yml` file.

- Update the environmental variables based on your environment by following the comments in the template. Save and commit it to the repository.

<details>

<summary>See the verification workflow template (Click here).</summary>

```yaml
# Set up notation and verify an image stored in ACR
name: notation-github-actions-verify-template

on:
  push:

env:
  ACR_REGISTRY_NAME: <registry_name_of_your_ACR>                       # example: myRegistry.azurecr.io
  ACR_REPO_NAME: <repository_name_of_your_ACR>                         # example: myRepo
  target_artifact_reference: <ACR_REGISTRY_NAME/ACR_REPO_NAME@digest>  # example: myRegistry.azurecr.io/myRepo@sha256:abcdef
  NOTATION_EXPERIMENTAL: 1                                             # [Optional] when set, can use Referrers API in the workflow

jobs:
  notation-verify:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      # Log into your ACR
      - name: docker login
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.ACR_OF_RELEASE }}
          username: ${{ env.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      # Install Notation CLI, the default version is "1.0.0"
      - name: setup notation
        uses: notaryproject/notation-action/setup@main
      
      # Verify the image
      - name: verify image
        uses: notaryproject/notation-action/verify@main
        with:
          target_artifact_reference: ${{ env.target_artifact_reference }}
          trust_policy: .github/trustpolicy/trustpolicy.json
          trust_store: .github/truststore
          allow_referrers_api: 'true'
```

</details>

## Trigger the GitHub Actions workflow

The workflow trigger logic has been set to `on: push` [event](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-events-to-trigger-workflows) in the sample workflow. Pushing the workflow file to a branch in your repository triggers your workflow. You can also configure to run a workflow manually by following the [GitHub doc](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow) based on your need. 

## View your GitHub Actions workflow results

Under your GitHub repository name, click **Actions** tab of your GitHub repository to see the workflow logs.

On success, you will be able to see the signed image is successfully verified. The sha256 digest of the verified image is returned in Notation's output message.