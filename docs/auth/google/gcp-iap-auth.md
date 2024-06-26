---
id: gcp-iap-auth
title: Google Identity-Aware Proxy Provider
sidebar_label: Google IAP
# prettier-ignore
description: Adding Google Identity-Aware Proxy as an authentication provider in Backstage
---

Backstage allows offloading the responsibility of authenticating users to the
Google HTTPS Load Balancer & [IAP](https://cloud.google.com/iap), leveraging the
authentication support on the latter.

This tutorial shows how to use authentication on an IAP sitting in front of
Backstage.

It is assumed an IAP is already serving traffic in front of a Backstage instance
configured to serve the frontend app from the backend.

## Configuration

Let's start by adding the following `auth` configuration in your
`app-config.yaml` or `app-config.production.yaml` or similar:

```yaml
auth:
  providers:
    gcp-iap:
      audience: '/projects/<project number>/global/backendServices/<backend service id>'
      jwtHeader: x-custom-header # Optional: Only if you are using a custom header for the IAP JWT
      signIn:
        resolvers:
          # typically you would pick one of these
          - resolver: emailMatchingUserEntityProfileEmail
          - resolver: emailLocalPartMatchingUserEntityName
          - resolver: emailMatchingUserEntityAnnotation
```

The full `audience` value can be obtained by visiting your [Identity-Aware Proxy Google Cloud console](https://console.cloud.google.com/security/iap), selecting your project, finding your Backend Service to proxy, clicking the 3 vertical dots then "Get JWT Audience Code", and copying from the resulting popup, which will look similar to the following:

![Identity-Aware Proxy JWT Audience Code popup](../../assets/auth/gcp-iap-jwt-audience-code-popup.png)

This config section must be in place for the provider to load at all. Now let's
add the provider itself.

### Resolvers

This provider includes several resolvers out of the box that you can use:

- `emailMatchingUserEntityProfileEmail`: Matches the email address from the auth provider with the User entity that has a matching `spec.profile.email`. If no match is found it will throw a `NotFoundError`.
- `emailLocalPartMatchingUserEntityName`: Matches the [local part](https://en.wikipedia.org/wiki/Email_address#Local-part) of the email address from the auth provider with the User entity that has a matching `name`. If no match is found it will throw a `NotFoundError`.
- `emailMatchingUserEntityAnnotation`: Matches the email address from the auth provider with the User entity where the value of the `google.com/email` annotation matches. If no match is found it will throw a `NotFoundError`.

> Note: The resolvers will be tried in order, but will only be skipped if they throw a `NotFoundError`.

If these resolvers do not fit your needs you can build a custom resolver, this is covered in the [Building Custom Resolvers](../identity-resolver.md#building-custom-resolvers) section of the Sign-in Identities and Resolvers documentation.

## Backend Changes

There is a module for this provider that you will need to add to your backend.

First you'll want to run this command to add the module:

```sh
 yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-gcp-iap-provider
```

Then you will need to add this to your backend:

```ts title="packages/backend/src/index.ts"
const backend = createBackend();

backend.add(import('@backstage/plugin-auth-backend'));
/* highlight-add-start */
backend.add(import('@backstage/plugin-auth-backend-module-gcp-iap-provider'));
/* highlight-add-end */
```

### Legacy Backend Changes

If you are still using the legacy backend you will need to make the changes outlined here.

This provider is not enabled by default in the auth backend code, because besides the config section above, it also needs to be given one or more callbacks in actual code as well as described below.

Add a `providerFactories` entry to the router in
`packages/backend/src/plugins/auth.ts`.

```ts title="packages/backend/src/plugins/auth.ts"
import { providers } from '@backstage/plugin-auth-backend';
import { stringifyEntityRef } from '@backstage/catalog-model';

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  return await createRouter({
    logger: env.logger,
    config: env.config,
    database: env.database,
    discovery: env.discovery,
    providerFactories: {
      'gcp-iap': providers.gcpIap.create({
        // Replace the auth handler if you want to customize the returned user
        // profile info (can be left out; the default implementation is shown
        // below which only returns the email). You may want to amend this code
        // with something that loads additional user profile data out of e.g.
        // GSuite or LDAP or similar.
        async authHandler({ iapToken }) {
          return { profile: { email: iapToken.email } };
        },
        signIn: {
          // You need to supply an identity resolver, that takes the profile
          // and the IAP token and produces the Backstage token with the
          // relevant user info.
          async resolver({ profile, result: { iapToken } }, ctx) {
            // Somehow compute the Backstage token claims. Just some sample code
            // shown here, but you may want to query your LDAP server, or
            // GSuite or similar, based on the IAP token sub/email claims
            const id = iapToken.email.split('@')[0];
            const sub = stringifyEntityRef({ kind: 'User', name: id });
            const ent = [
              sub,
              stringifyEntityRef({ kind: 'Group', name: 'team-name' }),
            ];
            return ctx.issueToken({ claims: { sub, ent } });
          },
        },
      }),
    },
  });
}
```

Now the backend is ready to serve auth requests on the
`/api/auth/gcp-iap/refresh` endpoint. All that's left is to update the frontend
sign-in mechanism to poll that endpoint through the IAP, on the user's behalf.

## Frontend Changes

It is recommended to use the `ProxiedSignInPage` for this provider, which is
installed in `packages/app/src/App.tsx` like this:

```tsx title="packages/app/src/App.tsx"
/* highlight-add-next-line */
import { ProxiedSignInPage } from '@backstage/core-components';

const app = createApp({
  /* highlight-add-start */
  components: {
    SignInPage: props => <ProxiedSignInPage {...props} provider="gcp-iap" />,
  },
  /* highlight-add-end */
  // ..
});
```

See the [Sign-In with Proxy Providers](../index.md#sign-in-with-proxy-providers) section for more information.
