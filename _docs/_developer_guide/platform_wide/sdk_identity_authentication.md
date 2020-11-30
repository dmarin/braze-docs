---
nav_title: SDK Identity Authenticaton
page_order: 5
hidden: true
description: "Verify the identity of SDK requests"
platform:
  - ios
  - android
  - web
---

# SDK Identity Authentication

{% alert important %}
This feature is in currently in _beta_
{% endalert %}

Identity Authentication helps makes sure that no one can impersonate or create Braze users using your SDK API Keys.

## How It Works

Braze SDKs, like other client-side libraries, use a public API key to identify your app. In order to ensure that Braze only trusts requests from authenticated users, this feature allows you to provide Braze with secure, cryptographic proof. After all, your server is the only who would know whether a user is who they claim to be in your app (using cookies or other persistent session data). 

When a user logs in to your app, your server will verify the user's identity and privately generate a crypographic signature which will be passed in to the Braze SDKs. If this signature is missing, invalid, or expired, you can have Braze reject and retry requests when a new, valid signature is supplied.

In simpler terms, your app can provide proof that a user is logged in which can't be replicated by an imposter. You can then configure Braze to only trust requests which are "verified" by your application. Requests which do not contain a valid "verification" from your app will be presumed untrustworthy.

**Note**: This feature is only available for iOS, Android, and Web applications with logged-in user states.

When enabled, this feature will prevent unauthorized access to send or receive data using your app's SDK API Keys, including:

* Sending custom events, attributes, purchases, and session data
* Creating new users in your Braze App Group
* Changing standard user profile attributes
* Requesting personalized messages

## Getting Started

There are three steps needed to get started:

### Step 1: [Server-side Integration][1]

  After generating a public and private key-pair, use your private key to create a JWT (JSON Web Token) for the currently logged-in user. 

### Step 2: [SDK Integration][2]

  After enabling this feature in our SDK, you will need to pass the user's JWT Token signed in your [server-side integration][1] to the Braze SDK.

### Step 3: [Enabling in the Braze Dashboard][3]

  This feature can be enabled on an app-by-app basis, which means you can integrate this into your different applications one at a time.

## Server-Side Integration {#server-side-integration}

### Generate a Public/Private Key-pair {#generate-keys}

First, you'll need to generate a public/private key-pair. Keep your Private Key secure, and copy the public key; you'll need to paste that into the Braze Dashboard.

Here's an example of how to generate keys (but you may prefer a different approach):

```bash
openssl genrsa -out private.pem 4096
openssl rsa -in private.pem -pubout -out public.pem
```

{% alert warning %}
Remember! Keep your private keys _private_. Never expose or hard-code your private key in your app or website. Anyone who knows your private key can impersonate or create users in your Braze account.
{% endalert %}

### Generating a user's JSON Web Token {#create-jwt}

At some point in your application's lifecycle, you will need to generate a JWT (server-side) for the currently logged-in user, and provide this to your app or website. 

Typically, this logic could go wherever your app normally requests the current user's profile.

For example, your application might have a `GET /users/me` endpoint, which is requested upon succesful login, session start, and periodically throughout the session.

Before implementing, your code may look something like this:

```javascript
app.get("/users/me", (request, response) => {
  const authorization_cookie = request.cookies.authorization;
  const user = lookupUserByCookie(authorization_cookie);
  if (user === false) {
    throw new Error("You are not logged in!");
  }
  
  return response.json({
    user: user
  });
});
```

Instead, we will also sign a JSON Web Token and return it in this same endpoint's response:

