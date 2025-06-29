---
title: "I Ran Free SAST Tools on chi and Found a GHSA"
date: 2025-06-29T09:11:02-07:00
draft: false
toc: true
images:
tags:
  - go
  - cve-analysis
---
[GHSA-vrw8-fxc6-2r93](https://github.com/go-chi/chi/security/advisories/GHSA-vrw8-fxc6-2r93) is a Host Header Injection issue in the [chi framework](https://github.com/go-chi/chi) which allows arbitrary redirect to any domain while redirecting slashes. I did not receive a CVE for this finding "Given the lower impact (can't be exploited in browser or email client)" (verbatim from the maintainers) - which I can agree with to some extent. I will say though, that lower impact vulnerabilities should also get CVEs, else there would be no CVSS "low" score CVEs. My slight disagreement is probably more influenced by me wanting CVEs to my name than a technical outlook. Having said that, GHSAs are pretty recognized too, and in these parts, we take what we get.

This is somewhat a part 2 of my [previous blog](https://baishya.xyz/posts/cve-2025-30161/) about using free SAST tools to find CVEs. I was particularly attracted to looking at go codebases (although my go skills are questionable at best) due to the large number of go vulnerabilities I have assessed for various reasons at work. The workflow is basically the same as described in the previous blog - so I will skip the details here. TL;DR: clone repo, run semgrep, profit $$$ (?)

## About chi
From the README:
> chi is a lightweight, idiomatic and composable router for building Go HTTP services. It's especially good at helping you write large REST API services that are kept maintainable as your project grows and changes. chi is built on the new context package introduced in Go 1.7 to handle signaling, cancelation and request-scoped values across a handler chain.

### Chi Middleware
One of chi's powerful features is middleware—modular functions that sit between the incoming request and your handlers. Middleware in Chi allows you to add cross-cutting behavior like logging, authentication, rate limiting, or CORS handling in a clean and reusable way. Middleware functions wrap around routes or groups of routes, making them easy to apply globally or selectively. This modularity helps keep your codebase maintainable and your concerns separated.

This is very similar to other popular frameworks such as Express, next.js or Django.

### Redirect Slashes Middleware
The RedirectSlashes middleware in the Chi router is a built-in middleware that automatically normalizes request URLs by handling trailing slashes. It ensures consistent routing behavior and helps avoid issues where `/foo` and `/foo/` are treated as different routes.

## GHSA-vrw8-fxc6-2r93

I am feeling lazy, and there is no mind-bending state-of-the-art exploitation technique here, so I'll just append the report I sent to the maintainers. (With the original `RedirectSlashes` function added in since the original code has been fixed. The maintainers also added a note which has been highlighted.)

### Summary
The `RedirectSlashes` function in `middleware/strip.go` is vulnerable to host header injection which leads to open redirect.

We consider this a **lower-severity** open redirect, as it can't be exploited from browsers or email clients (requires manipulation of a Host header). [Added by maintainers]

### Details
The `RedirectSlashes` method uses the Host header to construct the redirectURL 

```go
func RedirectSlashes(next http.Handler) http.Handler {
	fn := func(w http.ResponseWriter, r *http.Request) {
		var path string
		rctx := chi.RouteContext(r.Context())
		if rctx != nil && rctx.RoutePath != "" {
			path = rctx.RoutePath
		} else {
			path = r.URL.Path
		}
		if len(path) > 1 && path[len(path)-1] == '/' {
			if r.URL.RawQuery != "" {
				path = fmt.Sprintf("%s?%s", path[:len(path)-1], r.URL.RawQuery)
			} else {
				path = path[:len(path)-1]
			}
      // Host header used to create redirect URL here
			redirectURL := fmt.Sprintf("//%s%s", r.Host, path)
			http.Redirect(w, r, redirectURL, 301)
			return
		}
		next.ServeHTTP(w, r)
	}
	return http.HandlerFunc(fn)
}
```

The Host header can be manipulated by a user to be any arbitrary host. This leads to open redirect when using the `RedirectSlashes` middleware

### PoC
Create a simple server which uses the `RedirectSlashes` middleware
```
package main

import (
	"fmt"
	"net/http"

	"github.com/go-chi/chi/v5"
	"github.com/go-chi/chi/v5/middleware" // Import the middleware package
)

func main() {
	// Create a new Chi router
	r := chi.NewRouter()

	// Use the built-in RedirectSlashes middleware
	r.Use(middleware.RedirectSlashes) // Use middleware.RedirectSlashes

	// Define a route handler
	r.Get("/", func(w http.ResponseWriter, r *http.Request) {
		// A simple response
		w.Write([]byte("Hello, World!"))
	})

	// Start the server
	fmt.Println("Starting server on :8080")
	http.ListenAndServe(":8080", r)
}
```
Run the server `go run main.go`

Once the server is running, send a request that will trigger the `RedirectSlashes` function with an arbitrary Host header
`curl -iL -H "Host: example.com" http://localhost:8080/test/`

Observe that the request will be redirected to example.com

```
curl -L -H "Host: example.com" http://localhost:8080/test/

<!doctype html>
<html>
<head>
    <title>Example Domain</title>

    <meta charset="utf-8" />
    <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <style type="text/css">
    body {
        background-color: #f0f0f2;
        margin: 0;
        padding: 0;
        font-family: -apple-system, system-ui, BlinkMacSystemFont, "Segoe UI", "Open Sans", "Helvetica Neue", Helvetica, Arial, sans-serif;
... snipped ...
```
Without the host header, the response is returned from the test server
```
curl -L http://localhost:8080/test/

404 page not found
```

### Impact
An open redirect vulnerability allows attackers to trick users into visiting malicious sites. This can lead to phishing attacks, credential theft, and malware distribution, as users trust the application’s domain while being redirected to harmful sites.

### CVSS score
I had originally scored this CVE as a Medium severity CVE with a score of 5.1 (CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:A/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N). This is the current score in the advisory.

Reflecting back, it can be argued that the score is 2.1 Low (CVSS:4.0/AV:N/AC:L/AT:P/PR:N/UI:A/VC:L/VI:L/VA:N/SC:N/SI:N/SA:N)

## Conclusion
This is another win for my free SAST tools experiment. FWIW, this is also the first security advisory published for `chi`. I am not entirely sure if this is the first vulnerability discovered in `chi`. I did try looking for CVEs but I did not find any - so I would like to believe this is the first.

Sneak peek into my fuzzing escapades - I gave up on fuzzing using AFL++ on popular C/C++ codebases - due to many skill issues. I prefer spending my time away from work doing non-security things (which is probably obvious given the inconsistent post history here and the basic content of the few blogs that are posted). Therefore I determined that I don't want to spend the time needed to dive into the completely unchartered territory of fuzzing C / C++ binaries. I did however find that go code can be fuzzed too - and that had a easier entry point for me personally. So, I pivoted to fuzzing go code instead. Did I find something? 🤷. Thank you for reading.

## Timeline
* March 5, 2025: Reported issue to chi maintainers through Github.
* June 6, 2025: Commented on report due to no response from chi maintainers. Received acknowledgement of report receipt.
* June 9, 2025: chi maintainers confirmed the issue.
* June 19, 2025: Fix released, GHSA published.
* June 29, 2025: Blog published.
