Explainer: Locale Extensions 

## Table of Contents

- [Explainer:  Locale Extensions](#explainer-locale-extensions)
  - [Table of Contents](#table-of-contents)
  - [Authors:](#authors)
  - [Participate](#participate)
  - [Motivation](#motivation)
  - [Overview](#overview)
  - [Common Locale Extensions](#common-locale-extensions)
  - [JavaScript API](#agent-driven-negotiation-javascript-api)
      - [IDL](#idl)
      - [Proposed Syntax](#proposed-syntax)
  - [Client Hints](#proactive-content-negotiation-with-client-hints) 
      - [`Client Hint` Header Fields](#client-hint-header-fields)
      - [Usage Example](#usage-example)
  - [Privacy and Security Considerations](#privacy-and-security-considerations)
  - [FAQ](#faq)

#proactive-content-negotiation-with-client-hints#

## Authors:

- [Romulo Cintra](https://github.com/romulocintra)
- [Ujjwal Sharma](https://github.com/ryzokuken)
- [Ben Allen](https://github.com/ben-allen)

## Participate
- [GitHub repository](/)
- [Issue tracker](/issues)


## Motivation

Frequently web application users need content localized in ways that partially diverge from the defaults used in their language or region. Mismatches between desired and actually delivered content can sometimes result in user frustration or annoyance &mdash; to give one common example, consider the error-prone mental math involved in converting a temperature measured in the Fahrenheit scale when one is more familiar with Celcius. Beyond this,  failure to deliver the desired content tailorings may result in the content becoming inaccessible or even completely unintelligible to some users. This may occur, for example, for users whose preferred numbering system differs from the numbering system used by default in their locale/region pair. 

In the native environment these problems do not occur, since users can specify these desired customizations in their system settings. However, the full amount of flexibility allowed for in the native environment is not possible in the web environment. This proposal defines a mechanism by making a limited subset of the Unicode Extensions for BCP 47 available for content negotiation, providing options that address some of the worst problems with incomplete localization while only exposing coarse-grained data about the users who take advantage of these improvements.


## Overview 

Unicode Extensions for BCP 47 can be used to append additional information needed to identify locales to the end of language identifiers. Enabling support for a subset of BCP tags can help solve problems like the ones below:

1. Currently en-US is the typical untranslated language for software, even though en-US's region-specific formatting patterns differ from those used globally. As a result, often text with untranslated UI strings will be displayed in a language accessible to all users who speak English, but with temperatures represented in Fahrenheit, a scale that is confusing and unfamiliar to users from regions that use Celcius. 

2. In many regions both Latin and Arabic-Indic numerals are in common use. Users in these regions may find one or the other of these numbering systems not immediately intelligible, and desire content tailored to the numbering system with which they are most familiar. 

For **client-side applications**, the best way to get these preferences is through a browser API that fetches this information from the different platform-specific APIs. 

For **server-side applications**, one way to access this information is through the use of a Client Hints header on the request, signalling that Unicode Locale Extensions are to be used. 

In both of these cases the browser conveys data related to the user's operating system settings to servers, but only sends a tightly limited subset of that data. As currently proposed, exactly three potential locale extension settings are exposed, with each of these settings having at most three possible options. Neither case allows for passive fingerprinting: in order to read these settings, servers must either advertise their intent to use each individual setting via an `Accept-CH` header or else issue a detectable query to the client.

### Common Locale Extensions
The following table suggests a minimal set of commonly used locale extensions to be supported. Note that the list of supported possible values for each extension is exhaustive &mdash; limiting the range of available options to a few sensible values helps mitigate privacy and security concerns related to providing servers with preferred content tailorings.

<table>
  <tr><td>"hourCycle"<td>`hc`<td>`h12`, `h23`, `auto`<td>Preferred hour cycle</tr>
  <tr><td>"numberingSystem"<td>`nu`<td>`latn`, `native`, `auto`<td>Preferred numbering system</tr>
  <tr><td>"measurementUnit"<td>`mu`<td>`celcius`, `fahrenheit`, `auto`<td>Measurement unit for temperature</tr>
  <thead><tr><th>Locale Extension Name<th>Unicode Extension Key<th>Possible values<th>Description</thead>
</table>


> Note: The full set of extensions ultimately included need to be validated and agreed to by security teams and stakeholders. 


## Agent-Driven Negotiation: JavaScript API 

The locale extensions preferred by the user should be exposed as JavaScript APIs via `navigator.locales`, or by creating a new `navigator.localeExtensions` property. 

### IDL 

```
dictionary LocaleExtensions {
  DOMString measurementUnit;
  DOMString numberingSystem;
  DOMString hourCycle;
};

interface mixin NavigatorLocaleExtensions {
  readonly attribute LocaleExtensions localeExtensions;
};

Navigator includes NavigatorLocaleExtensions;
WorkerNavigator includes NavigatorLocaleExtensions;
```

### Proposed Syntax

```js

navigator.localeExtensions['numberingSystem'];
navigator.localeExtensions.numberingSystem;
self.navigator.localeExtensions.numberingSystem;
// Output => => "latn"

navigator.localeExtensions['measurementUnit'];
navigator.localeExtensions.measurementUnit;
self.navigator.localeExtensions.measurementUnit;
// Output => => "celcius"

navigator.localeExtensions['hourCycle'];
navigator.localeExtensions.hourCycle;
self.navigator.localeExtensions.hourCycle;
// Output => => "h12"

// Window or WorkerGlobalScope event

window.onlocaleextensions = (event) => {
  console.log('localeextensions event detected!');
};

// Or

window.addEventListener('localeextensions', () => {
  console.log('localeextensions event detected!');
});

```


## Proactive Content Negotiation With Client Hints ##

An <a href="https://datatracker.ietf.org/doc/rfc8942/">HTTP Client Hint</a> is a request header field that is sent by HTTP clients and used by servers to optimize content served to those clients. The Client Hints infrastructure defines an `Accept-CH` response header that servers can use to advertise their use of specific request headers for proactive content negotiation. This opt-in mechanism enables clients to send content adaptation data selectively, instead of appending all such data to every outgoing request. 

Because servers must specify the set of headers they are interested in receiving, the Client Hint mechanism eliminates many of the opportunities for hostile passive fingerprinting that arise when using other means for proactive content negotiation (for example, the `User-Agent` string). 


### `Client Hint` Header Fields

Servers cannot passively receive information about locale extension-related settings. Servers instead advertise their ability to use extensions, allowing clients the option to respond with their preferred content tailorings. 

To accomplish this, browsers should introduce new `Client Hint` header fields as part of a structured header as defined in <a href="https://tools.ietf.org/html/draft-ietf-httpbis-header-structure-19">Structured Field Values for HTTP</a>.

<table>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-Hour-Cycle`</dfn><td>`Sec-CH-Locale-Extensions-Hour-Cycle`  : "h23"</tr>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-Numbering-System`</dfn><td>`Sec-CH-Locale-Extensions-NumberingSystem`  : "native"</tr>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-MeasurementUnit`</dfn><td>`Sec-CH-Locale-Extensions-MeasurementUnit` : "auto"</tr>

  <thead><tr><th style=text:align left>Client Hint<th>Example output</thead>
</table>

The `Sec-` prefix used on these headers prevents scripts and other application content from setting them in user agents, and demarcates them as browser-controlled client hints so that they can be documented and included in requests without triggering CORS preflights. See [HTTP Client Hints Section 4.2, Deployment and Security Risks](https://datatracker.ietf.org/doc/html/rfc8942#section-4.2) for more information. 

### Usage Example

<div class=example>

1. The client makes an initial request to the server:

```http
GET / HTTP/1.1
Host: example.com
```

2. The server responds, sending along with the initial response an `Accept-CH` header (see [HTTP Client Hints Section 3.1, The `Accept-CH` Response Header Field](https://datatracker.ietf.org/doc/html/rfc8942#section-3.1)) with `Sec-CH-Locale-Extensions-NumberingSystem`. This response indicates that the server accepts that particular Client Hint and no others. 

```http
HTTP/1.1 200 OK
Content-Type: text/html
Accept-CH: Sec-CH-Locale-Extensions-NumberingSystem
```

3. Subsequent requests to https://example.com will include the following request headers in case the user agent sets `numberingSystem`.

```http
GET / HTTP/1.1
Host: example.com
Sec-CH-Locale-Extensions-NumberingSystem: "native" 
```

4. The server can then tailor the response accordingly. For example, if the current locale is `hi-IN`, the server could provide content with numbers represented using Devanagari numerals. 

Note that servers **must** ignore hints that they do not support. 
</div>

## Privacy and Security Considerations

There are two competing requirements at play when localizing content in the potentially hostile web environment. One is the need to make content accessible to and usable by people from as wide a range of linguistic and cultural contexts as possible. The other, equally important, is the need to preserve the safety and privacy of users. Often these two pressures appear diametrically opposed, since content negotiation necessarily requires revealing information about users.

The [Mitigating Browser Fingerprinting in Web Specifications](https://www.w3.org/TR/fingerprinting-guidance/#fingerprinting-mitigation-levels-of-success) W3C document identifies the following key elements for fingerprint mitigation: 

1. Decreasing the fingerprinting surface
2. Increasing the anonymity set
3. Making fingerprinting detectable (i.e. replacing passive fingerprinting methods with active ones) 
4. Clearable local state

The preservation of a relatively large anonymity set is our central strategy for mitigating fingerprinting risk as much as possible while also ensuring a substantial improvement in the localization experience for a wide range of users. 

* 'hourCycle' always provides users with both of the most commonly used hour cycles plus their region's default, should that default differ from "h12" and "h23", while protecting users from making themselves immediately individually indentifiable by revealing that they use a particularly uncommon hour cycle for their region.
* 'measurementUnit' already allows only three options.
* 'numberingSystem' has its set of available options limited to just "latn", "native", and "auto." This reduction allows users to choose between up to three numbering systems that are likely to be legible to them, extending the full set of three options available in language/region pairs that have a "native" numbering.
system and also a "default" numbering system that differs from both the "native" numbering system and from "latn". This compromise extends to as many users as possible the ability to specify an intelligible numbering system, without allowing for selections that are likely to immediately uniquely identify users. 

As noted in the [Security Considerations](https://datatracker.ietf.org/doc/html/rfc8942#section-4) section of the HTTP Client Hints RFC, a key benefit of the Client Hints architecture is that it allows for proactive content negotiation without exposing passive fingerprinting vectors, becuase servers must actively advertise their use of specific Client Hints headers. This makes it possible to remove preexisting passive fingerprinting vectors and replace them with relatively easily detectable active vectors. The Detectability section of [Mitigating Browser Fingerprinting in Web Specifications](https://www.w3.org/TR/fingerprinting-guidance/#detectability) describes instituting requirements for servers to advertise their use of particular data as a best practice, and mentions Client Hints as a tool for implementing this practice. In the absence of Client Hints, use of the JavaScript API can at least be detected by clients. In no case does this proposal allow for any new passive fingerprinting vectors. 

The use of the `Sec-` prefix forbids access to headers containing `Locale Extensions` information from JavaScript, and demarcates them as browser-controlled client hints so that they can be documented and included in requests without triggering CORS preflights. 

As in all uses of Client Hints, user agents must clear opt-in Client Hints settings when site data, browser caches, and cookies are cleared.

Implementations may also include additional fingerprinting mitigations. For example, clients could restrict the number of Locale Extension settings sent by users who already have a small anonymity set, either because of other information exposed by the client or because of being in a language/region pair with relatively fewer users. In this case preference would be given to always sending those headers most likely to impact content intelligibility. This ensures that as many users as possible can take advantage of crucial localization settings without making themselves individually identifiable. Additionally, user agents can make decisions about what to reveal to what sites; cross-origin sites, for example, could be allowed access to less data than top-level sites the user visits frequently. 

This proposal could help reduce the fingerprinting surface as a whole by providing a mechanism to get many of the benefits of the `Accept-Language` header without that header's passive fingerprinting implications.


## FAQ

### Why this specific selection of tags?

We've selected a best-guess set of tags and available options for those tags based on the following criteria:

1. Would revealing OS settings related to this tag result in major intelligibility benefits for users?
2. Would revealing OS settings related to this tag result in the reduction of annoyances for a large group of users?
3. Can we approach these goals while only revealing relatively coarse-grained information about users?

There are additional tags which can be considered: for example, `fw` is a candidate, presumably with valid options limited to the first day of week options most commonly used. `ms` would be a strong candidate, were it not slated to be deprecated. 

User research may be useful in determining what sets of tags are best for negotiating the balance between more precise content tailorings and the potential fingerprinting risks.

### What about clients that don't implement Client Hints?

Using Locale Extensions is still possible through the JavaScript API. Use of the API may present small drawbacks inherent to agent-side content negotiation in general &mdash; the extra request required, etc. 

### Maintaining a large anonymity set for users in smaller language/region pairs?

A user of `en-US` or `zh` is, all else being equal, going to be more anonymous than users of less-ubiquitous language/region pairs. As a result, it is significantly easier to provide a larger range of options to these users; even if we split them up into distinct smaller anonymity sets, it is still likely that they can hide in a crowd. This leads to the unpleasant conclusion that mitigating fingerprint risk may require allowing users of common language/region pairs significantly more control over their desired content tailorings than users of less common ones. Should it prove necessary to provide these users fewer options, preference should be given to the locale extensions that are most likely to directly affect content intelligibility.

### Alternate strategies to consider?

An alternate strategy for determining what options are available may involve allowing option selections that tend to correlate with each other (for example, the use of "h23" for hour cycle and "celcius" for measurement unit) to be set together, but not separately. User research would be required to determine the most needed sets of commonly-seen-together selections. 

One way to implement bundles of preferences that tend to track together may be to implement commonly used values for the `rg` Unicode Extension, which could allow users to (for example) set their locales to "en-US-u-rg-gbzzzz", which would result in content in US English but with region-specific defaults set to those of British English. 

### Options that aren't captured by Unicode Extensions for BCP 47

There exist other localization-related customizations that would be useful for site intelligibility and that could potentially be constrained to a small number of options -- notably, number separators and number patterns. Support for a commonly used subset of these options could be possible.  
