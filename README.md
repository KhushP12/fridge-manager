# Fridge app: deploy to AWS (eu-north-1)

Set `P=<your-profile-name>` first (macOS/Linux: `export P=myprofile`).

## 1. Deploy the backend (DynamoDB + Lambda + API Gateway)

Requires the [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).

```bash
cd fridge-app
sam deploy --stack-name fridge-app --region eu-north-1 --profile $P \
  --capabilities CAPABILITY_IAM --resolve-s3
```

Copy the `ApiUrl` value from the stack outputs. To see it again:

```bash
aws cloudformation describe-stacks --stack-name fridge-app --region eu-north-1 --profile $P \
  --query "Stacks[0].Outputs"
```

Quick test:

```bash
curl "$API_URL/items"
```

## 2. Point the frontend at the API

In `index.html`, set `const API_URL = "https://xxxx.execute-api.eu-north-1.amazonaws.com/prod";`

## 3. Host the frontend on Amplify (manual deployment)

```bash
zip site.zip index.html

APP_ID=$(aws amplify create-app --name fridge-app --region eu-north-1 --profile $P --query app.appId --output text)
aws amplify create-branch --app-id $APP_ID --branch-name main --region eu-north-1 --profile $P

aws amplify create-deployment --app-id $APP_ID --branch-name main --region eu-north-1 --profile $P
# note the jobId and zipUploadUrl in the output, then:
curl -T site.zip "<zipUploadUrl>"
aws amplify start-deployment --app-id $APP_ID --branch-name main --job-id <jobId> --region eu-north-1 --profile $P
```

Your site is at `https://main.<APP_ID>.amplifyapp.com`.

Prefer the console? In Amplify choose **Deploy without Git**, then drag `site.zip` in.

## 4. Lock down CORS (recommended)

In `template.yaml`, replace `AllowOrigin: "'*'"` with `"'https://main.<APP_ID>.amplifyapp.com'"` and the `Access-Control-Allow-Origin` header in the Lambda code, then run `sam deploy` again.

## Notes

- The API has no authentication, so anyone with the URL can edit the fridge. Add a Cognito authorizer or API key before sharing it widely.
- API: `GET /items`, `POST /items` with `{"name":"Milk","shelf":0}` (shelf 0-3), `DELETE /items/{id}`.
- Clean up with `sam delete --stack-name fridge-app --region eu-north-1 --profile $P` and `aws amplify delete-app --app-id $APP_ID`.
