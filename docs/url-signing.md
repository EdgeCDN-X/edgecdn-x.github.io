# URL Signing

The EdgeCDN-X Platform enables private content to be served to client via URL Signatures. It is possible to Signature a single URL for a single resource, or an URL prefix, which covers multiple files under a certain path.

## Signed URLs for single object
The object must be signed with a valid URL. The URL signate consists of the following headers:
* `EX-Expires` - Expires timestamp. Till when the URL is valid
* `EX-KeyName` - Name of the key to used to sign this request
* `EX-Sign` - Signature

The ordering of the Query Parameters must be preserved. e.g.:

```
https://resource.cdn.edgecdnx.com/my/favourite/file?user-query1=yes&EX-Expires=1861631432&EX-KeyName=key2&EX-Sign=8e36e0d2b6daeb9b2f1a0d12137c6de07894abac7766684ef8faa7732d6d58dc
```

It is possible to use user defined queries too, but they must be stored before the CDN specific Query Params

URL to be signed is:
`https://resource.cdn.edgecdnx.com/my/favourite/file?user-query1=yes&EX-Expires=1861631432&EX-KeyName=key2`

```go
	mac := hmac.New(sha256.New, []byte(key))
	mac.Write(payload)
	signature := mac.Sum(nil)
```

If the signature matches the signature in the Query Params 200 success is returned. Since we only validate a single URL in this case, we do not return any session cookie

## Signed URLs for Prefixes
Prefixes can be signed with an URL for live (HLS or DASH) streaming. In such cases the initial request must be signed including the `EX-UrlPrefix` Header.

* `EX-UrlPrefix` - Base64 URLEncoded Prefix
* `EX-Expires` - Expires timestamp. Till when the URL is valid
* `EX-KeyName` - Name of the key to used to sign this request
* `EX-Sign` - Signature

e.g.:
```
http://lsansdkaglh23as.cdn.edgecdnx.com/nice/movie/here/index.m3u8?EX-UrlPrefix=aHR0cDovL2xzYW5zZGthZ2xoMjNhcy5jZG4uZWRnZWNkbnguY29tL25pY2UvbW92aWUvaGVyZS8=&EX-Expires=1861631432&EX-KeyName=key2&EX-Sign=8e36e0d2b6daeb9b2f1a0d12137c6de07894abac7766684ef8faa7732d6d58dc
```

The Encoded URLPrefix query param contains the following string:
`http://lsansdkaglh23as.cdn.edgecdnx.com/nice/movie/here/`

This will first match if the current request matches the Prefix. If it does then the URL to be signed is built as:
```
http://lsansdkaglh23as.cdn.edgecdnx.com/nice/movie/here/index.m3u8?EX-UrlPrefix=aHR0cDovL2xzYW5zZGthZ2xoMjNhcy5jZG4uZWRnZWNkbnguY29tL25pY2UvbW92aWUvaGVyZS8=&EX-Expires=1861631432&EX-KeyName=key2
```

Please note it is not possible to set User Defined Query Params for URLPrefixes.

If the URL is valid a session cookie is returned, which embeds the following data:

```go
type CookieBody struct {
	KeyName   string `json:"keyName"`
	Expires   int64  `json:"expires"`
	Service   string `json:"service"`
	URLPrefix string `json:"url,omitempty"`
}
```

This object is marshalled, and then again signed with the Client's key defined in the KeyName field. The object is base64URLEncoded and the signature is appended after a dot as base64URLEncoded too.

e.g.:
```
ex-sec-session eyJrZXlOYW1lIjoia2V5MiIsImV4cGlyZXMiOjE3NTE5ODI1MTQsInNlcnZpY2UiOiJsc2Fuc2RrYWdsaDIzYXMuY2RuLmVkZ2VjZG54LmNvbSIsInVybCI6ImFIUjBjRG92TDJ4ellXNXpaR3RoWjJ4b01qTmhjeTVqWkc0dVpXUm5aV05rYm5ndVkyOXRMMjVwWTJVdmJXOTJhV1V2YUdWeVpTOD0ifQ==.Z69misZcfNtGAvI6YfyIFSvFOww_JTyf-wmOkJuF7uo=
```

This cookie has the PrefixSet according to the URLPrefix, so the client sends this session cookie alongside the subsequent requests for media chunks.


## Secure Cookies
The secure cookies in the previous step is sent along the Requests. The session cookie is first parsed and the URLPrefix is base64Decoded 

```go
func DecodeCookie(cookie string) (ck.CookieBody, []byte, error) {
	parts := strings.Split(cookie, ".")
	if len(parts) != 2 {
		logger.Debug("Invalid cookie format", zap.String(EX_COOKIE_NAME, cookie))
		return ck.CookieBody{}, []byte{}, errors.New("Invalid cookie format")
	}

	payload, err := base64.URLEncoding.DecodeString(parts[0])

	if err != nil {
		logger.Debug("Invalid cookie payload", zap.Error(err))
		return ck.CookieBody{}, []byte{}, errors.New("Invalid cookie payload")
	}

	signature, err := base64.URLEncoding.DecodeString(parts[1])
	if err != nil {
		logger.Debug("Invalid cookie signature", zap.Error(err))
		return ck.CookieBody{}, []byte{}, errors.New("Invalid cookie Signature")
	}

	cookiePayload := ck.CookieBody{}
	err = json.Unmarshal(payload, &cookiePayload)

	if err != nil {
		logger.Debug("Invalid cookie payload", zap.Error(err))
		return ck.CookieBody{}, []byte{}, errors.New("Json unmarshal error")
	}

	urlPrefix, err := base64.URLEncoding.DecodeString(cookiePayload.URLPrefix)
	if err != nil {
		logger.Debug("Invalid URLPrefix in cookie payload", zap.Error(err))
		return ck.CookieBody{}, []byte{}, errors.New("Invalid URLPrefix in cookie payload")
	}

	cookiePayload.URLPrefix = string(urlPrefix)

	return cookiePayload, signature, nil
}
```

The Session Cookie Object is again signed with the Key stored in the Clients configuration. If the signature is correct and not expired, the request is valid.

If the signature is about to expire (20 minutes before TTL) a new cookie is issued and is passed back to the client.


# Signing URLs
Examples

