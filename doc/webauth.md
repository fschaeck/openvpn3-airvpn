# Internal TODO/Changes
(This section will be removed before publishing publicly)

Major Changes to current behaviour:

- internal webview needs extra parameter to start hidden instead of default
- HEAD request is on /profile instead of / since that is easier to implement for
  third party.
- `retryStartupEvents` not documented since it seems not required and also seems
  like a workaround for a OpenVPN Connect bug
- added tls-cryptv2=1 parameter to web based profile generation 
- Website should also report ovpn-webauth without ?embedded to allow client without
  internal browser support to craft a url to open in an external browser
  
TODO:
- is `/profile` a good entry point? Should it something more ovpn specific to avoid
  clashes like `/openvpn-profile/`?
- do we need hidden initial webview for profile download, and yes why?
- Should we replace the ?embbeded=true with a HTTP header/User-agent? 

# Contents of this documents
This document describes various web based protocols that used with OpenVPN. Also they
are not part of core protocol of OpenVPN, interoperability between products building
on OpenVPN is desirable.

This document has a three parts: The first revoles web based authentication during
connection which is used when a web based authentication should be done after successful 
client certificate authentication during connect. The second describes standardised 
endpoints for clients to retrieve a client profile. And final third section describes
a simpler REST based interface that OpenVPN Connect Client and Access Server use to
download profiles to the client.

# OpenVPN web auth protocol during Connect

This document describes the assumption and what client and server should implement
to facilitate a web based second factor login for OpenVPN.

## Triggering web based authentication

To trigger web based authentication the client needs to signal its ability with `IV_SSO` and
the server needs to send the url to the client. The details are documented in
https://github.com/OpenVPN/openvpn/blob/master/doc/management-notes.txt
and are outside the scope of this document.


## Recieving a OPEN_URL request

When the client receives an `OPEN_URL` request to continue the authentication via web based
authentication the client should directly open the web page or prompt the user to open
the web page. This can be either can in an internal browser windows/webview or open the
web page in an external browser. A web based login should be able to handle both cases.

There is a special "initially hidden" mode that is explained in the internal webview section.

## Auth-token usage

The backend should try to minimise the number of web url open request to a minimum to
avoid issues with not being to reach the web portal on reconnect and also to avoid
user interaction, especially when the client opens the web page in an external browser.
For this reason the OpenVPN server should try to avoid reauthenticating a user when 
they reconnect. To avoid the reauthentication via web on the first connect, the server
sould send a `auth-token` in the push reply. This keeps the authenticated login session 
independent of the VPN connection. The `auth-gen-token` of OpenVPN can enabled to use these
generate tokens that are valid for a certain time or a custom auth-token can be implemented.

# Internal webview integration API

When the web login process is using ann internal browser in an application the API
allows tigher integration of the login 

## Initial hiding of a webview

There are situations where initially hiding the webview is desirable. Mainly when relying
on persistent storage/cookies on the client webview to determine if user input is required
or not. Starting all web auth hidden would break web login pages that are not specifically
designed to work with OpenVPN. Therfore this feature must be implemented strictly as opt-in.
To enable this mode the url should have the parameter `OVPN_WEBVIEW_HIDDEN=1` as paramter in
the url. When a client implements this mode it must also implement the State API to allow 
the webpage to unhide the webview. A web login must not depend on the feature being present. 

## State API

This API is optional and may not even be possible to implement, e.g.
when the user is directly send to an external identity provider that does not
have specific OpenVPN methods

To communicate the current state of the web based login with an application that 
implements an internal webview, the web page should use a JavaScript mechanism based on 
[postMessage](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage). 

The weppage should send events with either appEvent.postMessage if appEvent exists and
window.parent.postMessage otherwise. The data is a JSON dictonary with `type` and an optional
`data` key if needed.

    appEvent.postMessage({"type": <event_string>, "data": <any>})

The approach of first trying to use `appEvent` and otherwise `window.parent` is to maximise
compatibility with various webview implementations. 
  
The events that are defined are:

For events where `data` is not defined, the data is reserved and a client should ignore it when present.

- `ACTION_REQUIRED`: This signal to the client that a user input is required. 

  If the webview is currently hidden, the webview needs to be shown to the user. 
  
  This event can be ignored if the client does not implement the hidding initial webview.
  
