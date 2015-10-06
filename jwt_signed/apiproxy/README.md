JWT Test proxy
================

This api proxy validates and creates JWT.  JSON Web Tokens.  
Currently JWT is in in the IETF standards track. 
https://tools.ietf.org/html/rfc7519

In short, JWT are just a special kind of OAuth v2 token.  The Oauth v2 spec...
https://tools.ietf.org/html/rfc6749#section-1.4

...says that Bearer tokens are strings that:

- are usually opaque to the client. 
- may denote an identifier used to retrieve the authorization
  information or may self-contain the authorization information in a
  verifiable manner

JWT are simply a form of the latter - authorization information
contained in a verifiable string. It's a signed string that contains a
set of claims. Something like a SAML Token, but in JSON format.

Apigee Edge doesn't currently contain "native" capability to create or
verify JWT.  This proxy shows how to use a Java callout to do those
things.



~/dev/apiproxies/pushapi -v -d -o iloveapis2015 -e test -n jwt_signed "/Users/dino/Google Drive/iloveapis2015/Advanced-Security-Extensions-1/repo/jwt_signed/apiproxy"


Invoking
========

There are two kinds of requests you can make: generate and verify. There are two types of tokens: RS256 and HS256. 

Generate a token with alg=HS256: 

    curl -i -X POST -d '' http://myorg-myenv.apigee.net/java-jwt1/generate-hs256

The response is something like this: 

{
  "jwt" : "eyJhbGciOiJIUzI1NiJ9.eyJleHAiOjE0MzI3NDY2NDgsInN1YiI6ImNvc3Rjby1qd3QtdGVzdDEiLCJpc3MiOiJodHRwOlwvXC9kaW5vY2hpZXNhLm5ldCIsImlhdCI6MTQzMjY2MDI0OH0.sNIQcPpgdZ2zaEzhdUaom9bT0Fl7fNNq2xwXpn6G_PU"
}


Generate a token with alg=RS256: 

    curl -i -X POST -d '' http://myorg-myenv.apigee.net/java-jwt1/generate-rs256


Verify a token with HS256: 

  $ curl -i -X POST http://cap250-test.apigee.net/java-jwt1/validate-hs256 \
    -d "jwt=JWT_HERE"

Response:

    {
      "jwt" : "eyJhbGciOiJIUzI1NiJ9.eyJleHAiOjE0MzI3NDg1MzYsInN1YiI6ImNvc3Rjby1qd3QtdGVzdDEiLCJpc3MiOiJodHRwOlwvXC9kaW5vY2hpZXNhLm5ldCIsImlhdCI6MTQzMjY2MjEzNn0.SLwvrIdA-kAFvScG1JJGYuOsVU-N-n7NLL9FUOEQDig",
      "claims" : "{"exp":1432748536,"sub":"costco-jwt-test1","iss":"http:\/\/dinochiesa.net","iat":1432662136}",
      "secondsRemaining" : "84976",
      "timeRemainingFormatted" : "23:36:16.091",
      "isExpired" : "false"
    }


Verify a token with RS256: 

  $ curl -i -X POST http://cap250-test.apigee.net/java-jwt1/validate-rs256 \
    -d "jwt=JWT_HERE"


