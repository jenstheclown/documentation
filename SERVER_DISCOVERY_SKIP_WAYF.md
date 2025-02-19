# Server Discovery - Skipping the "WAYF"

This builds on [SERVER_DISCOVERY](SERVER_DISCOVERY.md).

Most "Secure Internet" servers use SAML to authenticate users and have a WAYF 
(Where Are You From) in the browser to redirect users to the 
IdP of the organization chosen by the user for the actual authentication. 

As we require the user to choose an organization in the eduVPN application 
already, therefor the user should NOT be required to have to choose _again_ 
in the browser.

This document explains how the eduVPN application can be modified to not show
the organization selection (WAYF) in the browser.

**NOTE**: this ONLY applies for "Secure Internet" and NOT for "Institute 
Access"!

## OAuth

As described in [API](API.md) the application uses OAuth to obtain 
authorization to act on behalf of the user towards the API to obtain 
configurations and certificates.

This document describes a way in which the OAuth authorization URL can be 
modified to include information about which organization the user chose in the 
application.

The typical OAuth authorization URL that is generated by the OAuth library that
is used in the eduVPN looks like this:

    https://nl.eduvpn.org/php-saml-sp/login?ReturnTo=https%3A%2F%2Fnl.eduvpn.org%2Fportal%2F_oauth%2Fauthorize%3Fresponse_type%3Dcode%26client_id%3Dorg.eduvpn.app.windows%26redirect_uri%3Dhttp%3A%252f%252f127.0.0.1%3A49705%252fcallback%26scope%3Dconfig%26state%3Dn5xGPCZHVsnCh-ZBzEwhQOuXKeCmlRJz7wjsCFhOWkA%26code_challenge_method%3DS256%26code_challenge%3DV8dv96OMSvnUVUC2hoX7KW5z9KrZP_znNKnOgOAOLK8

This is what is the URL that is opened by the application in the browser.

## Including Organization Information

The application _still_ needs to use this URL, but in a modified form. The 
application needs to know three things:

1. The OAuth authorization URL, as mentioned above;
2. The `org_id` of the entry the user chose from the `organization_list.json`, 
   for example `https://idp.surfnet.nl`. All entries are required to have an 
   `org_id`;
3. The OPTIONAL `authentication_url_template` from the `server_list.json`.

In case the `authentication_url_template` exists, it MUST be used by the 
application. If it does NOT exist, the OAuth URL as show above MUST be used 
directly, just like before.

For example the `https://nl.eduvpn.org/` server in `server_list.json` has the 
following `authentication_url_template`:

    https://nl.eduvpn.org/php-saml-sp/login?ReturnTo=@RETURN_TO@&IdP=@ORG_ID@

This template contains two values that MUST be substituted: 

* `@RETURN_TO@` is replaced with the _URL encoded_ OAuth authorization URL as 
  determined before;
* `@ORG_ID@` is replaced with the _URL encoded_ `org_id`.

The URL encoding of the OAuth authorization URL is done using 
`application/x-www-form-urlencoded` type, which differs from RFC 3986 encoding
in that spaces are encoded as `+` and not `%20`. The encoded OAuth 
authorization URL as mentioned above looks like this:

    https%3A%2F%2Fnl.eduvpn.org%2Fphp-saml-sp%2Flogin%3FReturnTo%3Dhttps%253A%252F%252Fnl.eduvpn.org%252Fportal%252F_oauth%252Fauthorize%253Fresponse_type%253Dcode%2526client_id%253Dorg.eduvpn.app.windows%2526redirect_uri%253Dhttp%253A%25252f%25252f127.0.0.1%253A49705%25252fcallback%2526scope%253Dconfig%2526state%253Dn5xGPCZHVsnCh-ZBzEwhQOuXKeCmlRJz7wjsCFhOWkA%2526code_challenge_method%253DS256%2526code_challenge%253DV8dv96OMSvnUVUC2hoX7KW5z9KrZP_znNKnOgOAOLK8

The URL encoded `org_id` as used above looks like this:

    https%3A%2F%2Fidp.surfnet.nl

Using the `authentication_url_template` and replacing the values results in 
this URL:

    https://nl.eduvpn.org/php-saml-sp/login?ReturnTo=https%3A%2F%2Fnl.eduvpn.org%2Fphp-saml-sp%2Flogin%3FReturnTo%3Dhttps%253A%252F%252Fnl.eduvpn.org%252Fportal%252F_oauth%252Fauthorize%253Fresponse_type%253Dcode%2526client_id%253Dorg.eduvpn.app.windows%2526redirect_uri%253Dhttp%253A%25252f%25252f127.0.0.1%253A49705%25252fcallback%2526scope%253Dconfig%2526state%253Dn5xGPCZHVsnCh-ZBzEwhQOuXKeCmlRJz7wjsCFhOWkA%2526code_challenge_method%253DS256%2526code_challenge%253DV8dv96OMSvnUVUC2hoX7KW5z9KrZP_znNKnOgOAOLK8&IdP=https%3A%2F%2Fidp.surfnet.nl

This then becomes the URL the application needs to open in the browser instead
of the OAuth authorization URL.
