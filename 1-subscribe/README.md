# Build a Serverless Mailchimp Clone with AWS Step Functions and Amazon Simple Email Service - Part 1 - Subscribe endpoint

Very short summary of the [full article available on dev.to](TODO).

## Simple Email Service (SES) setup

Be sure to adjust the variables in the `.json` snippets. Ideally, go through each command step by step.

```sh
# create email identity for sending emails from
EMAIL_ADDRESS='YOUR_EMAIL_ADDRESS'
aws sesv2 create-email-identity --email-identity $EMAIL_ADDRESS

# create email template
aws sesv2 create-email-template --cli-input-json file://welcome.create-email-template.input.json

aws sesv2 test-render-email-template \
  --template-name WelcomeTemplate \
  --template-data '{"email":"hello@example.com"}'

# MANUALLY adjust send-email.input.json
# then send an email
aws sesv2 send-email --cli-input-json file://send-email.input.json

# create contact list
aws sesv2 create-contact-list --cli-input-json file://create-contact-list.input.json

# create a test contact
CONTACTS_EMAIL_ADDRESS="YOUR_CONTACTS_EMAIL_ADDRESS"
aws sesv2 create-contact --contact-list-name "EmailNewsletter" --email-address $CONTACTS_EMAIL_ADDRESS

# list contacts to verify
aws sesv2 list-contacts --contact-list-name "EmailNewsletter"

# delete the test contact
CONTACTS_EMAIL_ADDRESS="YOUR_CONTACTS_EMAIL_ADDRESS"
aws sesv2 delete-contact --contact-list-name "EmailNewsletter" --email-address $CONTACTS_EMAIL_ADDRESS
```

## Step Functions setup

```sh
# MANUALLY adjust stepfunctions-newsletter-subscribe.policy.json
# then create the role
aws iam create-role \
 --role-name StepFunctionsNewsletterSubscribeRole \
 --assume-role-policy-document file://stepfunctions-newsletter-subscribe.trust-policy.json
# and attach a policy to the role
aws iam put-role-policy \
    --role-name StepFunctionsNewsletterSubscribeRole \
    --policy-name StepFunctionsNewsletterSubscribePolicy \
    --policy-document file://stepfunctions-newsletter-subscribe.policy.json

# MANUALLY adjust newsletter-subscribe.asl.json
# then create the state machine
ACCOUNT_ID="YOUR_ACCOUNT_ID"
aws stepfunctions create-state-machine \
    --name "NewsletterSubscribe" \
    --definition file://newsletter-subscribe.asl.json \
    --role-arn "arn:aws:iam::${ACCOUNT_ID}:role/StepFunctionsNewsletterSubscribeRole"

# optionally, create an execution from CLI to verify
REGION="YOUR_REGION"
ACCOUNT_ID="YOUR_ACCOUNT_ID"
CONTACTS_EMAIL_ADDRESS="YOUR_CONTACT_EMAIL"
aws stepfunctions start-execution \
    --state-machine-arn "arn:aws:states:${REGION}:${ACCOUNT_ID}:stateMachine:NewsletterSubscribe" \
    --input "{\"email\" : \"${CONTACTS_EMAIL_ADDRESS}\"}"
# and check the execution status
EXECUTION_ARN="YOUR_EXECUTION_ARN" # as returned by start-execution
aws stepfunctions describe-execution \
    --execution-arn $EXECUTION_ARN
```

## API Gateway setup

```sh
# create the api
aws apigatewayv2 create-api \
    --name "Newsletter" \
    --protocol-type HTTP \
    --description "API for managing the blog newsletter"

# create a default stage
API_ID='YOUR_API_ID'
aws apigatewayv2 create-stage \
    --api-id $API_ID \
    --stage-name '$default' \
    --auto-deploy

# MANUALLY adjust stepfunctions-start-execution-of-newsletter-subscribe.policy.json
# then create the role
aws iam create-role \
    --role-name ApiGatewayExecutionRoleForStepFunctionsNewsletterSubscribe \
    --assume-role-policy-document file://apigateway-execution-role.trust-policy.json
# and attach a policy to the role
aws iam put-role-policy \
    --role-name ApiGatewayExecutionRoleForStepFunctionsNewsletterSubscribe \
    --policy-name StepFunctionsStartExecutionOfNewsletterSubscribe \
    --policy-document file://stepfunctions-start-execution-of-newsletter-subscribe.policy.json

# create an integration to link the API Gateway to the Step Functions state machine
API_ID='YOUR_API_ID'
ACCOUNT_ID='YOUR_ACCOUNT_ID'
REGION='YOUR_REGION'
aws apigatewayv2 create-integration \
    --api-id $API_ID \
    --integration-type AWS_PROXY \
    --integration-subtype StepFunctions-StartExecution \
    --credentials-arn "arn:aws:iam::${ACCOUNT_ID}:role/ApiGatewayExecutionRoleForStepFunctionsNewsletterSubscribe" \
    --payload-format-version 1.0 \
    --request-parameters "{\"StateMachineArn\": \"arn:aws:states:${REGION}:${ACCOUNT_ID}:stateMachine:NewsletterSubscribe\", \"Input\": \"\$request.body\"}"

# create the POST /subscribe route
API_ID='YOUR_API_ID'
INTEGRATION_ID="YOUR_INTEGRATION_ID"
aws apigatewayv2 create-route \
  --api-id $API_ID \
  --route-key 'POST /subscribe' \
  --target "integrations/$INTEGRATION_ID"

# verify it works
API_ID='YOUR_API_ID'
REGION='YOUR_REGION'
CONTACTS_EMAIL_ADDRESS='YOUR_CONTACT_EMAIL'
curl --request POST \
  --url "https://${API_ID}.execute-api.${REGION}.amazonaws.com/subscribe" \
  --header 'Content-Type: application/json' \
  --data '{"email": "${CONTACTS_EMAIL_ADDRESS}"}'

# optionally, setup CORS
API_ID='YOUR_API_ID'
ORIGIN='YOUR_CORS_ORIGIN'
# for example, ORIGIN='https://www.example.com'
aws apigatewayv2 update-api --api-id $API_ID --cors-configuration AllowOrigins=$ORIGIN,AllowMethods="POST",AllowHeaders="Content-Type"
```

## Cleanup

```sh
API_ID="YOUR_API_ID"
REGION="YOUR_REGION"
ACCOUNT_ID="YOUR_ACCOUNT_ID"
SENDER_EMAIL_ADDRESS='YOUR_EMAIL_ADDRESS' # the email address you used as SES Identity

# API Gateway section
aws apigatewayv2 delete-api --api-id $API_ID # also takes care of the route etc.
aws iam delete-role-policy --role-name ApiGatewayExecutionRoleForStepFunctionsNewsletterSubscribe --policy-name StepFunctionsStartExecutionOfNewsletterSubscribe
aws iam delete-role --role-name ApiGatewayExecutionRoleForStepFunctionsNewsletterSubscribe

# Step Functions section
aws stepfunctions delete-state-machine --state-machine-arn "arn:aws:states:${REGION}:${ACCOUNT_ID}:stateMachine:NewsletterSubscribe"
aws iam delete-role-policy --role-name StepFunctionsNewsletterSubscribeRole --policy-name StepFunctionsNewsletterSubscribePolicy
aws iam delete-role --role-name StepFunctionsNewsletterSubscribeRole

# SES section
aws sesv2 delete-contact-list --contact-list EmailNewsletter
aws sesv2 delete-email-template --template-name WelcomeTemplate
aws sesv2 delete-email-identity --email-identity $SENDER_EMAIL_ADDRESS
```
