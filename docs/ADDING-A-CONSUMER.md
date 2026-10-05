# Adding a consumer

App clients are declared in `stacks/cognito.json`, not created by hand. The
template is therefore the record of who has API access, and git history shows
when each consumer was added and by whom.

Adding a consumer is a pull request plus a stack update. No change to the Lambda
or API Gateway stacks is needed — the authorizer trusts the pool, not individual
clients.

---

## 1. Add the client to the template

In `stacks/01-cognito.json`, copy the `ConsumerBusinessReporting` resource and
change two things: the logical ID and the `ClientName`.

```json
"ConsumerAcmeFinance": {
  "Type": "AWS::Cognito::UserPoolClient",
  "DependsOn": "AccountsResourceServer",
  "Properties": {
    "UserPoolId": { "Ref": "UserPool" },
    "ClientName": { "Fn::Sub": "acme-finance-${Environment}" },
    "GenerateSecret": true,
    "AllowedOAuthFlows": ["client_credentials"],
    "AllowedOAuthFlowsUserPoolClient": true,
    "AllowedOAuthScopes": ["accounts/read"],
    "SupportedIdentityProviders": ["COGNITO"],
    "AccessTokenValidity": { "Ref": "AccessTokenValidityMinutes" },
    "TokenValidityUnits": { "AccessToken": "minutes" }
  }
}
```

Add two outputs so the credentials can be retrieved:

```json
"AcmeFinanceClientId": {
  "Value": { "Ref": "ConsumerAcmeFinance" }
},
"GetAcmeFinanceSecret": {
  "Value": { "Fn::Sub": "aws cognito-idp describe-user-pool-client --user-pool-id ${UserPool} --client-id ${ConsumerAcmeFinance} --query UserPoolClient.ClientSecret --output text" }
}
```

⚠️ **Never change the logical ID of an existing consumer.** CloudFormation would
delete the old client and create a new one, with a new client ID and secret —
breaking that consumer immediately.

## 2. Raise a pull request

The diff is the access request. Worth recording in the description who asked,
who approved, and what they need access to.

## 3. Update the stacks

Run the update in **bede-identity**, once per environment the consumer needs:

- `bede-api-cognito-dev`
- `bede-api-cognito-uat`
- `bede-api-cognito-prod`

Only add the consumer to environments they actually need. A dev-only integrator
should not appear in the prod stack.

## 4. Issue the credentials

- Take the client ID from the stack outputs
- Run the `GetAcmeFinanceSecret` command to retrieve the secret
- Send both **through a password vault**, not email or chat

The consumer then calls `POST /auth/generate-token` with those credentials to
obtain a JWT.

---

## Removing a consumer

Delete the resource and its outputs, raise a pull request, update the stacks.
CloudFormation deletes the app client and any token issued to it stops working
within the hour, once the current token expires.

## Checking who has access

Read `stacks/01-cognito.json`. That is the list.

Each consumer's `client_id` also appears in the Lambda and API Gateway logs on
every call, so usage is attributable per consumer.

---

## Adding a new API, not a new consumer

**If the new API reads account data**, it reuses the `accounts/read` scope.
Nothing changes here — add a route to the API Gateway stack with the same
`AuthorizationScopes`. Existing tokens work immediately.

**If the new API needs a different permission** — writing rather than reading,
or a different domain such as payments — then:

1. Add the scope to the existing resource server, or add a new resource server
2. Add that scope to the `AllowedOAuthScopes` of whichever consumers should have
   it. A consumer not granted a scope cannot obtain a token for it
3. Set the new route's `AuthorizationScopes` in the API Gateway stack

Scopes are the permission boundary. Reusing one scope for everything means any
consumer can call any API, which is worth avoiding as the number of endpoints
grows.
