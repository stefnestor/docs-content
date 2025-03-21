# Connect to Amazon Bedrock [security-connect-to-bedrock]

This page provides step-by-step instructions for setting up an Amazon Bedrock connector for the first time. This connector type enables you to leverage large language models (LLMs) within {{kib}}. You’ll first need to configure AWS, then configure the connector in {{kib}}.

::::{note}
Only Amazon Bedrock’s `Anthropic` models are supported: `Claude` and `Claude instant`.

::::



## Configure AWS [security-connect-to-bedrock-configure-aws]


### Configure an IAM policy [security-connect-to-bedrock-configure-an-iam-policy]

First, configure an IAM policy with the necessary permissions:

1. Log into the AWS console and search for Identity and Access Management (IAM).
2. From the **IAM** menu, select **Policies** → **Create policy**.
3. To provide the necessary permissions, paste the following JSON into the **Specify permissions** menu.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": "*"
        }
    ]
}
```

::::{note}
These are the minimum required permissions. IAM policies with additional permissions are also supported.

::::


1. Click **Next**. Name your policy.

The following video demonstrates these steps.

 <iframe
  src="https://drive.google.com/file/d/1RnTQ0jjL9YdKQYy3eW0481YJzlZIix17/preview?usp=sharing"
  width="100%"
  height="100%"
  style="border:none"
></iframe>

### Configure an IAM User [security-connect-to-bedrock-configure-an-iam-user]

Next, assign the policy you just created to a new user:

1. Return to the **IAM** menu. Select **Users** from the navigation menu, then click **Create User**.
2. Name the user, then click **Next**.
3. Select **Attach policies directly**.
4. In the **Permissions policies** field, search for the policy you created earlier, select it, and click **Next**.
5. Review the configuration then click **Create user**.

The following video demonstrates these steps.

 <iframe
  src="https://drive.google.com/file/d/1qsla82M6XhCDdFumS8pxMQLyA2nsxQzf/preview?usp=sharing"
  width="100%"
  height="100%"
  style="border:none"
></iframe>

### Create an access key [security-connect-to-bedrock-create-an-access-key]

Create the access keys that will authenticate your Elastic connector:

1. Return to the **IAM** menu. Select **Users** from the navigation menu.
2. Search for the user you just created, and click its name.
3. Go to the **Security credentials** tab.
4. Under **Access keys**, click **Create access key**.
5. Select **Third-party service**, check the box under **Confirmation***, click ***Next**, then click **Create access key**.
6. Click **Download .csv file** to download the key. Store it securely.

The following video demonstrates these steps.

 <iframe
  src="https://drive.google.com/file/d/1bgv-r_xSE3KYOAf2ufvPyqVpn-deFELp/preview?usp=sharing"
  width="100%"
  height="100%"
  style="border:none"
></iframe>

### Enable model access [security-connect-to-bedrock-enable-model-access]

Make sure the supported Amazon Bedrock LLMs are enabled:

1. Search the AWS console for Amazon Bedrock.
2. From the Amazon Bedrock page, click **Get started**.
3. Select **Model access** from the left navigation menu, then click **Manage model access**.
4. Check the boxes for **Claude** and/or **Claude Instant**, depending which model or models you plan to use.
5. Click **Save changes**.

The following video demonstrates these steps.

 <iframe
  src="https://drive.google.com/file/d/1V6n5q2RAEiQN6Zmz-SZ0ANR5cr4HZnEj/preview?usp=sharing"
  width="100%"
  height="100%"
  style="border:none"
></iframe>

## Configure the Amazon Bedrock connector [security-connect-to-bedrock-configure-the-amazon-bedrock-connector]

Finally, configure the connector in {{kib}}:

1. Log in to {{kib}}.
2. Find **Connectors** in the navigation menu or use the global search field. Then click **Create Connector**, and select **Amazon Bedrock**.
3. Name your connector.
4. (Optional) Configure the Amazon Bedrock connector to use a different AWS region where Anthropic models are supported by editing the **URL** field, for example by changing `us-east-1` to `eu-central-1`.
5. (Optional) Add one of the following strings if you want to use a model other than the default:

    * For Haiku: `anthropic.claude-3-haiku-20240307-v1:0`
    * For Sonnet: `anthropic.claude-3-sonnet-20240229-v1:0`
    * For Opus: `anthropic.claude-3-opus-20240229-v1:0`

6. Enter the **Access Key** and **Secret** that you generated earlier, then click **Save**.

Your LLM connector is now configured. For more information on using Elastic AI Assistant, refer to [AI Assistant](https://docs.elastic.co/security/ai-assistant).

::::{important}
If you’re using [provisioned throughput](https://docs.aws.amazon.com/bedrock/latest/userguide/prov-throughput.html), your ARN becomes the model ID, and the connector settings **URL** value must be [encoded](https://www.urlencoder.org/) to work. For example, if the non-encoded ARN is `arn:aws:bedrock:us-east-2:123456789102:provisioned-model/3Ztr7hbzmkrqy1`, the encoded ARN would be `arn%3Aaws%3Abedrock%3Aus-east-2%3A123456789102%3Aprovisioned-model%2F3Ztr7hbzmkrqy1`.

::::


The following video demonstrates these steps.

 <iframe
  src="https://drive.google.com/file/d/1_0ipXxQ6b5mVSJYSYLhR9SXCxlmZ32RJ/preview?usp=sharing"
  width="100%"
  height="100%"
  style="border:none"
></iframe>