- `CONNECT_SUCCESS` and `CONNECT_FAILED`: The web login process was successful/failed. For the logic of the 
   application this state is just informal and can be shown in the VPN connection progress. The real 
   success/failure condition is determined by the VPN connection succeeding/failing.
   
   The application should close the webview on receiving this event.
   
-  `LOCATION_CHANGE`: This notifies the internal webview to change the title to the provided title as
   string in `data`, for example
    
       {"type": "LOCATION_CHANGE", "data": "Flower power VPN login step 37/420"}
       
    Web pages also need to implement the traditional changing of the web page to change the title when
    the page is opening in an external browser.
       
    TODO: is this event really necessary? Also is this optional or required from Connect App?
    
## Certificate checks
When doing this way, the app needs to be aware that the server might use a custom CA ceritficate
and should offer the usual way of presenting an unknown certificate and allow the user to accept
it.  The ovpn profile can include a certificate that should be implicitly trusted:

    # OVPN_ACCESS_SERVER_WEB_CA_BUNDLE_START
    # -----BEGIN CERTIFICATE-----
    # [...]
    # -----END CERTIFICATE-----
    # OVPN_ACCESS_SERVER_WEB_CA_BUNDLE_STOP


# Web based profile download

This API is intended to provide a uniform way for client to initiate the download of a profile
and have the login/download process performed through a webview. This API makes mainly sense
if implemented in a app since with an external browser it is just a normal web session that
concludes with a profile download. The content type of a ovpn profile should be 
`application/x-openvpn-profile`, so the handler of the installed application will be shown for
 opening the downloaded profile, especially on mobile platforms.
 
## Detection of web based profile download support

The classic way of a downloading profiles is outlined in the next section. To determinine
what method has to be used to download a profile from a server, the client should do 
a `HEAD` or `GET` request to `https://servername/profile`. If the response contains a header
`Ovpn-WebAuth` with any value, the web based method should be used.

In general websites should also report ovpn-webauth without ?embedded parameter to allow
clients without internal browser support to craft a url to open in an external browser that
contains the additional parameters like deviceID and tls-cryptv2 support.

**TODO**: **Do we need the hidden method here? If yes how do we communicate this**

To start the web based method the client should load the url
  
  https://servername/profile
  
with the following optional parameters:

- `deviceID`    unique device ID, must be identical with the ID provided with `IV_HWADDR` if provided   
- `deviceModel`	model of connected device
- `deviceOS`	OS version of connected device
- `appVersion`	Version of openvpn client 
- `embedded`    Should be set to true if the page is opened by the internal webbrowser 
                and supports the state API
- `tls-cryptv2`  Should be set to 1 if the client supports tls-crypt-v2

# State API

The state API is identical to the API used during connection. The web page can
also use the `LOCATION_CHANGED` event and additional should provide the following
event.

 - `PROFILE_DOWNLOAD_FAILED`: The process in the web page to download the profile was unsucessful. 
    The client should close the webview.
 - `PROFILE_DOWNLOAD_SUCCESS`: Download a of profile worked. The client should import the profile
     in the next step. `data` will contain the profile as json object with `profile` and `title` as
     keys. `profile` will contain the ovpn profile and title will have a suggested title
     for the new profile:
     
       {"type": "PROFILE_DOWNLOAD_SUCCESS", "data": {"profile": "<.ovpn profile>", "title": "<title>"}}
   
# Rest profile download API
REST is a simple and more lightweight interface to download profiles.
This is currently mainly implemented by OpenVPN Access Server API and 
Connect clients but can also be implemeneted by other server and clients.
The endpoint is https://servername/rest/methodname

## General API

Access server calls profile that require username and password Userlogin
profile and profile without Autologin. This is also replicated in the
method name (`rest/GetAutologin` or `rest/GetUserlogin`) in the request URL. 
The configuration file is returned as a text/plain HTTP document (Note: this
in contrast to the right type application/x-openvpn-profile). Credentials 
are specified using HTTP Basic Authentication.  The REST API is implemented
through an SSL web server. The client must do the normal SSL certificate check
but should also allow a user to pin a self-signed certificate.

The client should also indicate support supported features that influence
profile generation and cannot be negiotiated by the VPN protocol. 
Currently only TLS Crypt V2 support indicated by adding a tls-cryptv2=1
to the request