Verify the example token from the OpenID Connect spec:

  curl -i -X POST http://cap250-test.apigee.net/java-jwt1/validate-openid \
    -d "jwt=eyJhbGciOiJSUzI1NiIsImtpZCI6IjFlOWdkazcifQ.ewogImlzcyI6ICJodHRwOi8vc2VydmVyLmV4YW1wbGUuY29tIiwKICJzdWIiOiAiMjQ4Mjg5NzYxMDAxIiwKICJhdWQiOiAiczZCaGRSa3F0MyIsCiAibm9uY2UiOiAibi0wUzZfV3pBMk1qIiwKICJleHAiOiAxMzExMjgxOTcwLAogImlhdCI6IDEzMTEyODA5NzAKfQ.ggW8hZ1EuVLuxNuuIJKX_V8a_OMXzR0EHR9R6jgdqrOOF4daGU96Sr_P6qJp6IcmD3HP99Obi1PRs-cwh3LO-p146waJ8IhehcwL7F09JdijmBqkvPeB2T9CJNqeGpe-gccMg4vfKjkM8FcGvnzZUN4_KSP0aAp1tOJ1zZwgjxqGByKHiOtX7TpdQyHE5lcMiKPXfEIQILVq0pc_E2DzL7emopWoaoZTF_m0_N0YzFC6g6EJbOEoRoSK5hoDalrcvRYLSrQAZZKflyuVCyixEoV9GfNQC3_osjzw2PAithfubEEBLuVVk4XUVrWOLrLl0nx7RkKU8NXNHq-rvKMzqg"

The above command uses a fixed JWK, as provided by
http://openid.net/specs/openid-connect-core-1_0.html#IDToken and verifies
it using the public key provided in that same spec. The public key
provieded by OPenID Connect is bundled into the Java callout JAR.  See the
configuration for policy JavaCallout-JWT-Parse-OpenIDConnect for details.



Commentary:
============

The Java code includes two classes, one for JWT creation, and one for parsing. 
Configuring the Java callouts is done in the policy XML, using properties. 

For example, this is how to configure the JWT creation with algorithm=HS256, which implies symmetric key HMAC signing: 

  <JavaCallout name='JavaCallout-JWT-Create'>
    <DisplayName>JavaCallout-JWT-Create</DisplayName>
    <Properties>
      <Property name="algorithm">HS256</Property>
      <Property name="key">{organization.name}</Property>

      <!-- standard claims to embed -->
      <Property name="subject">{apiproxy.name}</Property>
      <Property name="issuer">http://dinochiesa.net</Property>
      <Property name="audience">Optional-String-or-URI</Property>
      <Property name="expiresIn">86400</Property> <!-- in seconds -->

      <!-- custom claims to embed -->
      <Property name="claim_claim1">{context.var.here}</Property>
      <Property name="claim_nonce">938983j3k9-MS</Property>

      <!-- used as a component of the names of output variables -->
      <Property name="stepname">JavaCallout-JWT-Create</Property>
    </Properties>

    <ClassName>com.dinochiesa.Jwt.JWT_Creator_Callout</ClassName>
    <ResourceURL>java://JwtCallout.jar</ResourceURL>
  </JavaCallout>

All properties accept a string as a value. If enclosed in curlies, the
string is treated as a variable name, which is dereferenced to obtain the
value.

The "stepname" property is used by the callout to create names of
context variables, into which it inserts output values. The variable
names used look like jwt_STEPNAME_VARNAME. These variables are
referenced in the AssignMessage policies used to format the response.


To configure JWT creation with private key signing using an RSA key: 

  <JavaCallout name='JavaCallout-JWT-Create'>
    <DisplayName>JavaCallout-JWT-Create</DisplayName>
    <Properties>
      <Property name="algorithm">RS256</Property>

      <!-- pemfile + private-key-password} used only for algorithm = RS256 -->
      <Property name="pemfile">private.pem</Property>
      <Property name="private-key-password">deecee123</Property>

      <!-- standard claims -->
      <Property name="subject">{apiproxy.name}</Property>
      <Property name="issuer">http://dinochiesa.net</Property>
      <Property name="audience">Optional-String-or-URI</Property>
      <Property name="expiresIn">86400</Property> <!-- in seconds -->

      <!-- custom claims to embed -->
      <Property name="claim_claim1">{context.var.here}</Property>
      <Property name="claim_shoesize">9</Property>

      <!-- used as a component of the names of output variables -->
      <Property name="stepname">JavaCallout-JWT-Create</Property>

    </Properties>

    <ClassName>com.dinochiesa.Jwt.JWT_Creator_Callout</ClassName>
    <ResourceURL>java://JwtCallout.jar</ResourceURL>
  </JavaCallout>


