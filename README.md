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

Unicode Extensions for BCP 47 can be used to append additional information capturing these settings to the end of language identifiers. Enabling support for a subset of BCP tags can help solve problems like the ones below:

> Currently en-US is the typical untranslated language for software. As a result, often text with untranslated UI strings will be displayed in a language accessible to all users who speak English, but with hours of the day represented in a non-preferred 12 hour format and with temperatures represented in a confusingly unfamiliar measuring system.

> In many regions both Latin and Arabic-Indic numerals are in common use. Users in this region may find one or the other of these numbering systems unintelligible. 

For **client-side applications**, the best way to get these preferences is through a browser API that fetches this information from the different platform-specific APIs. 

For **server-side applications**, one way to access this information is through the use of a [[!CLIENT-HINTS]] header on the request signalling that Unicode Extensions are to be used.  

### Common User Preferences 
The following table suggests a minimal set of commonly used locale extensions to be supported:
<table>
  <tr><td>"hourCycle"<td>`hc`<td>`h12`, `h23`, `auto`<td>12-hour or 24-hour hour cycle</tr>
  <tr><td>"numberingSystem"<td>`nu`<td>`latn`, `native`, `auto`<td>Preferred numbering system (Note restricted number of options)</tr>
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


--

There are two competing requirements at play when localizing content in the potentially hostile web environment. One is the need to make content and applications accessible and usable in as broad a range of linguistic and cultural contexts as possible. The other, equally important, is the need to preserve the safety and privacy of users. Often these two pressures appear diametrically opposed, since proactive content negotiation inevitably requires revealing information that can be used to uniquely identify users.


The [Mitigating Browser Fingerprinting in Web Specifications](https://www.w3.org/TR/fingerprinting-guidance/#fingerprinting-mitigation-levels-of-success) W3C document identifies the following key elements for fingerprint mitigation: 

1. Decreasing the fingerprinting surface
2. Increasing the anonymity set
3. Making fingerprinting detectable (i.e. replacing passive fingerprinting methods with active ones) 
4. Clearable local state


As noted in the [Security Considerations](https://datatracker.ietf.org/doc/html/rfc8942#section-4) section of HTTP Client Hints, a key benefit of the Client Hints architecture is that it allows for proactive content negotiation without exposing passive fingerprinting vectors, since servers must actively advertise their use of specific Client Hints headers. This makes it possible to remove preexisting passive fingerprinting vectors -- for example, the User Agent string -- and replace them with relatively easily detectable active vectors. 

The Detectability section of [Mitigating Browser Fingerprinting in Web Specifications](https://www.w3.org/TR/fingerprinting-guidance/#detectability) describes instituting requirements for servers to advertise their use of particular data as a best practice, and mentions Client Hints as a tool for implementing this practice.  

This proposal builds on the potential privacy benefits provided by Client Hints by restricting the available set of locale extension headers to a selection that only exposes low-granularity information. This results in a relatively small reduction of the size of the anonymity set. Both 'hourCycle' and 'measurementUnit' have three options apiece, as does 'numberingSystem', due to the reduction of available numbering system options to just "latn", "native", and "auto." This reduction allows users to choose between up to three numbering systems that are likely to be legible to them, without allowing for selections that may make them highly likely to become uniquely identifiable.

Implementations may also include other fingerprinting mitigations. For example, clients could restrict the number of Locale Extensions Client Hints sent by users who already have a small anonymity set, with preference given to sending those headers most likely to impact content intelligibility. This ensures that as many users as possible can take advantage of these localization features without making themselves individually identifiable. 

As in all uses of Client Hints, user agents must clear opt-in Client Hints settings when site data, browser caches, and cookies are cleared.




