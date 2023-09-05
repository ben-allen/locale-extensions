## Table of Contents

- [Explainer:  Locale Extensions](#explainer-locale-extensions)
  - [Table of Contents](#table-of-contents)
  - [Authors](#authors)
  - [Participate](#participate)
  - [Motivation](#motivation)
  - [Overview](#overview)
  - [Example Problem #1 and Proposed Solution](#example-problem-1-and-proposed-solution)
    - [Why not -u-rg?](#why-not--u-rg)
  - [Example Problem #2](#example-problem-2)
    [Solution: Locale Preference Strings](#solution-locale-preference-strings)
  - [Agent-Driven Negotiation: JavaScript API](#agent-driven-negotiation-javascript-api)
      - [IDL](#idl)
      - [Proposed Syntax](#proposed-syntax)
  - [Proactive Content Negotiation with Client Hints](#proactive-content-negotiation-with-client-hints) 
      - [`Client Hint` Header Fields](#client-hint-header-fields)
      - [Usage Example](#usage-example)
  - [Privacy and Security Considerations](#privacy-and-security-considerations)
  - [FAQ](#faq)

## Authors:

- [Ben Allen](https://github.com/ben-allen)

## Participate
- [GitHub repository](/)
- [Issue tracker](/issues)


## Motivation

On the Web platform, content localization is dependent only upon a user's language or region. However, this behavior can result in annoyance, frustration, offense, or even unintelligibility for some users. This proposal addresses common cases where users prefer locale-related tailorings that differ from the defaults in their locale. Consider the following problems:

1. A Spanish-speaking person from the United States has moved to Spain, and would prefer if local news sites would display weather forecasts using temperatures measured in Fahrenheit rather than Celsius.
2. An English-speaking user from Japan receives non-localized content in 'en'. They can read English, but nevertheless prefer seeing a 24-hour clock and calendars that have Monday, rather than Sunday, as the first day of the week. 
3. In general, 'en-US' is currently the typical untranslated language for software, even though 'en-US' has region-specific formatting patterns that differ from those used globally. As a result, often text with untranslated UI strings will be displayed in a language accessible to all users who speak English, but with temperatures and times represented in globally uncommon scales.
4. A user who has emigrated from one country to another sets their language dialect to one they can understand, but prefers that dates, times, and numbers be rendered according to local standards.
5. In some geographical regions multiple numbering systems are in use. For example, both Western Arabic (Latin) and Eastern Arabic numerals are in common use in Iran and Afghanistan. Users in these regions may find one or the other of these numbering systems not immediately intelligible, and therefore desire content tailored to the numbering system with which they are most familiar.
6. Western Arabic (Latin) numerals are used by default in 'hi-IN', even though many users find Devanāgarī numerals more intelligible.

In the native environment these problems do not occur, since users can specify these desired customizations in their system settings. However, offering the full amount of flexibility allowed in the native environment is not possible in the often hostile web environment &mdash; if a user's preferences as specified in their OS settings are at all uncommon, servers can use them to fingerprint that user.

This proposal defines a mechanism to let clients to read user preferences from their operating system and then relay a subset of those preferences to servers, while refraining from sending combinations of preferences that are likely to individuate those users.  This allows for significantly more complete &mdash; but not necessarily perfect &mdash; localization while only exposing relatively coarse-grained data about users. 

## Overview 

Unicode Extensions for BCP 47 can be used to append additional information needed to identify locales to the end of language identifiers. Enabling support for a subset of BCP tags can help solve problems like the ones above.

For **client-side applications**, the best way to get these preferences is through a browser API that fetches this information from the different platform-specific APIs. 

For **server-side applications**, one way to access this information is through the use of a `Client Hints` header on the request, signalling that Unicode Locale Extensions are to be used. 

Both of these mechanisms allow servers to receive data related to the user's operating system preferences. However, we do not expose *all* of the user's preferences. Instead, we determine what combinations of specific settings are in common use in each given locale, and only expose preferences that can be fit into one of those commonly-used combinations. This allows for significantly improved content tailoring, without exposing the type of relatively uncommon combinations of preferences that would allow servers to easily individuate users.

### Example Problem #1 and Proposed Solution

Consider the case of a student from the Netherlands who is spending a year at a university in Chicago. This student is likely a native-level English speaker, but could avoid annoyance and potential error if the university's course catalog displayed times on a 24 hour cycle instead of using the 12 hour clock common in the United States, and would likewise prefer if calendars be displayed with Monday, rather than Sunday, as the first day of the week. Additionally, they are not acclimated to Chicago's extreme winters, and use the weather display on a local news site to determine how many layers of clothes to wear. They would very much like to avoid the frustration and potential for mishap involved in converting temperatures from the unfamiliar Fahrenheit scale to the immediately comprehensible Celsius scale. 

It is possible to express this set of preferences using Unicode Locale Extension tags: 

'-u-fw-mon-hc-h23-mu-celsius'

Our example user's native applications read their preferences from the operating system and display dates, times, and temperatures, but user preferences are (wisely) shielded from potentially hostile web applications.

Nevertheless, if our hypothetical student were able to send this particular set of preferences to web sites, they likely could not be used to individuate them.  After all, those preferences are common across Europe and much of the globe, and so it is likely that there are sufficient other users who share our hypothetical student's preferences that none of the members of this group can be distinguished from any other in the group through inspection of these preferences. 

As our aim is to improve content localization, we can safely ignore highly idiosyncratic settings. For example, if a user's preferences as read from the operating system can be described by the locale extension string 'u-ca-dangi-hc-h11-mu-kelvin', meaning "give me the traditional Korean calendar, a clock that runs from 0 to 11, and temperatures in Kelvin," allowing servers to see those preferences would immediately make that user individually identifiable. Fortunately, there is no internationalization-related reason that we should accommodate that very rare combination of settings, and so we can safely disallow it.


#### Why not -u-rg?

Unicode Locale Extensions provides the -u-rg Region Override tag, which is used to represent a locale modified with region-specific defaults from another locale. For example, 'en-GB-u-rg-uszzzz' means British English, but with hour cycle, first day of week, measurement system, and so forth set as in 'en-US'. This is, unfortunately, not particularly well-adapted for our purposes: saying '-u-rg-nlzzzz' is in a way equivalent to saying '-u-fw-mon-hc-h23-mu-celsius', but with the disadvantage that '-u-rg-nlzzzz' reveals that the user prefers the formatting in specifically 'nl'. This exposes significantly more information about the user than if they were instead able to select the set of preferences that match those used in 'nl' &mdash; a set of preferences that are found in many locales &mdash; without revealing that the user specifically desires content tailored as in 'nl'.

Moreover, a system that explicitly specifies each setting rather than naming a region and requesting the defaults for that region allows us to accommodate the preferences of users who have system settings that differ from the defaults for their language/region pair. Consider the case of a user living in a region with multiple commonly used numbering systems. There may not exist any non-extended locale identifier that captures their preference for a particular non-default numbering system, but there may be a sufficiently large group of users who would prefer receiving content using that numbering system that they are all in a large enough anonymity set. Solutions depending on -u-rg do not allow these users to specify this preference. 


### Example Problem #2

It is possible that our hypothetical European student could have their entire set of locale-related preferences honored while in 'en-US', due to the relatively large number of other people in that locale who also prefer '-u-fw-mon-hc-h23-mu-celsius'.  However, users from locales with less common combinations of default locale-related content tailoring settings cannot count on having a similar crowd to hide in. For example, a businessperson from Thailand visiting Mexico may prefer something like '-u-ca-buddhist-fw-mon-hc-h23'. Although two of these options are shared by a significant number of other people, the request for the Thai solar calendar rather than the Gregorian calendar is sufficiently rare in the 'es-MX' locale that it may be possible to individuate that user from that setting alone. This means that we cannot safely set this full combination of options for this user. Nevertheless, we would like to provide them with content tailored to as many of their preferences as possible.

### Solution: Locale Preference Strings

We propose the following broad-strokes approach for accomodating as many of the user's preferences as possible while revealing as little information about them as possible:

* Each locale must allow the use of some number of locale extension strings, with the number and selection of available locale extension strings dependent upon demand for that combination of settings. If a significant number of users prefer a particular locale extension string, it is possible to allow access to that combination of settings while still allowing users to maintain a sufficiently large anonymity set. This string will therefore be valid.

* Valid locale extension strings are selected to, insofar as is possible, honor at least some part of the preferences of users from any locale. Users whose preferences are rare enough that expressing them in full would likely make them individuatable can be assigned a locale extension string that comes as close as possible to expressing their full preferences while exposing them to as little risk as possible. A user reading 'en-US' content who nevertheless prefers content tailorings as in 'th' could get most (but not all) of their preferences met by assigning them the locale extension string '-u-fw-mon-hc-h23-mu-celsius', even if their request for 'ca-buddhist' could not be met.

* The set of available locale extension strings must be individually constructed for each base locale, since the likely combinations of locale-related alternate preferences will vary from locale to locale. To use the 'th' example, it is possible that locales with regional or cultural connections to Thailand would in fact have enough users who prefer the Thai solar calendar for 'ca-buddhist' to appear in one or more locale extension strings. Likewise, it is possible that locales with a very large number of users &mdash; for example, 'en-US' &mdash; have enough users who prefer 'ca-buddhist' for this option to become available.

* Clients read user preferences the operating system, and compare them to the set of available locale extension strings for the document's locale. The only preferences that servers see are those expressible through one of the available locale extension strings for that locale. Client behaviour that conveys user locale preferences beyond those in the set of valid strings is considered non-normative.

* Clients that allow locale extension strings that, when used, are likely to reduce the user's anonymity set to shrink to a size such that the user may become easily individuatable are considered non-normative.

* Care must be taken in the construction of the set of valid locale extension strings, based on the impact on content intelligibility made by respecting or not respecting specific preferences. Ignoring some preferences can be annoying to users, but ignoring others (for example, preferred numbering system) can result in some users receiving unintelligible content. Preferences must therefore receive different weights, based on the likelihood that content will become unintelligible if that preference is ignored. Most notably, there are a number of locales in which numbering systems differing from the locale default are commonly used. In these locales support for locale extension strings requesting use of those commonly used alternate numbering systems is paramount. 

## Agent-Driven Negotiation: JavaScript API 

### IDL 
We expose the preferred options for these extensions in a JavaScript API via 'navigator.locales' or by creating a new 'navigator.localeExtensions' property: 

### IDL

```
interface LocaleExtensions localeExtensions {
  readonly attribute DOMString calendar;
  readonly attribute DOMString firstDayOfWeek;
  readonly attribute DOMString hourCycle;
  readonly attribute DOMString temperatureUnit;
  readonly attribute DOMString numberingSystem;
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
self.navigator.numberingSystem;
// "deva"

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

Servers cannot passively receive information about locale extension-related settings. Servers instead announce their ability to use extensions, allowing clients the option to respond with their preferred content tailorings.

To accomplish this, browsers should introduce new `Client Hint` header fields as part of a structured header as defined in <a href="https://tools.ietf.org/html/draft-ietf-httpbis-header-structure-19">Structured Field Values for HTTP</a>.

<table>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-Calendar`</dfn><td>`Sec-CH-Locale-Extensions-Calendar`  : "gregory"</tr>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-FirstDay`</dfn><td>`Sec-CH-Locale-Extensions-FirstDay`  : "mon"</tr>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-HourCycle`</dfn><td>`Sec-CH-Locale-Extensions-HourCycle`  : "h23"</tr>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-MeasurementUnit`</dfn><td>`Sec-CH-Locale-Extensions-MeasurementUnit` : "fahrenhe"</tr>
  <tr><td><dfn export>`Sec-CH-Locale-Extensions-NumberingSystem`</dfn><td>`Sec-CH-Locale-Extensions-NumberingSystem`  : "deva"</tr>

  <thead><tr><th style=text:align left>Client Hint<th>Example output</thead>
</table>

Designing the Client Hint header fields requires a tradeoff between fingerprinting mitigation and using a parsimonious set of headers. The approach that best prevents fingerprinting is to give each separate tag its own Client Hint header. Since servers must advertise their use of each header, fully separating the tags makes fingerprinting attempts more obvious &mdash;  a server that requests a large number of Client Hints without need is broadcasting its potential intent to use the information gathered from the client for the purpose of fingerprinting. However, if header bloat becomes a primary concern, some of these headers can be grouped. For example, `hc`, `fw` and `ca` could be grouped together as preferences related to date and time, or `fw`, `hc`, and `mu` could be grouped due not to conceptual similarity but instead to how they are strongly correlated with each other, as users following United States regional standards are likely to want `-u-fw-sun-hc-h12-mu-fahrenhe` while users in much of the rest of the world are likely to want `-u-fw-mon-hc-h23-mu-celsius`. 

Should the ability to customize settings beyond those expressible through BCP 47 tags become incorporated into this proposal, grouping will necessarily become a more pressing concern. For example, should additional preferences related to number formatting become part of the proposal, these could be grouped together with `nu`. 

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

3. If the user's preferred numbering system differs from the defaults for the locale &mdash; in this case, the user prefers Devanāgarī numerals &mdash; subsequent requests to https://example.com will include the following request headers.

```http
GET / HTTP/1.1
Host: example.com
Sec-CH-Locale-Extensions-NumberingSystem: "deva"
```

4. The server can then tailor the response accordingly. 

Note that servers **must** ignore hints that they do not support. Note also that although each of the locale extension preferences can be accessed individually, no `Client Hint` can be sent unless it is consistent with one of the valid locale extension strings for the content's locale.
</div>


## Privacy and Security Considerations

There are two competing requirements at play when localizing content in the potentially hostile web environment. One is the need to make content accessible to and usable by people from as wide a range of linguistic and cultural contexts as possible. The other, equally important, is the need to preserve the safety and privacy of users. Often these two pressures appear diametrically opposed, since content negotiation necessarily requires revealing information about users.

The [Mitigating Browser Fingerprinting in Web Specifications](https://www.w3.org/TR/fingerprinting-guidance/#fingerprinting-mitigation-levels-of-success) W3C document identifies the following key elements for fingerprint mitigation:

1. Decreasing the fingerprinting surface
2. Increasing the anonymity set
3. Making fingerprinting detectable (i.e. replacing passive fingerprinting methods with active ones)
4. Clearable local state

The preservation of a relatively large anonymity set for all users is our central strategy for mitigating fingerprinting risk as much as possible while also ensuring a substantial improvement in the localization experience for a wide range of users. Rather than increasing the fingerprinting attack surface, this proposal could in fact help *reduce* the fingerprinting surface as a whole by providing a mechanism to get many of the benefits of the 'Accept-Language' header without that header's passive fingerprinting implications.

As noted in the [Security Considerations](https://datatracker.ietf.org/doc/html/rfc8942#section-4) section of the HTTP Client Hints RFC, a key benefit of the Client Hints architecture is that it allows for proactive content negotiation without exposing passive fingerprinting vectors, because servers must actively advertise their use of specific Client Hints headers. This makes it possible to remove preexisting passive fingerprinting vectors and replace them with relatively easily detectable active vectors. The Detectability section of [Mitigating Browser Fingerprinting in Web Specifications](https://www.w3.org/TR/fingerprinting-guidance/#detectability) describes instituting requirements for servers to advertise their use of particular data as a best practice, and mentions Client Hints as a tool for implementing this practice. In the absence of Client Hints, use of the JavaScript API can at least be detected by clients. In no case does this proposal allow for any new passive fingerprinting vectors. 

The use of the 'Sec-' prefix forbids access to headers containing 'Locale Extensions' information from JavaScript, and demarcates them as browser-controlled client hints so that they can be documented and included in requests without triggering CORS preflights. 

As in all uses of Client Hints, user agents must clear opt-in Client Hints settings when site data, browser caches, and cookies are cleared.


## FAQ


### What about clients that don't implement Client Hints?

Using Locale Extensions is still possible through the JavaScript API. Use of the API may present small drawbacks inherent to agent-side content negotiation in general — the extra request required, etc.

### Maintaining a large anonymity set for content in smaller language/region pairs?

A user viewing content in 'en-US' or 'zh-CN' is, all else being equal, going to be more anonymous than users of less-ubiquitous language/region pairs. As a result, it is significantly easier to provide a larger range of options to these users; even if we split them up into distinct smaller anonymity sets, it is still likely that they can hide in a crowd. Locale preferences for content in less commonly-used locales will be less likely to be honored, unless those preferences reflect either a globally common set of preferences (for example, '-u-fw-mon-hc-h23-mu-celsius' or '-u-fw-sun-hc-h12-mu-fahrenhe') or a common local preference (as in preferring '-u-nu-deva' instead of -u-nu-latn' in 'hi-IN'). This restriction is unfortunate, but may be unavoidable.

### Criteria for selecting available locale extension strings? 

Determining the specific locale extension strings set for each region will require user research. It is possible that in many regions the only strings available will be some subset of `-u-fw-monday-hc-h23-mu-celsius` or `-u-fw-sunday-hc-h12-mu-fahrenhe`.

### Options that aren't captured by Unicode Extensions for BCP 47

There exist other localization-related customizations that would be useful for site intelligibility - most notably, number separators and number patterns. Support for a commonly used subset of these options could be possible, particularly in cases where they strongly correlate with a particular combination of valid lcale extension strings.

### Adding and removing additional locale extension strings

A conservative approach should be taken in adding and especially in removing available locale extension strings. This is in order to avoid situations wherein (for example) users are unsure what scale a given temperature is in, or situations where a user who had previously been allowed to use their preferred numbering system no longer have access to it.
