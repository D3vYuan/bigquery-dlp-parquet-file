<!-- Improved compatibility of back to top link: See: https://github.com/othneildrew/Best-README-Template/pull/73 -->
<div id="top"></div>

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <h2 align="center">Bigquery DLP with Parquet File</h2>
  <p align="center">
    Case Study - Bigquery DLP with Parquet File
  </p>
  <!--div>
    <img src="images/profile_pic.png" alt="Logo" width="80" height="80">
  </div-->
</div>

---

<!-- TABLE OF CONTENTS -->

## Table of Contents

<!-- <details> -->
<ol>
    <li>
        <a href="#about-the-project">About The Project</a>
        <ul>
            <li><a href="#architecture">Architecture</a></li>
            <li><a href="#data">Data</a></li>
        </ul>
    </li>
    <li>
        <a href="#setup">Setup</a>
        <ul>
            <li><a href="#enable-services">Enable Services</a></li>
            <li><a href="#create-cryptographic-key">Create Cryptographic Key</a></li>
            <li><a href="#checkout-source-codes">Checkout Source Codes</a></li>
            <li><a href="#create-inspecttemplate-for-custom-infotypes">Create InspectTemplate for Custom InfoTypes</a></li>
            <li><a href="#create-deidentifytemplate-for-encryption">Create DeidentifyTemplate for Encryption</a></li>
        </ul>
    </li>
    <li>
        <a href="#implementation">Implementation</a>
        <ul>
            <li><a href="#via-terraform">Terraform</a></li>
        </ul>
    </li>
    <li>
        <a href="#usage">Usage</a>
        <ul>
            <li><a href="#sensitive-data-protection">De-identify template testing via Sensitive Data Protection</a></li>
            <li><a href="#bigquery">Remote Functions via BigQuery</a></li>
        </ul>
    </li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
</ol>
<!-- </details> -->

<!-- ABOUT THE PROJECT -->

## About The Project

This project is created to showcase how we can run a `Data Loss Prevention (DLP)` API on BigQuery on a parquet file.

The following are some of the requirements:

- Perform encryption of Personal Identifiable Information (PII) data
- Perform decryption of Personal Identifiable Information (PII) data

In this case study, the PII data is phone number, in particular Singapore Phone Number, 
which happens to not be in the [built-in][ref-dlp-builtin-infotypes] infotypes 
so we will have to create a custom infotypes to de-identify this data.

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- Architecture -->

### Architecture

The following depicts the architecture of the following implementation. There will be functions deployed remotely into BigQuery, which will called the `DLP API` to encrypt the data. More information on the implemenation can be found [here][ref-dlp-tutorial]

| ![dlp-architecture][img-dlp-architecture] | 
|:--:| 
| *BigQuery De-identify Data Architecture* |

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- Data -->

### Data

According to [wikipedia][ref-dlp-sg-phone], Singapore mobile phone consists of the following format:
- `9`XXXXXXX
- `8`XXXXXXX

where `X` is a number between `0` to `9`

In order to capture this pattern we will be using the following regex:

```regex
[8-9]{1}[0-9]{7}
```

which will capture only when the first number is `8` or `9`, and for the remaining 7 numbers between `0` to `9`

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- Setup -->

## Setup

Base on the requirements, the following components are required to be setup:

- `BigQuery`
- `CloudBuild`
- `Sensitive Data Protection`
- `Cloud Run`
- `Artifact Registry`

We will need to carry out the following tasks before proceeding to the actual implementation
- Enable Services
- Create Cryptographic Key
- Checkout Source Codes
- Create InspectTemplate for Custom InfoTypes
- Create DeidentifyTemplate for Encryption

<p align="right">(<a href="#top">back to top</a>)</p>

### Enable Services

The following services will be used:

- Cloud Data Loss Prevention (DLP) API (`dlp.googleapis.com`)
- Cloud Key Management Service (KMS) API (`cloudkms.googleapis.com`)
- BigQuery Connection API (`bigqueryconnection.googleapis.com`)

To enable the services, the following command can be used:

```sh
gcloud services enable dlp.googleapis.com cloudkms.googleapis.com bigqueryconnection.googleapis.com
```