Typically a client app will present a username and password input field and
checkbox to enable/disable autologin. 

To get the Autologin configuration using curl:

    $ curl -u USERNAME:PASSWORD https://asdemo.openvpn.net/rest/GetAutologin

To get the Userlogin (requiring authentication) configuration using curl, indicating
that the client supports tls crypt v2

    $ curl -u USERNAME:PASSWORD https://asdemo.openvpn.net/rest/GetUserlogin?tls-cryptv2=1

Additional for User login profiles headers to initiate a direct VPN are provided to avoid
double 2FA login in a short amount of time:

    VPN-Session-User: base64_encoded_user
    VPN-Session-Token: base64_encoded_pw
    
These paramter are optional but should be used as VPN session user and token when 
initing a connection directly after downloading a profile.




## Error reporting with Rest API

Internally OpenVPN Access server used XMLRPC when this and the current 
implementation did not properly account for this, so error reporting is
done with xml replies. This is an unfortunate design/implemntation detail.
So the rest error replies look more like XMLRPC errors than rest error.
They carry a HTTP error status.

Authentication failed (bad USERNAME or PASSWORD):

    <?xml version="1.0" encoding="UTF-8"?>
    <Error>
      <Type>Authorization Required</Type>
      <Synopsis>REST method failed</Synopsis>
      <Message>AUTH_FAILED: Server Agent XML method requires authentication (9007)</Message>
    </Error>

User does not have permission to use an Autologin profile:

    <?xml version="1.0" encoding="UTF-8"?>
    <Error>
      <Type>Internal Server Error</Type>
      <Synopsis>REST method failed</Synopsis>
      <Message>NEED_AUTOLOGIN: User 'USERNAME' lacks autologin privilege (9000)</Message>
    </Error>

User is not enrolled through the WEB client yet:

    <?xml version="1.0" encoding="UTF-8"?>
    <Error>
      <Type>Access denied</Type>
      <Synopsis>REST method failed</Synopsis>
      <Message>You must enroll this user in Authenticator first before you are allowed to retrieve a connection profile. (9008)</Message>
    </Error>

## Challenge/response authentication
The challenge/response protocol for the Rest web api mirrors the approach
taken by the old (non using AUTH-PENDING,cr-response) challenge/response
of the OpenVPN protocol.

When the server issues a challenge to the authentication
request. For example suppose we have a user called 'test' and a password
of 'mypass".  Get the OpenVPN config file:

    curl -u test:mypass https://ACCESS_SERVER/rest/GetUserlogin

But instead of immediately receiving the config file,
we might get a challenge instead:

    <Error>
      <Type>Authorization Required</Type>
      <Synopsis>REST method failed</Synopsis>
      <Message>CRV1:R,E:miwN39AlF4k40Fd8X8r9j74FuOoaJKJM:dGVzdA==:Turing test: what is 1 x 3? (9007)</Message>
    </Error>


a challenge is indicated by the "CRV1:" prefix in the <Message> (meaning
Challenge Response protocol Version 1).  The CRV1 message is formatted
as follows:

    CRV1:<flags>:<state_id>:<username_base64>:<challenge_text>

`flags` : a series of optional, comma-separated flags:
  - `E` : echo the response when the user types it
  - `R` : a response is required

`state_id`: an opaque string that should be returned to the server
along with the response.

`username_base64` : the username formatted as base64

`challenge_text` : the challenge text to be shown to the user

After showing the challenge_text and getting a response from the user
(if `R` flag is specified), the client should resubmit the REST
request with the `USERNAME:PASSWORD` field in the HTTP header set
as follows:

    <username decoded from username_base64>:CRV1::<state_id>::<response_text>

Where state_id is taken from the challenge request and `response_text`
is what the user entered in response to the `challenge_text`.
If the `R` flag is not present, `response_text` may be the empty
string.

Using curl to respond to the turing test given in the example above:

    curl -u "test:CRV1::miwN39AlF4k40Fd8X8r9j74FuOoaJKJM::3" https://ACCESS_SERVER/rest/GetUserlogin

If the challenge response (In this case '3' in response to the turing
test) is verified by the server, it will then return the configuration
file per the GetUserlogin method.