# Async cookies API explained

This is a proposal to bring an asynchronous cookie API to scripts running in HTML documents and [service workers](https://github.com/slightlyoff/ServiceWorker).

[HTTP cookies](https://tools.ietf.org/html/rfc6265) have, since their origins at Netscape [(documentation preserved by archive.org)](https://web.archive.org/web/0/http://wp.netscape.com/newsref/std/cookie_spec.html), provided a [valuable state-management mechanism](http://www.montulli-blog.com/2013/05/the-reasoning-behind-web-cookies.html) for the web. 

The synchronous single-threaded script-level `document.cookie` and `<meta http-equiv="set-cookie" ...>` interface to cookies has been a source of [complexity and performance woes](https://lists.w3.org/Archives/Public/public-whatwg-archive/2009Sep/0083.html) further exacerbated by the move in many browsers from:
  - a single browser process,
  - a single-threaded event loop model, and
  - no general expectation of responsiveness for scripted event handling while processing cookie operations

... to the modern web which strives for smoothly responsive high performance:
  - in multiple browser processes,
  - with a multithreaded, multiple-event loop model, and
  - with an expectation of responsiveness on human-reflex time scales.

On the modern web a cookie operation in one part of a web application cannot block:
  - the rest of the web application,
  - the rest of the web origin, or
  - the browser as a whole.

Newer parts of the web built in service workers [need access to cookies too](https://github.com/slightlyoff/ServiceWorker/issues/707) but cannot use the synchronous, blocking `document.cookie` and `<meta http-equiv="set-cookie" ...>` interfaces at all as they both have no `document` and also cannot block the event loop as that would interfere with handling of unrelated events.

## A taste of the proposed change

Although it is tempting to [rethink cookies](https://discourse.wicg.io/t/rethinking-cookies/744) entirely, web sites today continue to rely heavily on them, and the script APIs for using them are largely unchanged over their first decades of usage.

Today writing a cookie means blocking your event loop while waiting for the browser to synchronously update the cookie jar with a carefully-crafted cookie string in `Set-Cookie` format:

```js
document.cookie =
  '__Secure-COOKIENAME=cookie-value' +
  '; Path=/' +
  '; expires=Fri, 12 Aug 2016 23:05:17 GMT' +
  '; Secure' +
  '; Domain=example.org';
// now we could assume the write succeeded, but since
// failure is silent it is difficult to tell, so we
// read to see whether the write succeeded
var successRegExp =
  /(^|; ?)__Secure-COOKIENAME=cookie-value(;|$)/;
if (String(document.cookie).match(successRegExp)) {
  console.log('It worked!');
} else {
  console.error('It did not work, and we do not know why');
}
```

What if you could instead write:

```js
cookieStore.set(
  '__Secure-COOKIENAME',
  'cookie-value',
  {
    expires: Date.now() + 24*60*60*1000,
    domain: 'example.org'
  }).then(function() {
    console.log('It worked!');
  }, function(reason) {
    console.error(
      'It did not work, and this is why:',
      reason);
  });
// Meanwhile we can do other things while waiting for
// the cookie store to process the write...
```

This also has the advantage of not relying on `document` and not blocking, which together make it usable from [service workers](https://github.com/slightlyoff/ServiceWorker), which otherwise do not have cookie access from script.

This proposal also includes a power-efficient monitoring API to replace `setTimeout`-based polling cookie monitors with cookie change observers.

## Summary

This proposal outlines an asynchronous API using Promises/async functions for the following cookie operations:

 * [write](#writing) (or "set") cookies
 * [delete](#clearing) (or "expire") cookies
 * [read](#reading) (or "get") [script-visible](#script-visibility) cookies
   * ... including for specified in-scope request paths in
   [service worker](https://github.com/slightlyoff/ServiceWorker) contexts
 * [monitor](#monitoring) [script-visible](#script-visibility) cookies for changes
   * ... [using `CookieObserver`](#single-execution-context) in long-running script contexts (e.g. `document`)
   * ... [using `CookieChangeEvent`](#service-worker) after registration during the `InstallEvent`
   in ephemeral [service worker](https://github.com/slightlyoff/ServiceWorker) contexts
   * ... again including for script-supplied in-scope request paths
   in [service worker](https://github.com/slightlyoff/ServiceWorker) contexts

#### Script visibility

A cookie is script-visible when it is in-scope and does not have the `HttpOnly` cookie flag.

### Motivations

Some service workers [need access to cookies](https://github.com/slightlyoff/ServiceWorker/issues/707) but
cannot use the synchronous, blocking `document.cookie` interface as they both have no `document` and
also cannot block the event loop as that would interfere with handling of unrelated events.

A new API may also provide a rare and valuable chance to address
some [outstanding cross-browser incompatibilities](https://github.com/inikulin/cookie-compat) and bring [divergent
specs and user-agent behavior](https://github.com/whatwg/html/issues/804) into closer correspondence.

A well-designed and opinionated API may actually make cookies easier to deal with correctly from
scripts, with the potential effect of reducing their accidental misuse. An efficient monitoring API, in particular,
can be used to replace power-hungry polling cookie scanners.

The API must interoperate well enough with existing cookie APIs (HTTP-level, HTML-level and script-level) that it can be adopted incrementally by a large or complex website.

### Opinions

This API defaults cookie paths to `/` for cookie write operations, including deletion/expiration. The implicit relative path-scoping of cookies to `.` has caused a lot of additional complexity for relatively little gain given their security equivalence under the same-origin policy and the difficulties arising from multiple same-named cookies at overlapping paths on the same domain. Cookie paths without a trailing `/` are treated as if they had a trailing `/` appended for cookie write operations. Cookie paths must start with `/` for write operations, and must not contain any `..` path segments. Query parameters and URL fragments are not allowed in paths for cookie write operations.

URLs without a trailing `/` are treated as if the final path segment had been removed for cookie read operations, including change monitoring. Paths for cookie read operations are resolved relative to the default read cookie path.

This API defaults cookies to "Secure" when they are written from a secure web origin. This is intended to prevent unintentional leakage to unsecured connections on the same domain. Furthermore it disallows (to the extent permitted by the browser implementation) [creation or modification of `Secure`-flagged cookies from unsecured web origins](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-alone-00) and [enforces special rules for the `__Host-` and `__Secure-` cookie name prefixes](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-prefixes-00).

This API defaults cookies to "Domain"-less, which in conjunction with "Secure" provides origin-scoped cookie
behavior in most modern browsers. When practical the [`__Host-` cookie name prefix](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-prefixes-00) should be used with these cookies so that cooperating browsers origin-scope them.

Serialization of expiration times for non-session cookies in a special cookie-specific format has proven cumbersome,
so this API allows JavaScript Date objects and numeric timestamps (milliseconds since the beginning of the Unix epoch) to be used instead. The inconsistently-implemented Max-Age parameter is not exposed, although similar functionality is available for the specific case of expiring a cookie.

Cookies without `=` in their HTTP Cookie header serialization are treated as having an empty name, consistent with the majority of current browsers. Cookies with an empty name cannot be set using values containing `=` as this would result in ambiguous serializations in the majority of current browsers.

Internationalized cookie usage from scripts has to date been slow and browser-specific due to lack of interoperability because although several major browsers use UTF-8 interpretation for cookie data, historically Safari and browsers based on WinINet have not. This API mandates UTF-8 interpretation for cookies read or written by this API.

Use of cookie-change-driven scripts has been hampered by the absence of a power-efficient (non-polling) API for this. This API provides observers for efficient monitoring in document contexts and interest registration for efficient monitoring in service worker contexts.

Scripts should not have to write and then read "test cookies" to determine whether script-initiated cookie write access is possible, nor should they have to correlate with cooperating server-side versions of the same write-then-read test to determine that script-initiated cookie read access is impossible despite cookies working at the HTTP level.

### Compatiblity

Some user-agents implement non-standard extensions to cookie behavior. The intent of this specification,
though, is to first capture a useful and interoperable (or mostly-interoperable) subset of cookie behavior implemented
across modern browsers. As new cookie features are specified and adopted it is expected that this API will be
extended to include them. A secondary goal is to converge with `document.cookie` behavior, `<meta http-equiv=set-cookie>`,
and the http cookie specification. See https://github.com/whatwg/html/issues/804 and https://inikulin.github.io/cookie-compat/
for the current state of this convergence.

Differences across browsers in how bytes outside the printable-ASCII subset are interpreted has led to
long-lasting user- and developer-visible incompatibilities across browsers making internationalized use of cookies
needlessly cumbersome. This API requires UTF-8 interpretation of cookie data and uses `USVString` for the script interface,
with the additional side-effects that subsequent uses of `document.cookie` to read a cookie read or written through this interface and subsequent uses of `document.cookie` or
`<meta http-equiv=set-cookie>` to update a cookie previously read or written through this interface will also use a UTF-8 interpretation of the cookie data. In practice this
will change the behavior of `WinINet`-based user agents and Safari but should bring their behavior into concordance
with other modern user agents.

## Using the async cookies API

*Note:* This is largely inspired by the [API sketch](https://github.com/WICG/async-cookies-api/issues/14)

### Reading

You can read the first in-scope script-visible value for a given cookie name. In a service worker context this defaults to the path
of the service worker's registered scope. In a document it defaults to the path of the current document and does not respect
changes from `replaceState` or `document.domain`.

```js
function getOneSimpleOriginCookie() {
  return cookieStore.get('__Host-COOKIENAME').then(function(cookie) {
    console.log(cookie ? ('Current value: ' + cookie.value) : 'Not set');
  });
}
getOneSimpleOriginCookie().then(function() {
  console.log('getOneSimpleOriginCookie succeeded!');
}, function(reason) {
  console.error('getOneSimpleOriginCookie did not succeed: ', reason);
});
```

You can use exactly the same Promise-based API with the newer `async` ... `await` syntax and arrow functions for more readable code:

```js
let getOneSimpleOriginCookieAsync = async () => {
  let cookie = await cookieStore.get('__Host-COOKIENAME');
  console.log(cookie ? ('Current value: ' + cookie.value) : 'Not set');
};
getOneSimpleOriginCookieAsync().then(
  () => console.log('getOneSimpleOriginCookieAsync succeeded!'),
  reason => console.error('getOneSimpleOriginCookieAsync did not succeed: ', reason));
```

Remaining examples use this syntax along with destructuring for clarity, and omit the calling code.

In a service worker context you can read a cookie from the point of view of a particular in-scope URL, which may be useful when handling regular (same-origin, in-scope) fetch events or foreign fetch events.

```js
let getOneCookieForRequestUrl = async () => {
  let cookie = await cookieStore.get('__Secure-COOKIENAME', {url: '/cgi-bin/reboot.php'});
  console.log(cookie ? ('Current value in /cgi-bin is ' + cookie.value) : 'Not set in /cgi-bin');
};
```

Sometimes you need to see the whole script-visible in-scope subset of the cookie jar, including potential reuse of the same
cookie name at multiple paths and/or domains (the paths and domains are not exposed to script by this API, though):

```js
let countCookies = async () => {
  let cookieList = await cookieStore.getAll();
  console.log('How many cookies? %d', cookieList.length);
  cookieList.forEach(cookie => console.log('Cookie %s has value %o', cookie.name, cookie.value));
};
```

Sometimes an expected cookie is known by a prefix rather than by an exact name, for instance when reading all cookies managed by a particular library (e.g. in [this one](https://developers.google.com/+/web/api/javascript#gapiinteractivepost_interactive_posts) the name prefix identifies the library) or when reading all cookie names owned by a particular application on a shared web host (a name prefix is often used to identify the owning application):

```js
let countMatchingSimpleOriginCookies = async () => {
  let cookieList = await cookieStore.getAll({name: '__Host-COOKIEN', matchType: 'startsWith'});
  console.log('How many matching cookies? %d', cookieList.length);
  cookieList.forEach(({name, value}) => console.log('Matching cookie %s has value %o', name, value));
};
```

In a service worker context you may need to read more than one cookie from an in-scope path different from the default, for instance while handling a fetch event which has important cookies scoped to a sub-path containing the fetched resource to avoid cookie name collisions:

```js
let countMatchingCookiesForRequestUrl = async () => {
  // 'equals' is the default matchType and indicates exact matching
  let cookieList =
    await cookieStore.getAll({name: 'LEGACYSORTPREFERENCE', matchType: 'equals', url: '/pictures/'});
  console.log('How many legacy sort preference cookies? %d', cookieList.length);
  cookieList.forEach(({value}) => console.log('Legacy sort preference cookie has value %o', value);
};
```

You might even need to read all of them:

```js
let countAllCookiesForRequestUrl = async () => {
  let cookieList = await cookieStore.getAll({url: '/sw-scope/session2/document5/'});
  console.log('How many script-visible cookies? %d', cookieList.length);
  cookieList.forEach(({name, value}) => console.log('Cookie %s has value %o', name, value));
};
```

### Writing

You can set a cookie too:

```js
let setOneSimpleOriginSessionCookie = async () => {
  await cookieStore.set('__Host-COOKIENAME', 'cookie-value');
  console.log('Set!');
};
```

That defaults to path `/` and *implicit* domain, and defaults to a Secure-if-https-origin, non-HttpOnly session cookie which will be visible to scripts. You can override any of these defaults except for HttpOnly (which is not settable from script in modern browsers) if needed:

```js
let setOneDaySecureCookieWithDate = async () => {
  // one day ahead, ignoring a possible leap-second
  let inTwentyFourHours = new Date(Date.now() + 24 * 60 * 60 * 1000);
  await cookieStore.set('__Secure-COOKIENAME', 'cookie-value', {
      path: '/cgi-bin/',
      expires: inTwentyFourHours,
      secure: true,
      domain: 'example.org'
    });
  console.log('Set!');
};
```

Of course the numeric form (milliseconds since the beginning of 1970 UTC) works too:

```js
let setOneDayUnsecuredCookieWithMillisecondsSinceEpoch = async () => {
  // one day ahead, ignoring a possible leap-second
  let inTwentyFourHours = Date.now() + 24 * 60 * 60 * 1000;
  await cookieStore.set('LEGACYCOOKIENAME', 'cookie-value', {
      path: '/cgi-bin/',
      expires: inTwentyFourHours,
      secure: false,
      domain: 'example.org'
    });
  console.log('Set!');
};
```

Sometimes an expiration date comes from existing script it's not easy or convenient to replace, though:

```js
let setSecureCookieWithHttpLikeExpirationString = async () => {
  await cookieStore.set('__Secure-COOKIENAME', 'cookie-value', {
      path: '/cgi-bin/',
      expires: 'Mon, 07 Jun 2021 07:07:07 GMT',
      secure: true,
      domain: 'example.org'
    });
  console.log('Set!');
};
```

In this case the syntax is that of the HTTP cookies spec; any other syntax will result in promise rejection.

You can set multiple cookies too, but - as with HTTP `Set-Cookie` - the multiple write operations have no guarantee of atomicity:

```js
let setThreeSimpleOriginSessionCookiesSequentially = async () => {
  await cookieStore.set('__Host-🍪', '🔵cookie-value1🔴');
  await cookieStore.set('__Host-🌟', '🌠cookie-value2🌠');
  await cookieStore.set('__Host-🌱', '🔶cookie-value3🔷');
  console.log('All set!');
  // NOTE: this assumes no concurrent writes from elsewhere; it also
  // uses three separate cookie jar read operations where a single getAll
  // would be more efficient, but this way the CookieStore does the filtering
  // for us.
  let matchingValues = await Promise.all(['🍪', '🌟', '🌱'].map(
    async ಠ_ಠ => (await cookieStore.get('__Host-' + ಠ_ಠ)).value));
  let actual = matchingValues.join(';');
  let expected = '🔵cookie-value1🔴;🌠cookie-value2🌠;🔶cookie-value3🔷';
  if (actual !== expected) {
    throw new Error([
      'Expected ',
      JSON.stringify(expected),
      ' but got ',
      JSON.stringify(actual)].join(''));
  }
  console.log('All verified!');
};
```

If the relative order is unimportant the operations can be performed without specifying the order:

```js
let setThreeSimpleOriginSessionCookiesNonsequentially = async () => {
  await Promise.all([
    cookieStore.set('__Host-unordered🍪', '🔵unordered-cookie-value1🔴'),
    cookieStore.set('__Host-unordered🌟', '🌠unordered-cookie-value2🌠'),
    cookieStore.set('__Host-unordered🌱', '🔶unordered-cookie-value3🔷')]);
  console.log('All set!');
  // NOTE: this assumes no concurrent writes from elsewhere; it also
  // uses three separate cookie jar read operations where a single getAll
  // would be more efficient, but this way the CookieStore does the filtering
  // for us.
  let matchingCookies = await Promise.all(['🍪', '🌟', '🌱'].map(
    ಠ_ಠ => cookieStore.get('__Host-unordered' + ಠ_ಠ)));
  let actual = matchingCookies.map(({value}) => value).join(';');
  let expected =
    '🔵unordered-cookie-value1🔴;🌠unordered-cookie-value2🌠;🔶unordered-cookie-value3🔷';
  if (actual !== expected) {
    throw new Error([
      'Expected ',
      JSON.stringify(expected),
      ' but got ',
      JSON.stringify(actual)].join(''));
  }
  console.log('All verified!');
};
```

#### Clearing

Clearing (deleting) a cookie is accomplished by expiration, that is by replacing it with an equivalent-scope cookie with
an expiration in the past:

```js
let setExpiredSecureCookieWithDomainPathAndFallbackValue = async () => {
  let theVeryRecentPast = Date.now();
  let expiredCookieSentinelValue = 'EXPIRED';
  await cookieStore.set('__Secure-COOKIENAME', expiredCookieSentinelValue, {
      path: '/cgi-bin/',
      expires: theVeryRecentPast,
      secure: true,
      domain: 'example.org'
    });
  console.log('Expired! Deleted!! Cleared!!1!');
};
```

In this case the cookie's value is not important unless a clock is somehow re-set incorrectly or otherwise behaves nonmonotonically or incoherently.

A syntactic shorthand is also provided which is equivalent to the above except that the clock's accuracy and monotonicity becomes irrelevant:

```js
let deleteSimpleOriginCookie = async () => {
  await cookieStore.delete('__Host-COOKIENAME');
  console.log('Expired! Deleted!! Cleared!!1!');
};
```

Again, the path and/or domain can be specified explicitly here.

```js
let deleteSecureCookieWithDomainAndPath = async () => {
  await cookieStore.delete('__Secure-COOKIENAME', {
      path: '/cgi-bin/',
      domain: 'example.org',
      secure: true
    });
  console.log('Expired! Deleted!! Cleared!!1!');
};
```

This API has semantics aligned with the interpretation of `Max-Age=0` common to most modern browsers.

#### Rejection

Any cookie write operation that would result in a cookie setting failure should be rejected.

In addition to any rejections covered by cookie setting failures, the following operations will be rejected regardless of whether they would not result in a cookie setting failure according to the HTTP cookies specification:

Syntactic rejection: a cookie write operation (`set` or `delete`) may fail to actually set the requested cookie. A cookie name, value or parameter validation problem (for instance, a semicolon `;` cannot appear in any of these) will result in the promise being rejected.

Implementation limits: exceeding implementation size limits, setting an out-of-supported-range expiration date, or setting a cookie on a different eTLD+1 domain will also reject the promise and the cookie will not be set/modified. Other implementation-specific limits could lead to rejection, too.

Special prefixes: a cookie write operation for a cookie using one of the `__Host-` and `__Secure-` name prefixes from [Cookie Prefixes](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-prefixes-00) will be rejected if the other cookie parameters do not conform to the special rules for the prefix. For both prefixes the Secure parameter must be set (either explicitly set to `true` or implicitly due to the script running on a secure origin), and the script must be running on a secure origin. Additionally for the `__Host-` prefix the Path parameter must have the value `/` (either explicitly or implicitly) and the Domain parameter must be absent.

`Secure`-from-unsecured: a cookie write operation from an unsecured web origin creating, modifying or overwriting a `Secure`-flagged cookie will be rejected unless browser implementation details prevent this, in accordance with [Leave Secure Cookies Alone](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-alone-00). In any case the `Secure` flag must be false either explicitly or implicitly in all cookie write operations using this API when running on an unsecured web origin or the operation will be rejected.

### Monitoring

*Note:* multiple cookie changes in rapid succession may cause the user agent to only check for script-visible changes (relative to the last time the observer was called or the event was fired) after all the changes have been applied. In some cases (for instance, a very short-lived cookie being set and then expiring) this may cause the observer/event handler to miss the (now-expired) ephemeral cookie entirely. Expiration of a cookie monitored by a service worker might not deliver the cookie change event until the next time a document or service worker in the cookie's domain or origin is used.

Other parts of an application need to be prepared to run after a cookie they rely on has expired but before the change event has been delivered. The (read API)[#reading] should be used in cases where a consistent view of the cookie store is needed prior to completing another task - for instance, displaying a user's private messages might best be deferred until the presence of an (unexpired) session cookie is verified.

To avoid unneccessary work (executing JavaScript needlessly, or even starting an otherwise-stopped service worker for an unrelated cookie change, e.g. [CSRF token cookies updated on every HTTP request](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet#Double_Submit_Cookie)), monitoring interfaces allow the cookies names or name prefixes of interest to be specified.

#### Single execution context

You can monitor for script-visible cookie changes (creation, modification and deletion) during the lifetime of your script's execution context:

```js
let startMonitoring = new Promise(resolve => {
  // This will get invoked (asynchronously) shortly after the observe(...) call to
  // provide an initial snapshot; in that case the length of cookieChanges may be 0,
  // indicating no matching script-visible cookies for any URL+cookieStore currently
  // observed. The CookieObserver instance is passed as the second parameter to allow
  // additional calls to observe or disconnect.
  let callback = (cookieChanges, observer) => {
    console.log(
      '%d script-visible cookie changes for CookieObserver %o',
      cookieChanges.length,
      observer);
    cookieChanges.forEach(({cookieStore, type, url, name, value, all}) => {
      console.log(
        'CookieChange type %s for observed url %s in CookieStore %o; all: %o',
        type,
        // Note that this will be the passed-in or defaulted value for the corresponding
        // call to observe(...).
        url,
        // This is the same CookieStore passed to observe(...)
        cookieStore,
        // This means we do not need to maintain our own shadow cookie jar and disambiguates in
        // cases where the same cookie name appears more than once in the store with differing scope
        all);
      switch(type) {
      case 'visible':
        // Creation or modification (e.g. change in value, or removal of HttpOnly), or
        // appearance to script due to change in policy or permissions
        console.log('Cookie %s now visible to script with value %s', name, value);
        break;
      case 'hidden':
        // Deletion/expiration or disappearance (e.g. due to modification adding HttpOnly),
        // or disappearance from script due to change in policy or permissions
        console.log('Cookie %s with value %s expired or no longer visible to script', name, value);
        break;
      default:
        throw 'Unexpected CookieChange type ' + type;
      }
    });
    if (resolve == null) return;
    // Resolve on completion of first callback
    resolve();
    resolve = null;
  };
  stopMonitoring();
  let observer = self.observer = new CookieObserver(callback);
  // If null or omitted this defaults to location.pathname up to and
  // including the final '/' in a document context, or worker scope up
  // to and including the final '/' in a service worker context.
  let url = (location.pathname).replace(/[^\/]+$/, '');
  // If null or omitted this defaults to interest in all
  // script-visible cookies.
  let interests = [
    // Interested in all secure cookies named '__Secure-COOKIENAME';
    // the default matchType is 'equals' at the given URL.
    {name: '__Secure-COOKIENAME', url: url},
    // Interested in all simple origin cookies named like
    // /^__Host-COOKIEN.*$/ at the default URL.
    {name: '__Host-COOKIEN', matchType: 'startsWith'},
    // Interested in all simple origin cookies named '__Host-🍪'
    // at the default URL.
    {name: '__Host-🍪'},
    // Interested in all cookies named 'OLDCOOKIENAME' at the given URL.
    {name: 'OLDCOOKIENAME', matchType: 'equals', url: url},
    // Interested in all simple origin cookies named like
    // /^__Host-AUTHTOKEN.*$/ at the given URL.
    {name: '__Host-AUTHTOKEN', matchType: 'startsWith', url: url + 'auth/'}
  ];
  observer.observe(cookieStore, interests);
  // Default interest: all script-visible changes, default URL
  observer.observe(cookieStore);
});
```

Successive attempts to `observe` on the same CookieObserver are additive but a single change to a single cookie will only be reported once for each URL where it is observed in a given CookieStore.

Eventually you may want to stop monitoring for script-visible cookie changes:

```js
let stopMonitoring = () => {
  if (self.observer == null) return;
  let observer = self.observer;
  self.observer = null;
  // No more callbacks until another call to observer.observe(...)
  observer.disconnect();
};
```

#### Service worker

A service worker does not have a persistent JavaScript execution context, so a different API is needed for interest registration. Register your interest while handling the `InstallEvent` to ensure your service worker will run when a cookie you care about changes.

```js
addEventListener('install', event => {
  let url = '/sw-scope/';
  // If null or omitted this defaults to interest in all
  // script-visible cookies.
  let interests = [
    // Interested in all secure cookies named '__Secure-COOKIENAME';
    // the default matchType is 'equals' at the given URL.
    {name: '__Secure-COOKIENAME', url: url},
    // Interested in all simple origin cookies named like
    // /^__Host-COOKIEN.*$/ at the default URL.
    {name: '__Host-COOKIEN', matchType: 'startsWith'},
    // Interested in all simple origin cookies named '__Host-🍪'
    // at the default URL.
    {name: '__Host-🍪'},
    // Interested in all cookies named 'OLDCOOKIENAME' at the given URL.
    {name: 'OLDCOOKIENAME', matchType: 'equals', url: url},
    // Interested in all simple origin cookies named like
    // /^__Host-AUTHTOKEN.*$/ at the given URL.
    {name: '__Host-AUTHTOKEN', matchType: 'startsWith', url: url + 'auth/'}
  ];
  // Cookie change interest is registered during the InstallEvent in a service
  // worker context; parameters are identical to CookieObserver's observe(...)
  // method. The url must be inside the directory portion of the registration scope
  // of the service worker, and defaults to the directory portion of the registration
  // scope if null or omitted.
  event.registerCookieChangeInterest(cookieStore); // all cookies, default url
  // Call it more than once to register additional interests:
  event.registerCookieChangeInterest(cookieStore, interests);
});
```

*Note:* cookie changes which occur at paths not yet known during handling of the `InstallEvent` cannot be monitored in a service worker context after the execution context ends using this API.

You also need to be sure to handle the `CookieChangeEvent`:

```js
// This will get invoked once (asynchronously) after activation to provide an initial
// snapshot of the script-visible cookie jar; in that case the length of the cookieChangeList may
// be 0, indicating no matching script-visible cookies for any URL for which cookie interest was
// registered.
addEventListener('cookiechange', event => {
  // event.detail is CookieChanges, analogous to the one passed to CookieObserver's callback
  let cookieChanges = event.detail;
  console.log('%d script-visible cookie changes for ServiceWorker', cookieChanges.length);
  cookieChanges.forEach(({cookieStore, type, url, name, value, all}) => {
    console.log(
      'CookieChange type %s for observed url %s in CookieStore %o; all: %o',
      type,
      // Note that this will be the passed-in or defaulted value for the corresponding
      // call to observe(...).
      url,
      // This is the same CookieStore passed to observe(...)
      cookieStore,
      // This means we do not need to maintain our own shadow cookie jar and disambiguates in
      // cases where the same cookie name appears more than once in the store with differing scope
      all);
    switch(type) {
    case 'visible':
      // Creation or modification (e.g. change in value, or removal of HttpOnly), or
      // appearance to script due to change in policy or permissions
      console.log('Cookie %s now visible to script with value %s', name, value);
      break;
    case 'hidden':
      // Deletion/expiration or disappearance (e.g. due to modification adding HttpOnly),
      // or disappearance from script due to change in policy or permissions
      console.log('Cookie %s with value %s expired or no longer visible to script', name, value);
      break;
    default:
      throw 'Unexpected CookieChange type ' + type;
    }
  });
});
```

*Note:* a service worker script needs to be prepared to handle "duplicate" notifications after updates to the service worker script with overlapping cookie change interests compared to those interests previously registered.

The power and resource consumption of this feature of the API will need to be carefully balanced against its utility, and for this reason delivery of a change event may be deferred until after the next browser start-up (for unrelated reasons) in cases where the change notification is for a cookie expiration and the browser is not already running.

This API may cause new service worker start-up after browser start-up in cases where a cookie monitored by the service worker expired while the browser was not running. This will not block other parts of browser start-up, however. Expiration of a cookie monitored by a service worker also might not deliver the cookie change event until the next time a document or service worker in the cookie's domain or origin is used.

## Security

Other than cookie access from service worker contexts, this API is not intended to expose any new capabilities to the web.

### Gotcha!

Although browser cookie implementations are now evolving in the direction of better security and fewer surprising and error-prone defaults, there are at present few guarantees about cookie data security.

 * unsecured origins can typically overwrite cookies used on secure origins
 * superdomains can typically overwrite cookies seen by subdomains
 * cross-site scripting attacts and other script and header injection attacks can be used to forge cookies too
 * cookie read operations (both from script and on web servers) don't give any indication of where the cookie came from
 * browsers sometimes truncate, transform or evict cookie data in surprising and counterintuitive ways
   * ... due to reaching storage limits
   * ... due to character encoding differences
   * ... due to differing syntactic and semantic rules for cookies

For these reasons it is best to use caution when interpreting any cookie's value, and never execute a cookie's value as script, HTML, CSS, XML, PDF, or any other executable format.

### Restrict?

This API may have the unintended side-effect of making cookies easier to use and consequently encouraging their further use. If it causes their further use in unsecured `http` contexts this could result in a web less safe for users. For that reason it may be desirable to restrict its use, or at least the use of the `set` and `delete` operations, to secure origins running in secure contexts.

### Surprises

Some existing cookie behavior (especially domain-rather-than-origin orientation, unsecured contexts being able to set cookies readable in secure contexts, and script being able to set cookies unreadable from script contexts) may be quite surprising from a web security standpoint.

Other surprises are documented in [Section 1 of HTTP State Management Mechanism (RFC 6265)](https://tools.ietf.org/html/rfc6265#section-1) - for instance, a cookie may be set for a superdomain (e.g. app.example.com may set a cookie for the whole example.com domain), and a cookie may be readable across all port numbers on a given domain name.

Further complicating this are historical differences in cookie-handling across major browsers, although some of those (e.g. port number handling) are now handled with more consistency than they once were.

### Prefixes

Where feasible the examples use the `__Host-` and `__Secure-` name prefixes from [Cookie Prefixes](https://tools.ietf.org/html/draft-ietf-httpbis-cookie-prefixes-00) which causes some current browsers to disallow overwriting from unsecured contexts, disallow overwriting with no `Secure` flag, and -- in the case of `__Host-` -- disallow overwriting with an explicit `Domain` or non-'/' `Path` attribute (effectively enforcing same-origin semantics.) These prefixes provide important security benefits in those browsers implementing Secure Cookies and degrade gracefully (i.e. the special semantics may not be enforced in other cookie APIs but the cookies work normally and the async cookies API enforces the secure semantics for write operations) in other browsers. A major goal of this API is interoperation with existing cookies, though, so a few examples have also been provided using cookie names lacking these prefixes.

Prefix rules are also enforced in write operations by this API, but may not be enforced in the same browser for other APIs. For this reason it is inadvisable to rely on their enforcement too heavily until and unless they are more broadly adopted.

### URL scoping

Although a service worker script cannot directly access cookies today, it can already use controlled rendering of in-scope HTML and script resources to inject cookie-monitoring code under the remote control of the service worker script. This means that cookie access inside the scope of the service worker is technically possible already, it's just not very convenient.

When the service worker is scoped more narrowly than `/` it may still be able to read path-scoped cookies from outside its scope's path space by successfully guessing/constructing a 404 page URL which allows IFRAME-ing and then running script inside it the same technique could expand to the whole origin, but a carefully constructed site (one where no out-of-scope pages are IFRAME-able) can actually deny this capability to a path-scoped service worker today and I was reluctant to remove that restriction without further discussion of the implications.

### Cookie aversion

To reduce complexity for developers and eliminate the need for ephemeral test cookies, this async cookies API will explicitly reject attempts to write or delete cookies when the operation would be ignored. Likewise it will explicitly reject attempts to read cookies when that operation would ignore actual cookie data and simulate an empty cookie jar. Attempts to observe cookie changes in these contexts will still "work", but won't invoke the callback until and unless read access becomes allowed (due e.g. to changed site permissions.)

Today writing to `document.cookie` in contexts where script-initiated cookie-writing is disallowed typically is a no-op. However, many cookie-writing scripts and frameworks always write a test cookie and then check for its existence to determine whether script-initiated cookie-writing is possible.

Likewise, today reading `document.cookie` in contexts where script-initiated cookie-reading is disallowed typically returns an empty string. However, a cooperating web server can verify that server-initiated cookie-writing and cookie-reading work and report this to the script (which still sees empty string) and the script can use this information to infer that script-initiated cookie-reading is disallowed.

## Non-goals

Some topics have frequently come up in discussion of this proposal but are not explicit goals. They are enumerated here in order to explain why they are not explicit goals of the proposed API.

### Common subset [non-goal]

It is not a goal of this proposal to only allow the common interoperable subset of current `document.cookie` behavior. Doing so would introduce some unacceptable limitations:

- no support for characters outside the printable-ASCII range;
- no support for implicit-`Domain` cookies distinct from explicit-`Domain` cookies;
- no support for cookies larger than the smallest interoperable size; and
- no support for efficient cookie change monitoring.

### Bug-for-bug compatible [non-goal]

On the other hand, it is also not a goal of this proposal to allow all cookie behavior available through any current `document.cookie` implementation. By restricting behavior to a reasonable extension of a useful subset of current behavior, changing defaults and behavior where absolutely necessary, the API can remain straightforward enough to be implementable by many browsers while still offering useful features in a form usable across browsers without major differences in behavior for which content authors/app developers would need to make special allowances.

### Polyfilling [non-goal]

It is not a goal of this proposal to restrict the new API's semantics or implementation such that it could be built entirely in terms of the existing `document.cookie` API.

Most of this API's behavior could be approximated by a [polyfill atop `document.cookie`](cookies.js) for document contexts where that API already exists. There are limits to such an approximation:

- some `document.cookie` operations silently fail in ways not currently easily detectable or predictable by scripts and so a native implementation of the async cookies API would reject the operation when a `document.cookie`-based polyfill would instead incorrectly treat the operation as resolved,
- any `document.cookie`-based polyfill would introduce detectable blocking pauses in the script context's event loop - this is generally undesirable for performance reasons and is one of the motivations for the new API, and
- any `document.cookie`-based polyfill would pay a high performance price for cookie monitoring since it would need to run a high-frequency cookie scanner for change detection.

### API forwarding [non-goal]

It is not a goal of this API to support cross-context API forwarding, but the present form of the proposal does not prevent it either.

This API is currently designed:

- with Promises ⇔ async functions for all operations, 
- with neither opaque nor nontransferable data types,
- with no thread-blocking operations,
- with no strong transactional guarantees,
- with no guarantees of predictable realtime performance,
- with no explicit object sealing or other API-level tamper-resistance,
- with no custom getters/setters,
- with no explicit timing guarantees about when exactly changes made through this API will take effect and
- with no explicit coherency guarantees about read operations in the presence of overlapping reads or writes.

So long as these continue to hold, a replacement or a shim using postMessage or a similar cross-context messaging primitive and exposing a CookieStore to a different execution contexts (e.g. to a sandboxed IFRAME) using the same API should be possible.

Any forwarding of the API should be done bearing in mind the implications (including security and privacy, among others) of the forwarding and ensuring the forwarding implementation does not open up new security holes or violate privacy expectations of site visitors/app users.