<p align="right">(<a href="#top">back to top</a>)</p>

### Create Cryptographic Key

<div id="crypto-key"></div>

The following are the steps required to create the cryptographic key
- Create Keyring
- Create Key
- List Key
- Create AES Key
- Encode AES Key
- Wrapped AES Key

#### Create Keyring

```sh
gcloud kms keyrings create "<key-ring-name>" \
    --location "<location>"
```

*Example:*

```sh
gcloud kms keyrings create "dlp-keyring" \
    --location "asia-southeast1"
```

<p align="right">(<a href="#crypto-key">back to list</a>)</p>

#### Create Key

```sh
gcloud kms keys create "<key-name>" \
    --location "<location>" \
    --keyring "<key-ring-name>" \
    --purpose "encryption"
```

*Example:*

```sh
gcloud kms keys create "dlp-key" \
    --location "asia-southeast1" \
    --keyring "dlp-keyring" \
    --purpose "encryption"
```

<p align="right">(<a href="#crypto-key">back to list</a>)</p>

#### List Key

```sh
gcloud kms keys list \
    --location "<location>" \
    --keyring "<key-ring-name>"
```

*Example:*

```sh
gcloud kms keys list \
    --location "asia-southeast1" \
    --keyring "dlp-keyring"
```

<p align="right">(<a href="#crypto-key">back to list</a>)</p>

#### Create AES Key

```sh
openssl rand -out "./aes_key.bin" 32
```

<p align="right">(<a href="#crypto-key">back to list</a>)</p>

#### Encode AES Key

The output of this command is also refer as the unwrapped key and will use in the following step to generate the wrapped key

```sh
base64 -i ./aes_key.bin
```

<p align="right">(<a href="#crypto-key">back to list</a>)</p>

#### Wrapped AES Key

The output of this command is also refer as the wrapped key and will be used in the de-identify template

```sh
curl "https://cloudkms.googleapis.com/v1/projects/<project_id></project_id>/locations/<location>/keyRings/<key-ring-name>/cryptoKeys/<key-name>:encrypt" \
  --request "POST" \
  --header "Authorization:Bearer $(gcloud auth application-default print-access-token)" \
  --header "content-type: application/json" \
  --data "{\"plaintext\": \"<unwrapped_key>\"}"
```

<p align="right">(<a href="#crypto-key">back to list</a>)</p>

### Checkout Source Codes

The source codes can be found in the following github [repository][ref-dlp-github] and we'll need to clone to our local environment (or `cloud shell`) to make the necessary configuration

```sh
git clone https://github.com/GoogleCloudPlatform/bigquery-dlp-remote-function.git
```

<p align="right">(<a href="#top">back to top</a>)</p>

### Create InspectTemplate for Custom InfoTypes
Along with the source codes, the repository also comes with the sample configuration file for the ` inspectTemplate` and `deidentifyTemplate` - *sample_dlp_deid_config.json*

We will make a copy of this and create the `custom` infotypes using the `inspectTemplate`

```bash
cp -pv sample_dlp_deid_config.json phone_number_isp_config_custom.json
```

The `inspectTemplate` contains information on the metadata of the template and the way to identify the data, in our case we are using `regex`. More information can be found [here][ref-dlp-custom-infotypes]

**phone_number_isp_config_custom.json** 
```json
{
    "inspectTemplate": {
        "displayName": "SG Phone number inspection",
        "description": "Scans for SG phone numbers",
        "inspectConfig": {
            "customInfoTypes": [
                {
                    "infoType": {
                        "name": "CUSTOM_SG_PHONE_NUMBER"
                    },
                    "regex": {
                        "pattern": "[8-9]{1}[0-9]{7}"
                    },
                    "likelihood": "POSSIBLE"
                }
            ]
        }
    }
}

```

Once the configuration in the template is defined, we can load it into `Sensitive Data Protection` to be used by the `deidentifyTemplate` subsequently

`NOTE` Replace the `PROJECT_ID`, `DLP_ISP_TEMPLATE`, `REGION` with the corresponding values, with the `DLP_ISP_TEMPLATE` being the filename of the `inspectTemplate` (E.g phone_number_isp_config_custom.json)

