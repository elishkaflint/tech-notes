# AUTH NOTES

- a lot of these notes taken for this great resource: https://fusionauth.io/learn/expert-advice/oauth/modern-guide-to-oauth 
    - the code used in the article can be found here https://github.com/FusionAuth/fusionauth-example-modern-guide-to-oauth

- Important concepts
    - OAuth lets me delegate the authentication and authorization of my users to someone else
    - My application sends the user over to an OAuth server, the user logs in and the user is sent back to the application

## What are the different types of auth flow?

### Local login and registration (THIS IS WHAT WE'RE USING)
- This is where you control both the application and the auth server (you may not have written the auth server, but you control it)
- This is good if you are looking to outsource your authentication and authorization to a safe, secure and standards-friendly auth system
- This mode often feels like the application is using native forms and not delegating anything at all, which is why it's called "local login and registration". However in reality the application is delegating management to the auth server rather than handling everything itself. 
- There are no permission screens in this mode.

### Third-party login and registration
- This is where you add a page on your application which says "sign in with...github, facebook etc"
- This is good when you want to avoid storing and managing any user credentials like passwords

## What are the different types of OAuth grants (and how do they relate to the flows above)?

### Authorization Code Grant (THIS IS WHAT WE SHOULD BE USING)

- The most common grant and most secure

- It requires the interaction of a user via the browser to handle the OAuth flows above (so cannot be used for machine to machine)

- Important terminology:
    - **Authorize endpoint:** This is the location that starts the workflow and is a URL that the browser is taken to. Normally, users register or log in at this location. It is on the auth server. 
    - **Authorization code:** This is a random string of printable ASCII characters that the OAuth server includes in the redirect after the user has registered or logged in. This is exchanged for tokens by the application backend.
    - **Token endpoint:** This is an API that is used to get tokens from the OAuth server after the user has logged in. The application backend uses the authorisation code when it calls the token endpoint.

- Note how the authorization code is passed through to the browser. PKCE (Proof Key for Code Exchange - pronounced Pixy) is an optional security layer that sits on top of the Authorisation Code Grant flow where the application also generates a secret key, hashes it using SHA-256 (one way so it can't be reversed by an attacker) and sends it to the auth server. When exchanging the authorisation code for tokens, the application send the secret key, meaning an attacker with the authorisation code cannot use it to get important tokens because they cannot access the secret key.

- Implementation

    - Create page with a link to login that calls a /login endpoint in our controller

    - The /login endpoint in our controller returns a redirect to the location on the auth server that starts the Authorisation Code grant (eg. https://gds-tech-spike.auth.eu-west-2.amazoncognito.com/login?[lots of parameters])

        - The location on the auth server that starts the Authorisation Code grant takes a bunch of parameters:

            - Mandatory
                - `client_id` - this identifies that application that the user is using and which is calling the auth server
                - `redirect_uri` - this is the URL in your application to which the OAuth server will redirect the user to after they log in. This URL must be registered with the OAuth server and it must point to a controller in your app (rather than a static page), because your app must do additional work after this URL is called.
                - `response_type` - this should always be set to `code` for this grant. This tells the OAuth server you are using the Authorization Code grant.

            - Optional
                - `code` and `code_challenge` are related to PKCE
                - `state` is a parameter that is echoed back by the OAuth server, so can be useful to persist info across the OAuth workflow
                - `nonce` is used in OpenIDC flows (the nonce parameter will be included in the Id token that the OAuth server generates, we can verify that when we retrieve the Id token)
                    - These values often need to be persisted across browser requests and redirects. There are 2 storage options:
                        - server side session `req.session` 
                        - client side cookies `res.cookie` 
                            - here we can use `{httpOnly: true, secure: true}` to ensure that no malicious Javascript code in the browser can read them, we can also encrypt although limited benefit 
                            - can also encrypt, although thsi is generally not needed, especially for the state and nonce parameters since those are sent as plaintext on the redirect anyways, but if you need ultimate security and want to use cookies, this is the best way to secure these values.
            - in our working example, the output is https://gds-tech-spike.auth.eu-west-2.amazoncognito.com/login?response_type=token&client_id=3qmts8r1jafau3d3ko8qs77sv3&redirect_uri=http://localhost:3000/callback

    - User logs in

    - Auth server provides a redirect

        - OAuth server redirects the browser back to the application. The exact location of the redirect is controlled by the redirect_uri parameter we passed on the URL above. When the OAuth server redirects the browser back to this location, it will add a few parameters to the URL. These are:
            - code - this is the authorization code that the OAuth server created after the user was logged in. We’ll exchange this code for tokens.
            - state (optional) - this is the same value of the state parameter we passed to the OAuth server. This is echoed back to the application so that the application can verify that the code came from the correct location.

    - Application handles this redirect
        - We write an endpoint that receives the request and calls the token endpoint with the following parameters
                        - code - this is the authorization code we are exchanging for tokens.
                        - client_id - this is client id that identifies our application.
                        - client_secret - this is a secret key that is provided by the OAuth server. This should never be made public and should only ever be stored in your application on the server.
                        - code_verifier - this is the code verifier value we created above and either stored in the session or in a cookie.
                        - grant_type - this will always be the value authorization_code to let the OAuth server know we are sending it an authorization code.
                        - redirect_uri - this is the redirect URI that we sent to the OAuth server above. It must be exactly the same value.

                - Application receives tokens from token endpoint
                    - access_token: This is a JWT that contains information about the user including their id, permissions, and anything else we might need from the OAuth server.
                    - id_token: This is a JWT that contains public information about the user such as their name. This token is usually safe to store in non-secure cookies or local storage because it can’t be used to call APIs on behalf of the user.
                    - refresh_token: This is an opaque token (not a JWT) that can be used to create new access tokens. Access tokens expire and might need to be renewed, depending on your requirements (for example how long you want access tokens to last versus how long you want users to stay logged in)

                - Now that we have these tokens, we need to create a session for the user
                    - 1. store them in browser cookies (seems like id token can be stored less securely)
                        - when we call the backend, we don't actually need to formally send any cookies: This is one of the strengths of the cookie approach. As long as we’re calling APIs from the same domain, cookies are sent for free. If you need to send cookies to a different domain, make sure you check your CORS settings.
                    - 2. store them in the session
                        - here we might also use a browser cookie, but only to store the session id

### Implicit Grant

- generally recommended NOT to use this as it is not secure

- the reason is that the implicit grant skips the step above, where the auth server returns an authorization code, which the back end exchanges for the access/id/refresh tokens. it just sticks the access token right in the browser redirect. 
    - https://my-app.com/#token-goes-here
    - The token is added to the redirect URL after the # symbol, which means it is technically the fragment portion of the URL. What this really means is that wherever the OAuth server redirects the browser to, the access token is accessible to basically everyone.
    - Specifically, the access token is accessible to any and all JavaScript running in the browser. Since this token allows the browser to make API calls and web requests on behalf of the user, having this token be accessible to third-party code is extremely dangerous.

- The Implicit grant type was created for JavaScript apps while trying to also be easier to use than the Authorization Code grant. In practice, any benefit gained from the initial simplicity is lost in the other factors required to make this flow secure.