```javascript
app.get("/users/me", (request, response) => {
  import jwt from "jsonwebtoken"; // use a JWT library of your choice

  const authorization_cookie = request.cookies.authorization;
  const user = lookupUserByCookie(authorization_cookie);
  if (user === false) {
    throw new Error("You are not logged in!");
  }
  
  const braze_token_payload = {
    sub: user.id // the subject should match the user ID supplied to Braze SDK when calling `changeUser`
    exp: Math.floor(Date.now() / 1000) + ONE_DAY_IN_SECONDS) // the expiration for this temporary token
  }
  const braze_token = jwt.sign(braze_token_payload, PRIVATE_KEY, { algorithm: "RS256" });

  return response.json({
    user: user,
    braze_token: braze_token // new property returned in this endpoint
  });
});
```

Now, whenever your app or website normally refreshes its user profile, your application will have access to this new `braze_token` JWT. You'll need pass this token to the Braze SDK in the next step's [SDK Integration][2].

**JWT Fields**

|Field|Required|Description|
|-----|-----|-----|
|`sub`|Yes|The "subject" should equal the User ID you supply Braze SDKs when calling `changeUser`|
|`exp`|Yes|The "expiration" of when you want this token to expire. [How long should it be?](#faq-expiration)|


## SDK Integration {#sdk-integration}

### Enable this feature in the Braze SDK.

This will add your generated JWT tokens in network requests made to Braze.

{% tabs %}
{% tab Web SDK %}
When calling `appboy.initialize`, set the optional `sdkAuthentication` property to `true`

```javascript
appboy.initialize('YOUR-API-KEY-HERE', {
    baseUrl: "YOUR-SDK-ENDPOINT-HERE",
    sdkAuthentication: true
});
```
{% endtab %}
{% tab Android SDK %}

When configuring the Appboy instance, set the `setIsSdkAuthenticationEnabled` to `true`

```java
AppboyConfig.Builder appboyConfigBuilder = new AppboyConfig.Builder()
    .setIsSdkAuthenticationEnabled(true);
Appboy.configure(this, appboyConfigBuilder.build());
```
{% endtab %}
{% tab iOS SDK %}
todo
{% endtab %}
{% endtabs %}


### Pass the current user's JWT token when calling `changeUser`. 

Typically when a user logs in or is created for the first time, your app will call Braze's `changeUser` method, which clears any previous user data. Add the JWT token [generated server-side][4] to the Braze SDK's `changeUser` method.

You can also update the JWT when refreshing the user's profile to avoid the token from expiring mid-session.

{% tabs %}
{% tab Web SDK %}

Supply the JWT Token when calling `appboy.changeUser`:

```javascript
appboy.changeUser("NEW-USER-ID", {
  sdkAuthenticationToken: "JWT-TOKEN-FROM-SERVER"
})
```

Or, when you have refreshed the user's token mid-session:

```javascript
appboy.setAuthenticationToken:("NEW-JWT-TOKEN-FROM-SERVER");
```

{% endtab %}
{% tab Android SDK %}

Supply the JWT Token when calling `appboy.changeUser`:

```java
Appboy.getInstance(this).changeUser("NEW-USER-ID", "JWT-TOKEN-FROM-SERVER");
```

Or, when you have refreshed the user's token mid-session:

```java
Appboy.getInstance(this).setSdkAuthenticationSignature("NEW-JWT-TOKEN-FROM-SERVER");
```
{% endtab %}
{% tab iOS SDK %}
todo
{% endtab %}
{% endtabs %}


### Listen for a callback function when your current JWT token is no longer valid. {#sdk-callback}

When this feature is being enforced, SDK requests will be rejected for the following reasons:

- JWT has expired
- JWT was empty
- JWT failed to verify for the Public Keys you provided

When the SDK requests fails for one of these reasons, requests will be retried periodically, and will also ask your app for a new token.

To fix any invalid tokens and continue to successfully send data to Braze, your app should request a new JWT from your server and supply Braze's SDK with this new valid token.

{% tabs %}
{% tab Web SDK %}
```javascript
appboy.onSdkAuthenticationFailure(async () => {
  const updated_jwt = await somehowRefreshToken();
  appboy.setAuthenticationToken(updated_jwt);
});
```
{% endtab %}
{% tab Android SDK %}
```java
Appboy.getInstance(this).subscribeToSdkAuthenticationFailures(errorEvent -> {
    String newToken = getNewTokenSomehow(errorEvent);
    Appboy.getInstance(getContext()).setSdkAuthenticationSignature(newToken);
});
```
{% endtab %}
{% tab iOS SDK %}
todo
{% endtab %}
{% endtabs %}


## Enabling in the Braze Dashboard {#braze-dashboard}

Once your [Server-side Integration][1] and [SDK Integration][2] are complete, you can begin to enable this feature for those specific apps.

Keep in mind, SDK requests will continue to flow as usual _until_ you set an app's Identity Authentication to **required**. 

Should anything go wrong with your integration (i.e. your app is incorrectly passing tokens to the SDK, or your server is generating invalid tokens), simply **disable** this feature in the Braze Dashboard.

### Enforcement Options {#enforcement-options}

Within the Braze Dashboard's `App Settings` page, each app can be in one of three SDK Identity Authentication states:

- **Disabled** Braze will not verify the user's authenticity. (Default Setting)
- **Optional** Braze will verify requests for logged-in users, but will only report on failures through [Currents][5]. No requests will be rejected for invalid tokens.
- **Required** Braze will verify requests for logged-in users, and will reject requests made with invalid tokens. Errors will also be sent to [Currents][5].


Using the "**Optional**" setting is a useful way to monitor the potential impact this feature will have on your app's SDK data when it comes to implementation correctness, as well as monitoring older app versions from before you had implemented this feature. 

Invalid JWT signatures will be reported in both **Optional** and **Required** states, however only the **Required** state will rejeect SDK requests causing apps to retry and request new signatures.

For more information on monitoring failed requests, see the [Currents Schema][5] for this feature.


## Frequently Asked Questions {#faq}

#### Which SDKs support Identity Authentication? {#faq-sdk-support}

Braze Web SDK (vX.Y), Android SDK (vX.Y), and iOS SDK (vX.Y) include support for this feature.

#### Can I use this feature on only some of my apps? {#faq-app-by-app}

Yes, this feature can be enabled for specific apps and doesn't need to be used on all of your apps.

#### What happens to users who are still on older versions of my app? {#faq-sdk-backward-compatibility}

When you begin to enforce this feature, requests made by older app versions will be rejected by Braze and retried by the SDKs. Once users upgrade their app to a supported version, those enqueued requests will begin to be accepted again.

If possible, you should push users to upgrade as you would for any other mandatory upgrade. Alternatively, you can keep the feature ["optional"][6] until you see that an acceptable percentage of users have upgraded.

#### What expiration should I use when generating JWT tokens? {#faq-expiration}

We recommend using the higer value of: average session duration, session cookie/token expiration, or the frequency at which your application would otherwise refresh the current user's profile. 

#### What happens if a JWT expires in the middle of a user's session?

Should a user's token expire mid-session, the SDK has a [callback function][7] it will invoke to let your app know that a new JWT token is needed to continue sending data to Braze.

#### What happens if my server-side integration breaks and I can no longer create JWTs?

If your server is not able to provide JWT tokens or you notice some integration issue, you can always disable the feature in the Braze Dashboard.

Once disabled, any pending failed SDK requests will eventually be retried by the SDK, and accepted by Braze.

#### Why does this feature use Public/Private keys instead of Shared Secrets?

When using Shared Secrets, anyone with access to that shared secret (i.e. the Braze Dashboard page) would be able to generate tokens and impersonate your end users.

Instead, we use Private Keys so that not even Braze Employees (let alone your own Dashboard users) could possibly discover your Private Keys.

[1]: #server-side-integration
[2]: #sdk-integration
[3]: #braze-dashboard
[4]: #create-jwt
[5]: #todo
[6]: #enforcement-options
[7]: #sdk-callback