```sh
ISP_TEMPLATE=$(curl -X POST \
    -H "Authorization: Bearer `gcloud auth print-access-token`" \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -H "X-Goog-User-Project: ${PROJECT_ID}" \
    --data-binary "@${DLP_ISP_TEMPLATE}" \
"https://dlp.googleapis.com/v2/projects/${PROJECT_ID}/locations/${REGION}/inspectTemplates")
ISP_TEMPLATE_NAME="$(echo ${ISP_TEMPLATE} | jq -r '.name')"
echo ${ISP_TEMPLATE_NAME}
```

The output of other above command will be the resource location of the `inspectTemplate` and will be used subsequently in the setup `terraform` script. 

Alternatively, you may navigate to the `Sensitive Data Protection` page, under the `Configuration` > `Template` > `Inspect`, the uploaded `inspectTemplate` should be present as well

| ![dlp-inspect-template-configuration][img-dlp-inspect-template-configuration] | 
|:--:| 
| *BigQuery De-identify Data Architecture* |

<p align="right">(<a href="#top">back to top</a>)</p>

### Create DeidentifyTemplate for Encryption

Similar to `inspectTemplate`, we will make a copy of the sample file and create the configuration for encryption for the `deidentifyTemplate`

```bash
cp -pv sample_dlp_deid_config.json phone_number_dlp_config_custom.json
```

The `deidentifyTemplate` contains information on the metadata of the template and the way to de-identify the data, in this case we are using the `custom` infotypes (i.e *CUSTOM_SG_PHONE_NUMBER*) specified earlier by the `inspectTemplate`. More information can be found [here][ref-dlp-deidentify-template]

`NOTE` Replace the `project-id`, `location`, `keyring-name`, `key-name`, `wrapped-key` with the corresponding values

**phone_number_dlp_config_custom.json** 

```json
{
    "deidentifyTemplate": {
        "displayName": "Phone number masker",
        "description": "De-identifies phone numbers with wrapped key.",
        "deidentifyConfig": {
            "infoTypeTransformations": {
                "transformations": [
                    {
                        "infoTypes": [
                            {
                              "name": "CUSTOM_SG_PHONE_NUMBER"
                            }
                        ],
                        "primitiveTransformation": {
                            "cryptoDeterministicConfig": {
                                "cryptoKey": {
                                    "kmsWrapped": {
                                        "cryptoKeyName": "projects/<project-id>/locations/<location>/keyRings/<keyring-name>/cryptoKeys/<key-name>",
                                        "wrappedKey": "<wrapped-key>"
                                    }
                                },
                                "surrogateInfoType": {
                                    "name": "PHONE_NUMBER"
                                }
                            }
                        }
                    }
                ]
            }
        }
    }
}
```

<p align="right">(<a href="#top">back to top</a>)</p>

## Implementation

Base on the requirements, the following are the tasks in the process:

- Setting up `Remote Functions` using `Terraform` script

### via Terraform

Before applying the `Terraform` script, we will need to gather the variables required

|#|Variables|Description|Remarks|
|:--:|--|--|--|
|1|`project_id`|The project that the resources will be deployed to||
|2|`region`|The location where the resources will be deployed to||
|3|`dlp_deid_template_json_file`|The `deidentifyTemplate` file name|Example: <br/>xxxx.`json`|
|4|`dlp_inspect_template_full_path`|The resource location of the `inspectTemplate`|Example: <br/>projects/\<project-id\>/locations/\<location\>/`inspectTemplates`/\<template-id\>|

<br/>

`NOTE` Replace `<project_id>`, `<region>`, `<dlp_deid_template_json_file>`, `<dlp_inspect_template_full_path>` with the corresponding values

```sh
export PROJECT_ID="<project_id>"
export REGION="<region>"
export DLP_DEID_TEMPLATE="<dlp_deid_template_json_file>"
export DLP_INSPECT_TEMPLATE="<dlp_inspect_template_full_path>"

terraform init && \
	terraform apply \
		-var "project_id=${PROJECT_ID}" \
		-var "region=${REGION}" \
		-var "dlp_deid_template_json_file=${DLP_DEID_TEMPLATE}" \
		-var "dlp_inspect_template_full_path=${DLP_INSPECT_TEMPLATE}"
```

