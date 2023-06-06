# Explainer: Locale Extensions 

## Table of Contents

- [Explainer:  Locale Extensions](#explainer-locale-extensions)
  - [Table of Contents](#table-of-contents)
  - [Authors:](#authors)
  - [Participate](#participate)
  - [Introduction](#introduction)
  - [Common User Preferences](#common-user-preferences)
  - [Client Hints](#client-hints)
      - [`Client Hint` Header Fields](#client-hint-header-fields)
      - [Usage Example](#usage-example)
    - [Javascript API](#javascript-api)
      - [IDL](#idl)
      - [Proposed Syntax](#proposed-syntax)
  - [Privacy and Security Considerations](#privacy-and-security-considerations)

## Authors:

- [Romulo Cintra](https://github.com/romulocintra)
- [Ujjwal Sharma](https://github.com/ryzokuken)
- [Ben Allen](https://github.com/ben-allen)

## Participate
- [GitHub repository](/)
- [Issue tracker](/issues)


## Introduction

Unicode Extensions for BCP 47 can be used to append additional information capturing these settings to the end of language identifiers. Enabling support for BCP tags can help solve cases like:

>Astrid works with a Swedish company and uses 'sv', but is most comfortable having dates and times displayed as in 'en-US'. Her OS is set to display time and measurements using these formats, and would find websites that use more accessible. 

>Developers of a NodeJS program may want to respect tailorings related to numbering system, as programs using non-preferred numbering systems may become unintellible to users. 


For **client-side applications**, the best way to get these preferences is through a browser API that fetches this information from the different platform-specific APIs. 

For **server-side applications**, one way to access this information is through the use of a [[!CLIENT-HINTS]] header on the request signalling that Unicode Extensions are to be used.  

### Common User Preferences 
The following table suggests a minimal set of commonly used locale extensions to be supported:
<table>
  <tr><td>"hourCycle"<td>`hc`<td>`h12`, `h23`, `auto`<td>12-hour or 24-hour hour cycle</tr>
  <tr><td>"numberingSystem"<td>`nu`<td>`latn`, `native`, `auto`<td>Preferred numbering system</tr>
  <tr><td>"measurementUnit"<td>`mu`<td>`celcius`, `fahrenheit`, `auto`<td>Measurement unit for temperature</tr>
  <thead><tr><th>Locale Extension Name<th>Unicode Extension Key<th>Possible values<th>Description</thead>
</table>

> Note: The full set of extensions ultimately included need to be validated and agreed to by security teams and stakeholders.


## Client Hints ##

An <a href="https://datatracker.ietf.org/doc/rfc8942/">HTTP Client Hint</a> is a request header field that is used by HTTP clients can be used by the server to optimize content served to those clients. The Client Hints infrastructure defines an `Accept-CH` response header that servers can use to advertise their use of specific request headers for proactive content negotiation. This opt-in mechanism enables clients to send content adaptation data selectively, instead of appending all such data to every outgoing request. 

Because servers must specify the set of headers they are interested in receiving, the Client Hint mechanism eliminates many of the opportunities for hostile passive fingerprinting that arise when using other means for proactive content negotiation (for example, the User-Agent string). 


### `Client Hint` Header Fields

Servers cannot passively receive information about locale extension-related settings. Servers instead advertise their ability to use extensions, allowing clients the option to respond with preferred content tailorings. 

To accomplish this, browsers should introduce new `Client Hint` header fields as part of a structured header as defined in <a href="https://tools.ietf.org/html/draft-ietf-httpbis-header-structure-19">Structured Field Values for HTTP</a>.

<table>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-Hour-Cycle`</dfn><td>`Sec-CH-Locale-Extensions-Hour-Cycle`  : "h24"</tr>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-Numbering-System`</dfn><td>`Sec-CH-Locale-Extensions-NumberingSystem`  : "native"</tr>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-MeasurementUnit`</dfn><td>`Sec-CH-Locale-Extensions-MeasurementUnit` : "auto"</tr>

  <thead><tr><th style=text:align left>Client Hint<th>Example output</thead>
</table>

### Usage Example

<div class=example>

1. The client makes an initial request to the server:

```http
GET / HTTP/1.1
Host: example.com
```

2. The server responds, telling the client via an `Accept-CH` header (Section 2.2.1 of [[!RFC8942]]) along with the initial response with `Sec-CH-Locale-Extensions-NumberingSystem`, indicating that the server accepts that particular Client Hint and no others.

```http
HTTP/1.1 200 OK
Content-Type: text/html
Accept-CH: Sec-CH-Locale-Extensions-NumberingSystem
```

3. Subsequent requests to https://example.com will include the following request headers in case the user sets `numberingSystem`.

```http
GET / HTTP/1.1
Host: example.com
Sec-CH-Locale-Extensions-NumberingSystem: "native" 
```

4. The server can then tailor the response accordingly. For example, if the current locale is 'hi-IN', the server could generate content with numbers represented in Devanagari numerals.

</div>

## JavaScript API 

These client hints should also be exposed as JavaScript APIs via `navigator.locales`, or by creating a new `navigator.localeExtensions` information as below:

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
self.navigator.localePreferences.languageRegion;
// Output => => {numberingSystem: "latn"}

navigator.localeExtensions['measurementUnit'];
navigator.localeExtensions.measurementUnit;
self.navigator.localePreferences.measurementUnit;
// Output => => {measurementUnit: "celcius"}

navigator.localeExtensions['hourCycle'];
navigator.localeExtensions.hourCycle;
self.navigator.localePreferences.hourCycle;
// Output => => {measurementUnit: "h12"}

// Window or WorkerGlobalScope even</a>t

window.onlocaleextensions = (event) => {
  console.log('localeextensions event detected!');
};

// Or

window.addEventListener('localeextensions', () => {
  console.log('localeextensions event detected!');
});

```

## Privacy and Security Considerations

There are some concerns that exposing this information would give trackers, advertisers and malicious web services another fingerprinting vector. That said, this information may or may not already be available to a certain extent to such services, based on the host and the user’s settings. The use of `Sec-CH-` prefix is to forbid access to these headers containing `Locale Preferences` information from JavaScript, and demarcate them as browser-controlled client hints so they can be documented and included in requests without triggering CORS preflights.

Client Hints provides a powerful content negotiation mechanism that enables us to adapt content to users' needs without compromising their privacy. It does that by requiring server opt-in, which guarantees that access to the information requires active and tracable action on the server's side. As such, the mechanism does not increase the web's current active fingerprinting surface. The [Security Considerations](https://datatracker.ietf.org/doc/html/rfc8942#section-4) of HTTP Client Hints and the [Security Considerations](https://tools.ietf.org/html/draft-davidben-http-client-hint-reliability-02#section-5) of Client Hint Reliability likewise apply to this proposal.



