---
title: "Analyzing new-ish CVEs in PyLoad"
date: 2024-04-20T07:56:09-07:00
draft: false
toc: false
images:
tags:
  - python
  - cve-analysis
---

[PyLoad](https://pyload.net/) is an open source download manager written in Python. I first came across PyLoad while solving a HackTheBox machine (PC). Solving the machine required exploiting a pre-auth RCE ([CVE-2023-0297](https://nvd.nist.gov/vuln/detail/CVE-2023-0297)). I then rediscovered PyLoad when I was looking for CVEs to analyze. My intention was to analyze around 3-4 CVEs in the same product and PyLoad fit the bill perfectly, as there were 4 CVEs disclosed in January and February 2024. In this article, I will explain the CVEs, examine the provided proof of concepts, and evaluate the fixes.

## CVE-2024-24808 (Open Redirect)
[CVE-2024-24808](https://nvd.nist.gov/vuln/detail/CVE-2024-24808) is a open redirect vulnerability in PyLoad due to incorrect sanitization of the redirect URL which allows an attacker to bypass the URL sanitization function and redirect users to arbitrary URLs. The details provided here are taken from the [detailed advisory](https://github.com/pyload/pyload/security/advisories/GHSA-g3cm-qg2v-2hj5) available in the PyLoad repository.


### Vulnerability
PyLoad allows redirecting users to a new page after login by using a `next` query parameter (eg: `https://pyload.test/login?next=home`). PyLoad parses the provided `next` parameter and performs some checks in the `get_redirect_url` function:

```Python
def login():
  ...
  next = get_redirect_url(fallback=flask.url_for("app.dashboard"))
  ...

def get_redirect_url(fallback=None):
    login_url = urljoin(flask.request.url_root, flask.url_for('app.login'))
    request_url = unquote(flask.request.url)
    for location in flask.request.values.get("next"), flask.request.referrer:
        if not location:
            continue
        if location in (request_url, login_url):  # don't redirect to same location
            continue
        if is_safe_url(location):
            return location
    return fallback
```

One of the performed checks is `is_safe_url`, which is supposed to check if the URL provided is in the same host. The check should return `False` when arbitrary URLs, outside the domain in which PyLoad is running, are tested.

```Python
#: Checks if location belongs to same host address
def is_safe_url(location):
    ref_url = urlparse(flask.request.host_url)
    test_url = urlparse(urljoin(flask.request.host_url, location))
    return test_url.scheme in ('http', 'https') and ref_url.netloc == test_url.netloc
```

This works for the most part, except if malformed URLs are provided to `urlparse`. According to [`urlparse`](https://docs.python.org/3/library/urllib.parse.html#urllib.parse.urlparse) documentation, any malformed URLs are presumed to be a relative URL. This means it has no `netloc` and has a `path` component.

```
urlparse("http://a.xyz")
ParseResult(scheme='http', netloc='a.xyz', path='', params='', query='', fragment='')

urlparse("http:///a.xyz")
ParseResult(scheme='http', netloc='', path='/a.xyz', params='', query='', fragment='')
```

Therefore, when a malformed URL is provided to `is_safe_url` it returns `True`, i.e. the URL is safe. This can be seen in the example below. The provided `location` is first joined to the request host URL (`localhost:8080`) and then parsed with `urlparse` in `is_safe_url`. When malformed URL, `http:///a.xyz`, is provided, after `urljoin`, `a.xyz` is joined to `localhost:8080` as the malformed URL is treated as a path. This causes the test to return `True`.
```
Location: /home
Joined URL: http://localhost:8080/home
Ref URL: ParseResult(scheme='http', netloc='localhost:8080', path='/', params='', query='', fragment='')
Test URL: ParseResult(scheme='http', netloc='localhost:8080', path='/home', params='', query='', fragment='')
True

Location: http://a.xyz
Joined URL: http://a.xyz
Ref URL: ParseResult(scheme='http', netloc='localhost:8080', path='/', params='', query='', fragment='')
Test URL: ParseResult(scheme='http', netloc='a.xyz', path='', params='', query='', fragment='')
False

Location: http:///a.xyz
Joined URL: http://localhost:8080/a.xyz
Ref URL: ParseResult(scheme='http', netloc='localhost:8080', path='/', params='', query='', fragment='')
Test URL: ParseResult(scheme='http', netloc='localhost:8080', path='/a.xyz', params='', query='', fragment='')
True
```

When `is_safe_url` returns `True`, the redirect URL is set to the malformed URL `http:///a.xyz` which is then normalized to `http://a.xyz` leading to open redirect.

### Fix
The vulnerability is fixed by [this PR](https://github.com/pyload/pyload/commit/fe94451dcc2be90b3889e2fd9d07b483c8a6dccd). 

In the PR, the `is_safe_url` function is updated, but I did not find any usages of the function, so I will not discuss it here. The fix appears to be modifying the `get_redirect_url` function to use `flask.url_for`. According to the [documentation](https://flask.palletsprojects.com/en/3.0.x/api/#flask.Flask.url_for) `url_for` can be used to generate URLs for an endpoint. If an endpoint is not found for the provided value, it throws a `werkzeug.routing.BuildError`. 

```Python
def get_redirect_url(fallback=None):
    next_arg = flask.request.values.get("next")
    redirect_url = flask.url_for(fallback)
    if next_arg and next_arg != "login":  # don't redirect to same location
        try:
            redirect_url = flask.url_for(f"app.{next_arg}")
        except werkzeug.routing.BuildError:
            pass

    return urljoin(flask.request.url_root, redirect_url)
```  

Now for any provided value of `next`, `flask` will attempt to build a URL for `app.{next_arg}` and if its not found, it will redirect to the fallback. 
```
/home <-- this endpoint exists
127.0.0.1 - - [16/Mar/2024 11:19:35] "GET /?next=home HTTP/1.1" 200 -
Could not build url for endpoint 'http:///example.com'. Did you mean 'hello_world' instead?
127.0.0.1 - - [16/Mar/2024 11:19:45] "GET /?next=http:///example.com HTTP/1.1" 200 
```
***
## CVE-2024-22416 (Cross Site Request Forgery)
[CVE-2024-22416](https://nvd.nist.gov/vuln/detail/CVE-2024-22416) is a cross site request forgery (CSRF) vulnerability in PyLoad. The `SameSite` attribute in the session cookie used with PyLoad APIs is not set to `SameSite: strict`. This allows an attacker to make API calls using GET requests on behalf of other users via a CSRF attack, thereby performing actions on behalf of said users. The details provided here are taken from the [advisory](https://github.com/pyload/pyload/security/advisories/GHSA-pgpj-v85q-h5fm) available in the PyLoad repository.

### Vulnerability
There is not much to explain for this CVE. It is a textbook CSRF vulnerability, as the cookie is not prevented from being sent in cross site requests. If an attacker can trick a legitimate user to visit a malicious site, which has an embedded GET request to a PyLoad API, the browser will make the GET request and send the cookies of the legitimate user with the request (provided the user was logged in, and had a valid session cookie in the cookie jar). This will lead to PyLoad performing the action associated with the API as if the legitimate user is performing it. The example in the advisory mentions the `/add_user` API, which can be called with a username and a password like `api/add_user/%22username%22,%22password%22`. When triggered via an admin's session token, a user with username  `username` and password `password` will be created with admin privileges.

### Fix
The vulnerability is fixed by [this PR](https://github.com/pyload/pyload/commit/1374c824271cb7e927740664d06d2e577624ca3e). 

Flask provides a `config` object that allows setting certain Flask configuration values. One of the values that can be configured is [`SESSION_COOKIE_SAMESITE`](https://flask.palletsprojects.com/en/3.0.x/config/#SESSION_COOKIE_SAMESITE) that sets the value of the `SameSite` attribute of the session cookie. The fix is setting this to `Strict`.

```Python
app.config["SESSION_COOKIE_SAMESITE"] = "Strict"
```
***
## CVE-2024-21644 (Flask configuration leak)
[CVE-2024-21644](https://nvd.nist.gov/vuln/detail/CVE-2024-21644) is a configuration leak vulnerabilty in PyLoad which allows an unauthenticated user to view the Flask config used by PyLoad including secrets by browsing to a specific URL.The details provided here are taken from the [advisory](https://github.com/pyload/pyload/security/advisories/GHSA-mqpq-2p68-46fv) available in the PyLoad repository.

### Vulnerability
This vulnerability is caused due to an interesting mix of circumstances. First, there is a route `/render/<path:filename>` which allows an unautheticated user to view predefined templates. 
```Python
@bp.route("/render/<path:filename>", endpoint="render")
def render(filename):
    mimetype = mimetypes.guess_type(filename)[0] or "text/html"
    data = render_template(filename)
    return flask.Response(data, mimetype=mimetype)
```

Next, there is a template `info.html` which renders a variable named `config`.
```HTML
<tr>
    <td>{{ _("Config folder:") }}</td>
    <td>{{ config }}</td>
</tr>
```

In normal operation, the value of config is supposed to be provided using the `render_template` function. In this case `**context` provides the values to display to `info.html`.
```Python
context = {
    "python": sys.version,
    "os": " ".join((os.name, sys.platform) + extra),
    "version": api.get_server_version(),
    "folder": PKGDIR,
    "config": api.get_userdir(),
    "download": conf["general"]["storage_folder"]["value"],
    "freespace": format.size(api.free_space()),
    "webif": conf["webui"]["port"]["value"],
    "language": conf["general"]["language"]["value"],
}
return render_template("info.html", **context)
```

However, all Flask templates also receive the Flask configuration as a variable named `config`. Therefore, when `info.html` is requested using `/render`, since a second parameter is not sent to `render_template`, the `config` value used to render `info.html` is the Flask configuration which contains sensitive information. These sensitive values are then displayed to an unauthenticated user.

### Fix
This issue is fixed by [this PR](https://github.com/pyload/pyload/commit/bb22063a875ffeca357aaf6e2edcd09705688c40). The fix is quite simple: the `config` variable in `info.html` is renamed to `config_folder`.
***

## CVE-2024-21645 (Log Injection)

[CVE-2024-21645](https://nvd.nist.gov/vuln/detail/CVE-2024-21645) is a log injection vulnerability in PyLoad that allows an attacker to inject arbitrary logs by sending a crafted `username` parameter while attempting to log in.

### Vulnerability

PyLoad logs the username provided on an unsuccessful login attempt. 

```Python
api = flask.current_app.config["PYLOAD_API"]
user_info = api.check_auth(user, password)

if not user_info:
    log.error(f"Login failed for user '{user}'")
```
A login attempt can be made using a request like
```
http://localhost:8000/login?next=http://localhost:8000/' -X POST -H 'Content-Type: application/x-www-form-urlencoded' --data-raw $'do=login&username=user&password=password&submit=Login'
```
The value sent as the `username` parameter is logged as-is without any sanitization. Therefore, an attacker can craft a `username` param with newlines (and log string format) and inject a log. A malicious request has been provided by the reporter in the advisory

```
http://localhost:8000/login?next=http://localhost:8000/' -X POST -H 'Content-Type: application/x-www-form-urlencoded' --data-raw $'do=login&username=wrong\'%0a[2024-01-05 02:49:19]  HACKER               PinkDraconian  THIS ENTRY HAS BEEN INJECTED&password=wrong&submit=Login'
```
This will insert a log like
```
2024-01-05 02:49:19]  HACKER               PinkDraconian  THIS ENTRY HAS BEEN INJECTED
```

### Fix
This issue is fixed by [this PR](https://github.com/pyload/pyload/commit/4159a1191ec4fe6d927e57a9c4bb8f54e16c381d). 

The fix is to sanitize the value of the username paramater to remove newlines and carriage returns. 

```Python
 api = flask.current_app.config["PYLOAD_API"]
user_info = api.check_auth(user, password)

sanitized_user = user.replace("\n", "\\n").replace("\r", "\\r")
if not user_info:
    log.error(f"Login failed for user '{sanitized_user}'")
    return jsonify(False)
```
***

I enjoyed analyzing these CVEs and fixes. I especially found the `urlparse` issue in CVE-2024-24808 very interesting, since I was not aware of the described behavior of `urlparse`. As a side effect of analyzing these CVEs, I also learnt a few things about Flask internals. I'm still hunting my first CVE, and studying these advisories has taught me lots of useful techniques, tips, and tricks for my search. Hopefully, I will find the CVE soon. If not, there's plenty of CVEs to analyze and learn from!