The pemfile need not be encrypted. If it is, obviously you need to
specify the password . Despite the name of the property, the file can
be in DER format or PEM format. The class looks for the file in the
jarfile under the /resources directory.

You can also specify the PEM-encoded private key directly in the XML
configuration, using the private-key Property, like this:

<JavaCallout name='JavaCallout-JWT-Create-RS256-2' >
  <DisplayName>JavaCallout-JWT-Create-RS256-2</DisplayName>
  <Properties>
    <Property name="algorithm">RS256</Property>

    <!-- private-key and private-key-password used only for algorithm = RS256 -->
    <Property name="private-key">
    -----BEGIN RSA PRIVATE KEY-----
    Proc-Type: 4,ENCRYPTED
    DEK-Info: DES-EDE3-CBC,049E6103F40FBE84

    EZVWs5v4FoRrFdK+YbpjCmW0KoHUmBAW7XLvS+vK3BdSM2Yx/hPhDO9URCVl9Oar
    ApEZC1CxzsyRfvKDtiKWfQKdYKLccl8pA4Jj0sCxVgL4MBFDNDDEau4vRfXBv2EF
    eGVZiG0/oaGbOUI9bgPKXmDsZQ3LHM9JONTOxaBapc06Gxcj0btkkzwB/dZQVRvb
    ....
    7ZOF1UXVaoldDs+izZo5biVF/NNIBtg2FkZd4hh/cFlF1PV+M5+5mA==
    -----END RSA PRIVATE KEY-----
    </Property>
    <Property name="private-key-password">deecee123</Property>

    <!-- standard claims -->
    <Property name="subject">{apiproxy.name}</Property>
    <Property name="issuer">http://dinochiesa.net</Property>
    <Property name="audience">Optional-String-or-URI</Property>
    <Property name="expiresIn">86400</Property> <!-- in seconds -->

    <!-- custom claims -->
    <Property name="claim_primarylanguage">English</Property>
    <Property name="claim_shoesize">8.5</Property>

    <!-- used as a component of the names of output variables -->
    <Property name="stepname">JavaCallout-JWT-Create</Property>

  </Properties>

  <ClassName>com.dinochiesa.Jwt.JWT_Creator_Callout</ClassName>
  <ResourceURL>java://JwtCallout.jar</ResourceURL>
</JavaCallout>


If you specify both pemfile and private-key, the latter will be used.
Either of these properties can also reference a context variable. 

Any Property with a name that begins with claim_ is treated as a claim to
embed in the signed token. This means there are two ways to specify the
subject: "claim_sub" and "subject".  Likewise the other standard
claims. If you specify both forms, the claim_ form will apply.


To configure JWT parsing with HS256: 

  <JavaCallout name='JavaCallout-JWT-Parse'>
    <DisplayName>JavaCallout-JWT-Parse</DisplayName>
    <Properties>
      <Property name="algorithm">HS256</Property>
      <Property name="jwt">{request.formparam.jwt}</Property>
      <Property name="key">{organization.name}</Property>

      <!-- used as a component of the names of output variables -->
      <Property name="stepname">JavaCallout-JWT-Parse</Property>
    </Properties>

    <ClassName>com.dinochiesa.Jwt.JWT_Parser_Callout</ClassName>
    <ResourceURL>java://JwtCallout.jar</ResourceURL>
  </JavaCallout>

Note: it's the same jar, a different class. 

The key can specify an immediate string, or a variable name enclosed in
curlies which contains the key.  This must be the same key used to
encrypt the JWT. If decryption fails, you'll get an error message.

If you wanted to pass the JWT in an Authorization header, you could
specify {request.header.authorization} for the jwt.  The Java code
removes the Bearer prefix if it is found.