| ![dlp-terraform-success][img-dlp-terraform-success] | 
|:--:| 
| *Successful Application of Terraform* |

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- USAGE EXAMPLES -->

## Usage

There are 2 ways which we can use the implementation
- `De-identify` template testing via `Sensitive Data Protection`
- `Remote Functions` via `BigQuery`

<p align="right">(<a href="#top">back to top</a>)</p>

### Sensitive Data Protection

Once the `Terraform` script completed successfully, the `deidentifyTemplate` will be uploaded to the `Sensitive Data Protection`.

You may navigate to the `Sensitive Data Protection` page, under the `Configuration` > `Template` > `De-Identify`, the uploaded `deidentifyTemplate` should be present as well

| ![dlp-deidentify-template-configuration][img-dlp-deidentify-template-configuration] | 
|:--:| 
| *De-Identify Template Configuration* |

If we clicked into the details page, there is a tab where we can run various test cases to verify the detection and encryption

| ![dlp-deidentify-template-testing][img-dlp-deidentify-template-testing] | 
|:--:| 
| *De-Identify Template Testing* |

<p align="right">(<a href="#top">back to top</a>)</p>

### BigQuery

Once the `Terraform` script completed successfully, the remote functions `dlp_freetext_decrypt` and `dlp_freetext_encrypt` will be created in `BigQuery`

To test the remote functions, we can use the following query

```sql
SELECT
    pii_column,
    fns.dlp_freetext_encrypt(pii_column) AS dlp_encrypted,
    fns.dlp_freetext_decrypt(fns.dlp_freetext_encrypt(pii_column)) AS dlp_decrypted
FROM
    UNNEST(
    [
        'My name is John Doe. My number is 81234567',
        'Some non PII data',
        '91234567',
        'some script with simple number 12345678']) AS pii_column;
```

| ![dlp-remote-functions-results][img-dlp-remote-functions-results] | 
|:--:| 
| *Remote Functions Encryption Results* |

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- ACKNOWLEDGMENTS -->

## Acknowledgments

<!-- - [AEAD Encryption Functions][ref-aead-function] -->
- [Readme Template][template-resource]
- [Reference Github][ref-dlp-github]
- [Reference Setup][ref-dlp-tutorial]

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
[template-resource]: https://github.com/othneildrew/Best-README-Template/blob/master/README.md

[ref-dlp-tutorial]: https://cloud.google.com/dlp/docs/deidentify-bq-tutorial#console
[ref-dlp-github]: https://github.com/GoogleCloudPlatform/bigquery-dlp-remote-function

[ref-dlp-builtin-infotypes]: https://cloud.google.com/dlp/docs/infotypes-reference
[ref-dlp-custom-infotypes]: https://cloud.google.com/dlp/docs/examples-custom-infotypes
[ref-dlp-deidentify-template]: https://cloud.google.com/dlp/docs/creating-templates-deid#create_a_de-identification_template

[ref-dlp-sg-phone]: https://en.wikipedia.org/wiki/National_conventions_for_writing_telephone_numbers#:~:text=XXX\)%20YYY%20ZZZZ.-,Singapore,-In%20Singapore%2C%20every

[img-dlp-architecture]: ./images/dlp_architecture.png

[img-dlp-inspect-template-configuration]: ./images/dlp_inspect_template_configuration.png
[img-dlp-inspect-template-details]: ./images/dlp_inspect_template_details.png

[img-dlp-deidentify-template-configuration]: ./images/dlp_deidentify_template_configuration.png
[img-dlp-deidentify-template-details]: ./images/dlp_deidentify_template_details.png
[img-dlp-deidentify-template-testing]: ./images/dlp_deidentify_template_testing.png

[img-dlp-terraform-success]: ./images/dlp_terraform_success.png

[img-dlp-remote-functions]: ./images/dlp_remote_functions.png
[img-dlp-remote-functions-testing]: ./images/dlp_remote_functions_testing.png
[img-dlp-remote-functions-results]: ./images/dlp_remote_functions_results.png

