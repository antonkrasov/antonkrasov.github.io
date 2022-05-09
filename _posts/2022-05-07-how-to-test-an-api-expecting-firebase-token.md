---
title: How to test an API expecting Firebase token
date: 2022-05-07 19:50:00 -0700
categories: [Tech, Tutorial]
tags: [firebase, postman, backend]
---

Let's say you are developing a backend API and you are protecting it with firebase tokens. This saves lots of development time on the frontend (mobile or web) just because Firebase Auth SDK does all the heavy lifting for token generation, refresh, etc, as well as on the backend for the exact same reason. But when you want to test this API with Postman or any other REST client you may face a problem, how to get the same token as the client receives from the Firebase SDK?

One of the ways to do it is to launch a client app and grab a token from there. Personally, I wasted lots of time doing this.

## Better Solution

There is a way better option for doing it, and I was surprised by how easy it is. All we need to do is to use Firebase Auth REST API ([https://firebase.google.com/docs/reference/rest/auth](https://firebase.google.com/docs/reference/rest/auth)). It provides simple API calls we can use to manage our Firebase users as well as tokens.

## Example

Let’s see by example how we can do it, I created a sample firebase project and deployed a simple cloud function:

```jsx
const functions = require("firebase-functions");
const serviceAccount = require("./service-account.json");
const { initializeApp, cert } = require('firebase-admin/app');
const { getAuth } = require('firebase-admin/auth');

const app = initializeApp({
    credential: cert(serviceAccount)
})
const auth = getAuth(app)

exports.protectedEndpoint = functions.https.onRequest(async (request, response) => {
    try {
        const xApiToken = request.headers['x-api-token']
        const decodedToken = await auth.verifyIdToken(xApiToken)
        response.send({ decodedToken });
    } catch (ex) {
        response.status(403).json({
            error: "'x-api-token' header is required!"
        })
    }
});
```

Let's try to call it via cURL:

```bash
➜  ~ curl https://us-central1-givemefirebasetoken-sample.cloudfunctions.net/protectedEndpoint
{"error":"'x-api-token' header is required!"}%
```

As expected we got our error, because `auth.verifyIdToken()` failed.

Now let’s try to get the token via Rest API. The first thing we will need is an API_KEY for our Firebase project. The easiest way to get it is to create a Web app in your firebase console. Next, you’ll see your API_KEY in it. Here is an example from my sample project:

![Screen Shot 2022-05-07 at 8.15.32 PM.png](/assets/img/2022-05-08-how-to-test-an-api-expecting-firebase-token/Screen_Shot_2022-05-07_at_8.15.32_PM.png)

After we have our `apiKey` we can start making calls to the firebase, link to the detailed API reference: [https://firebase.google.com/docs/reference/rest/auth](https://firebase.google.com/docs/reference/rest/auth)

Personally I use 3 methods:

- Sign up with email / password ([https://firebase.google.com/docs/reference/rest/auth#section-create-email-password](https://firebase.google.com/docs/reference/rest/auth#section-create-email-password))
- Sign in with email / password ([https://firebase.google.com/docs/reference/rest/auth#section-sign-in-email-password](https://firebase.google.com/docs/reference/rest/auth#section-sign-in-email-password))
- Delete account ([https://firebase.google.com/docs/reference/rest/auth#section-delete-account](https://firebase.google.com/docs/reference/rest/auth#section-delete-account))

`Delete` is useful if we want to create a new user each time, we can use it in our autotests. So our flow would be like this:

1. Sign Up with email/password.
2. Run API tests.
3. Delete user.

Ok, let’s create a new user with API request:

```bash
curl --location --request POST 'https://identitytoolkit.googleapis.com/v1/accounts:signUp?key=AIzaSyAHV1eUu8EqCSUeycofXqfL******' \
--header 'Content-Type: application/json' \
--data-raw '{
    "email": "anton.krasov@gmail.com",
    "password": "******",
    "returnSecureToken": true
}'
```

Our response should look like this:

```bash
{
  "kind": "identitytoolkit#SignupNewUserResponse",
  "idToken": "eyJhbGciOiJSUzI1NiIsImtpZ....gkULJig1ypXaA",
  "email": "anton.krasov@gmail.com",
  "refreshToken": "AIwUaOkfNuMg8Kdatx-Hy8_5Ti0sm....-dKiN0CGkDfX1ch5f69g5Y5K",
  "expiresIn": "3600",
  "localId": "aoiC7bM0v....ueGMRegf1k2"
}
```

`idToken` is exactly the same token we will receive from our Firebase Auth SDK.

`localId` is our new user Id.

Just remember that you need to have `Email/Password` auth provider enabled in your `Authentication` settings:

![Screen Shot 2022-05-07 at 9.33.33 PM.png](/assets/img/2022-05-08-how-to-test-an-api-expecting-firebase-token/Screen_Shot_2022-05-07_at_9.33.33_PM.png)

Now, we have an `idToken` let’s try to execute our sample call again:

```bash
curl --location --request GET 'https://us-central1-givemefirebasetoken-sample.cloudfunctions.net/protectedEndpoint' \
--header 'x-api-token: eyJhbGciOiJSUzI1NiIsImtpZ....09HlrF49MRWo9qq0_c6UiMHdBZUavNXyuFyjC85dpSxJrNtJSwH5OQa22dIwdbNmfMqz6ljAivIkjw5aX2GrF5ViFCdp4Tew5iw'
```

and we got our `decodedToken` in response:

```bash
{"decodedToken":{"iss":"https://securetoken.google.com/givemefirebasetoken-sample","aud":"givemefirebasetoken-sample","auth_time":1651977397,"user_id":"aoiC7bM0vFQTMPj24ueGMRegf1k2","sub":"aoiC7bM0vFQTMPj24ueGMRegf1k2","iat":1651977397,"exp":1651980997,"email":"anton.krasov@gmail.com","email_verified":false,"firebase":{"identities":{"email":["anton.krasov@gmail.com"]},"sign_in_provider":"password"},"uid":"aoiC7bM0vFQTMPj24ueGMRegf1k2"}}
```

## Leverage Postman Environments

Now we know the API calls we need to execute to get the user’s info, let’s leverage postman variables and environments ([https://learning.postman.com/docs/sending-requests/variables/](https://learning.postman.com/docs/sending-requests/variables/)) to store the token and use it with our own backend API calls.

First thing, let’s create env and setup variables we need:

![Screen Shot 2022-05-07 at 9.40.33 PM.png](/assets/img/2022-05-08-how-to-test-an-api-expecting-firebase-token/Screen_Shot_2022-05-07_at_9.40.33_PM.png)

Variables are pretty self explanatory ;)

Now, let’s create our `accounts:signUp` request, and execute it:

![Screen Shot 2022-05-07 at 9.44.21 PM.png](/assets/img/2022-05-08-how-to-test-an-api-expecting-firebase-token/Screen_Shot_2022-05-07_at_9.44.21_PM.png)

Now, once we receive our token, let’s save it in our environment using `Tests` tab ([https://learning.postman.com/docs/writing-scripts/test-scripts/](https://learning.postman.com/docs/writing-scripts/test-scripts/)):

```jsx
var jsonData = JSON.parse(responseBody);
pm.expect(jsonData.idToken).to.be.a('string');
postman.setEnvironmentVariable("firebaseIdToken", jsonData.idToken);
```

On the second line, we also validate that we have `idToken` and it’s a string. This is required to prevent setting up the wrong `firebaseIdToken` env var if our request failed. Let’s say you called `accounts:signUp` request twice and got an error from the firebase saying that the user already exists.

Now, after the execution we'll have our token in our env, and we can use it in our own API calls.

![Screen Shot 2022-05-07 at 9.48.30 PM.png](/assets/img/2022-05-08-how-to-test-an-api-expecting-firebase-token/Screen_Shot_2022-05-07_at_9.48.30_PM.png)

## Automatic token retrieval

Postman has a nice feature called `pre-request scripts:` [https://learning.postman.com/docs/writing-scripts/pre-request-scripts/](https://learning.postman.com/docs/writing-scripts/pre-request-scripts/)

We can use it to automate retrieving of our token and setting of our env, and the best thing about it is that we can create a pre-request script **for the whole collection**! This means that we won't need to call any of the firebase requests manually.

Here is an example script: 

```jsx
const firebaseApiKey = pm.environment.get("firebaseApiKey")
const firebaseTestUserEmail = pm.environment.get("firebaseTestUserEmail")
const firebaseTestUserPassword = pm.environment.get("firebaseTestUserPassword")

const options = {
    url: `https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key=${firebaseApiKey}`,
    method: 'POST',
    header: {
        'Accept': '*/*',
        'Content-Type': 'application/json',
    },
    body: {
        mode: 'raw',
        raw: JSON.stringify({
            "email": firebaseTestUserEmail,
            "password": firebaseTestUserPassword,
            "returnSecureToken": true
        })
    }
};

pm.sendRequest(options, function (err, res) {
    const respData = res.json();

    pm.environment.set("firebaseIdToken", respData.idToken);
    pm.environment.set("firebaseUID", respData.localId);
});
```

---

Thank you, feel free to contact me via email: anton.krasov@gmail.com if you have any questions ;)