This callout sets these context variables: 

        jwt_<stepname>_claims - a json-formatted string of all claims
        jwt_<stepname>_issuer
        jwt_<stepname>_audience
        jwt_<stepname>_subject
        jwt_<stepname>_issueTime
        jwt_<stepname>_issueTimeFormatted ("yyyy-MM-dd'T'HH:mm:ss.SSSZ")
        jwt_<stepname>_expirationTime
        jwt_<stepname>_expirationTimeFormatted
        jwt_<stepname>_secondsRemaining
        jwt_<stepname>_timeRemainingFormatted   (HH:mm:ss.xxx)
        jwt_<stepname>_isExpired  (true/false)

By default, the parser verifies that the times are all valid - that the
issue time is before now, that the not-before-time is before now, and
that the expiration time is after now. The parser by default does not
verify if the issuer, the subject, and other non-standard time claims in
the JWT against any values. You must configure additional properties on
the policy to get that behavior. Like so:

  <JavaCallout name='JavaCallout-JWT-Parse'>
    <DisplayName>JavaCallout-JWT-Parse</DisplayName>
    <Properties>
      <Property name="algorithm">HS256</Property>
      <Property name="jwt">{request.formparam.jwt}</Property>
      <Property name="key">{organization.name}</Property>

      <!-- claims to verify -->
      <Property name="claim_iss">Freddie</Property>
      <Property name="claim_shoesize">9</Property>
      <Property name="claim_gender">M</Property>

      <!-- used as a component of the names of output variables -->
      <Property name="stepname">JavaCallout-JWT-Parse</Property>
    </Properties>

    <ClassName>com.dinochiesa.Jwt.JWT_Parser_Callout</ClassName>
    <ResourceURL>java://JwtCallout.jar</ResourceURL>
  </JavaCallout>


The above says to verify that the (standard claim) issuer is "Freddie",
that the custom claims shoesize and gender are 9 and M, respectively.
These all test for string equivalence. 

You must use the JSON property names. Therefore claims_iss and
claims_aud, not claims_issuer or claims_audience.

You could also insert the appropriate conditions as tests in the proxy
flow, after this parser step completes, to examine the context variables
that this parser sets, for each of the claims.


To configure JWT parsing with RS256: 

  <JavaCallout name='JavaCallout-JWT-Parse'>
    <DisplayName>JavaCallout-JWT-Parse</DisplayName>
    <Properties>
      <Property name="algorithm">RS256</Property>
      <Property name="jwt">{request.formparam.jwt}</Property>

      <!-- pemfile used only for algorithm = RS256 -->
      <Property name="pemfile">public-key.pem</Property>

      <Property name="stepname">JavaCallout-JWT-Parse</Property>
    </Properties>

    <ClassName>com.dinochiesa.Jwt.JWT_Parser_Callout</ClassName>
    <ResourceURL>java://JwtCallout.jar</ResourceURL>
  </JavaCallout>


The pemfile must be in PEM base64-encoded PKCS#8 format or PKCS#1 format. Eg:

  -----BEGIN PUBLIC KEY-----
  MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtxlohiBDbI/jejs5WLKe
  Vpb4SCNM9puY+poGkgMkurPRAUROvjCUYm2g9vXiFQl+ZKfZ2BolfnEYIXXVJjUm
  zzaX9lBnYK/v9GQz1i2zrxOnSRfhhYEb7F8tvvKWMChK3tArrOXUDdOp2YUZBY2b
  sl1iBDkc5ul/UgtjhHntA0r2FcUE4kEj2lwU1di9EzJv7sdE/YKPrPtFoNoxmthI
  OvvEC45QxfNJ6OwpqgSOyKFwE230x8UPKmgGDQmED3PNrio3PlcM0XONDtgBewL0
  3+OgERo/6JcZbs4CtORrpPxpJd6kvBiDgG07pUxMNKC2EbQGxkXer4bvlyqLiVzt
  bwIDAQAB
  -----END PUBLIC KEY-----

