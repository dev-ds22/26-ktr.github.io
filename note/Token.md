POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-length: 261
content-type: application/x-www-form-urlencoded
user-agent: google-oauth-playground

code=4%2F0AdkVLPyWi6pyyXQG_pIT5sUMQKh5qsjA5hivyDkeLXtClGabPtZGxCld8kqCD9f2C3s_Hg&redirect_uri=https%3A%2F%2Fdevelopers.google.com%2Foauthplayground&client_id=407408718192.apps.googleusercontent.com&client_secret=************&scope=&grant_type=authorization_code

HTTP/1.1 200 OK
Content-length: 555
X-xss-protection: 0
X-content-type-options: nosniff
Transfer-encoding: chunked
Expires: Mon, 01 Jan 1990 00:00:00 GMT
Vary: Origin, X-Origin, Referer
Server: scaffolding on HTTPServer2
-content-encoding: gzip
Pragma: no-cache
Cache-control: no-cache, no-store, max-age=0, must-revalidate
Date: Wed, 17 Jun 2026 03:03:50 GMT
X-frame-options: SAMEORIGIN
Alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000
Content-type: application/json; charset=utf-8

{  "access_token": "ya29.a0AT3oNZ_36JPunrmaT-V5u6ZKUrZ8t-jYW6kBKAIc_uhXkt3UxBN_cX6XskDCIHumf7FqG0dBOGrBWkpdI2wi-_XUr_1ioOPrE1yS9MOhyPbwrHwyC7x48xCvq__HRZUWFDfTwuccvIJuI-wlFFjUSHudYB18xj3ycbI-RaVeXsu7-bMHLFuvk2ZfhOdfYJYUMMTF740aCgYKAXASARISFQHGX2Mi-LVI1LNZ7MY5f9Z2O9j2Qw0206",   "refresh_token_expires_in": 604799,   "expires_in": 3599,   "token_type": "Bearer",   "scope": "[https://www.googleapis.com/auth/webmasters.readonly](https://www.googleapis.com/auth/webmasters.readonly)",   "refresh_token": "1//04Fjzt2nukQC-CgYIARAAGAQSNwF-L9IrhoMmWFPabgb9Di7bqbggoVW__EvAiIzSD1HekxiqZ8v7gqPS4Bim0V6k6MOfK616Mrw"  
}