To specify a pem file, it must be bundled as a resource into the JAR. 

You may also specify the PEM directly in the XML configuration, using the
public-key property, like so:


  <JavaCallout name='JavaCallout-JWT-Parse-OpenIDConnect'>
    <Properties>
      <Property name="algorithm">RS256</Property>
      <Property name="jwt">{request.formparam.jwt}</Property>

      <!-- public-key used only for algorithm = RS256 -->
      <Property name="public-key">
      -----BEGIN RSA PUBLIC KEY-----
      MIIBCgKCAQEAw7Zdfmece8iaB0kiTY8pCtiBtzbptJmP28nSWwtdjRu0f2GFpajv
      WE4VhfJAjEsOcwYzay7XGN0b+X84BfC8hmCTOj2b2eHT7NsZegFPKRUQzJ9wW8ip
      n/aDJWMGDuB1XyqT1E7DYqjUCEOD1b4FLpy/xPn6oV/TYOfQ9fZdbE5HGxJUzeku
      GcOKqOQ8M7wfYHhHHLxGpQVgL0apWuP2gDDOdTtpuld4D2LK1MZK99s9gaSjRHE8
      JDb1Z4IGhEcEyzkxswVdPndUWzfvWBBWXWxtSUvQGBRkuy1BHOa4sP6FKjWEeeF7
      gm7UMs2Nm2QUgNZw6xvEDGaLk4KASdIxRQIDAQAB
      -----END RSA PUBLIC KEY-----
      </Property>

      <!-- claims to verify -->
      <Property name="claim_iss">http://server.example.com</Property>
      <Property name="claim_aud">s6BhdRkqt3</Property>
      <Property name="claim_nonce">n-0S6_WzA2Mj</Property>
      <Property name="claim_sub">248289761001</Property>

      <Property name="stepname">JavaCallout-JWT-Parse</Property>
    </Properties>

    <ClassName>com.dinochiesa.Jwt.JWT_Parser_Callout</ClassName>
    <ResourceURL>java://JwtCallout.jar</ResourceURL>
  </JavaCallout>


If you specify both the pemfile and the public-key, the public-key is used. 


The response of this example apiproxy is like this: 

{
   "jwt" : "eyJhbGciOiJSUzI1NiJ9.eyJleHAiOjE0MzI5MTMyNTQsInN1YiI6ImphdmEtand0MSIsImF1ZCI6Ik9wdGlvbmFsLVN0cmluZy1vci1VUkkiLCJpc3MiOiJodHRwOlwvXC9kaW5vY2hpZXNhLm5ldCIsImlhdCI6MTQzMjgyNjg1NH0.ZQuiqomtBFY7YcW4sBAgyyLsNlyJIMgbZuP6KbY4BwvaULldrgxQ1eq2ciqJT7sIaEdsCZL-KA3dpj0SDwLV6X0Awxu6wpMIpF2_zZSazUbQSxwVL2-3TEAH_RUxLhR5ghSWFzEi3NcOyhvYxOMUv2FM8zD6PAhFFzKlkw3EsEhHIGoNUS2s_pgqMzfmQhG-vugCB2AN5lLXAFCLZTrP_pfNxIsuP3r2J7NR__CAFZx9HOrl9gGkmHYfTpD-P6XPN7AhOvFzTQKuMnzIsRrhZ2PwiIPakpoWL8hbgyN1gour1J0ZVFNLsL3g58xLbFdKYOtPqOY8ga18Ic9zAbhZTQ",
   "claims" : "{"exp":1432913254,"sub":"java-jwt1","aud":"Optional-String-or-URI","iss":"http:\/\/dinochiesa.net","iat":1432826854}",
   "secondsRemaining" : "80855",
   "timeRemainingFormatted" : "22:27:35.831",
   "isExpired" : "false"
 }

But more importantly, the Java callout policy sets the various 
context variables which can then be tested in the proxy flow. 


