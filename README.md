## Motivation

On the Web platform, content localization is dependent only upon a user's language or region. However, this behavior can result in annoyance, frustration, offense, or even unintelligibility for some users. This proposal addresses common cases where users prefer locale-related tailorings that differ from the locale defaults. Consider the following problems:

1. People who have moved from anywhere else in the world to the United States often prefer temperatures on weather sites be displayed in Celsius rather than Fahrenheit.
2. In some locales multiple numbering systems are in common use. Users seeking content in these locales may find one or the other of these numbering systems not immediately intelligible, and therefore need a way to request content they can read.
3. An English-speaking user from Japan receives non-localized content in 'en-US'. They can read English, but nevertheless prefer seeing a 24-hour clock and calendars that have Monday, rather than Sunday, as the first day of the week.
4. More generally: 'en-US' is currently the typical untranslated language for software, even though 'en-US' has region-specific formatting patterns that differ from those used globally. As a result, often text with untranslated UI strings will be displayed in a language accessible to all users who speak English, but with temperatures and times represented in globally uncommon scales.
5. Users who has emigrated from one country to another may want to set their language dialect to one they can understand, but prefer that dates, times, and numbers be rendered according to local standards.

In the native environment these problems are easily solved, since users can specify their preferences in their system settings. However, offering the full amount of flexibility that the native environment allows is not possible in the often hostile Web environment. If a user's preferences as specified in their OS settings are at all uncommon, revealing them can result in privacy loss. This is because servers can use these uncommon settings to fingerprint users &mdash; to distinguish them from all other users, and thereby undetectably track those users without permission.

This proposal defines a mechanism for web clients to read user preferences from their operating system and then relay a safe subset of those preferences to servers, while refraining from sending combinations of preferences that are likely to individuate those users. This allows for significantly more complete &mdash; but not necessarily perfect &mdash; localization while only exposing relatively coarse-grained data about users. 

## Overview 

Unicode Extensions for BCP 47 can be used to append additional information needed to identify locales to the end of language identifiers. Enabling limited support for a carefully selected subset of BCP tags can help solve problems like the ones above.

The proposal includes two mechanisms for conveying OS settings to servers:

For **client-side applications**, a browser API that fetches this information from the different platform-specific APIs. 

For **server-side applications**, a set of `Client Hints` request header fields. Servers indicate the specific proactive content negotiation headers they accept in the `Accept-CH` response header.

The API and `Client Hints` infrastructure are straightforward: the API provides methods for accessing each individual preference separately, and the `Client Hints` headers provide a means to request each individual preference separately. There is no provided way in either mechanism to request all preferences at once -- each preference must be explicitly requested. 

We propose supporting commonly used combinations of the following customization options. The list is given in rough order of priority, with numbering system being the highest priority customization and preferred first day of week the lowest.

1. Preferred numbering system (Devanāgari, Bengali, Eastern Arabic, etc.). This option corresponds to the `nu` Unicode Extensions for BCP 47 tag.
2. Preferred hour cycle (12-hour clock or 24-hour clock). This option corresponds to the `hc` tag
3. Preferred temperature measurement unit (Celsius or Fahrenheit). This option corresponds to the `mu` tag.
4. Preferred calendar (Gregorian, Islamic, Buddhist Solar, etc.). This option corresponds to the `ca` tag.
5. Preferred first day of week for calendars. This option corresponds to the `fw` tag

Most of the complexity of the proposal is concerned with determining what combinations of preferences can be safely exposed to servers without impacting user privacy.

In its current state, this proposal features a number of back-of-the-envelope estimations made using uncertain or incomplete data, and the frequent use of vague terms like "likely" and "unlikely". Replacing these placeholder terms with clear normative statements will require user research data, and ideally will require on-the-ground knowledge of each supported language and region. 

User research data will also be necessary to determine whether all five of these customization options can be made available, or if in some cases one or more must be dropped.

This proposal discusses the five tags above in two groups:

* `fw`, `hc`, and `mu` (first day of week, clock, temperature measurement unit). These tags makes sense to discuss in one group because the number of valid settings for each of them is naturally limited.
* `ca` and `nu` (calendar, numbering system). A large number of options are available for these tags, but in most locales any setting other than the default would immediately make a user individuatable. The `nu` option in particular presents a delicate problem: in many contexts, using it at all would result in the user becoming immediately individuatable, but in some contexts failing to provide it results in a number of users receiving unintelligible content.


## Fingerprinting and Internationalization

What makes this proposal difficult to design is the problem of *fingerprinting*.  This is the practice of uniquely identifying users by gathering together information about the user's system that when taken as a totality serve to distinguish that user from all others, even though none of the individual pieces of information could in isolation be used to de-anonymize the user. Fingerprinting allows sites to track users without their knowledge or consent, thereby meaningfully violating user privacy.

Data gathered by the Electronic Frontier Foundation (EFF) for Peter Eckersley's 2010 paper [How Unique is your Browser?](https://coveryourtracks.eff.org/static/browser-uniqueness.pdf) estimated that 83.6% of browsers visiting their site bore a unique fingerprint. Some improvements have been made in the intervening years. Notably, the end of Adobe Flash and Java applets as Web technologies has foreclosed a number of potential fingerprinting attacks, and substantial measures have been taken to mitigate the risk of font-based fingerprinting.  Nevertheless, browsers today are nearly as fingerprintable as browsers in 2010 were. The process of reducing the potential for fingerprinting on the Web platform is necessarily a long and slow one  -- a process measured in decades rather than years -- and involves the gradual replacement of technologies that expose more user information with technologies that expose less.

Fingerprinting is a particularly diabolical problem in the context of internationalization. The most straightforward way to prevent fingerprinting is to make it impossible to send rare combinations of settings. However, equitable internationalization requires providing access to content in less commonly used locales, with this content appropriately tailored for all communities of users regardless of the size of that community -- accommodating rare requests as well as common noes.  Moreover, requesting properly tailored localized content necessarily requires exposing a significant amount of particularly sensitive information about the user, information that is not just useful for individuating the user but also can reveal details of the identity categories the user falls into.

The [Mitigating Browser Fingerprinting in Web Specifications](https://www.w3.org/TR/fingerprinting-guidance/#detectability) WICG Interest Group Note, the best practices from which were used as a primary framework in the design of this proposal, observes that there exists no plausible way to eliminate fingerprinting. We can at most mitigate fingerprinting, either by reducing the available surface for fingerprinting, i.e. revealing less information, or by ensuring that whatever fingerprinting occurs is in some way observable ("active fingerprinting") instead of invisible to the user ("passive fingerprinting").  Some fingerprinting tactics [are only questionably legal in the European Union](https://www.eff.org/deeplinks/2018/06/gdpr-and-browser-fingerprinting-how-it-changes-game-sneakiest-web-trackers) under the General Data Protection Regulation (GDPR). By ensuring that the only fingerprinting opportunities made available require action taken by the server, it becomes more possible to control fingerprinting through regulatory means. 

This proposal involves exposing data about the OS settings of users, which necessarily exposes an attack surface for fingerprinting. However, we have made this surface as small as possible by limiting the number of individual preferences that can be revealed, by limiting the combinations that can be sent, and by ensuring that all attack surfaces made available as the result of this proposal are active fingerprinting surfaces.  

_Mitigating Browser Fingerprinting_ makes clear that fingerprint mitigation requires minimizing the fingerprinting surface even when the information that could be gathered is already exposed by other parts of the Web platform. This is because, among other reasons, the Web platform is not static: if the same bits of entropy are exposed via two means, it becomes harder to remove the fingerprinting surface in the future as it will require removing it from two different places at once. This spec aims to adhere to this guideline as closely as possible, with the goal of exposing significantly information than one locale listed in the `Accept-Language` header's language list.  This goal is somewhat arbitrary -- but not entirely arbitrary. Although the `Accept-Language` header provides the ability to request a preferred locale and several preferred fallback locales, it is generally known that users should never change the content of this header away from browser defaults. See, for example, [the MDN page on `Accept-Language`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language), which notes that "Users rarely change [this header] and such changes are not recommended because they may lead to fingerprinting." Google is currently trialling the reduction of the number of languages in `Accept-Language` to just one, due to the large attack surface exposed when multiple languages are listed. 

A user who will accept content tailored to either of two or more different locales &mdash; say they want 'fr-FR' but accept 'fr-CA' as a fallback  &mdash; might be tempted to list both of these in `Accept-Language`. Even though this combination of preferences may be somewhat common, the amount of information revealed through explicitly requesting it could easily make the user immediately identifiable. This proposal provides a safe mechanism of achieving a similar goal, while ensuring that the user remains indistinguishable from a large group of other users.

## Supported tags group 1: Preferred first day of week, preferred clock, preferred temperature measuring unit.

The Unicode Locale Extension tags `fw`, `hc`, and `mu` can be used to request a preferred first day of week, hour cycle, and temperature measurement unit. These three tags are useful to consider together. This is because there is a limited set of commonly used options for each of these tags: 

* No locale has as its default hour cycle anything but `h12` (hours from 1 to 12) or `h23` (hours from 0 to 23). 
* No locale defaults to anything but `celsius` or `fahrenhe` for `mu`. 
* No locale defaults to anything but `mon`, `fri`, `sat`, or `sun` as its first day, with only one region (the Maldives) defaulting to `fri`. 

As such, for most users their combination of preferred settings for these three options will be shared with hundreds of millions or even billions of other people.


### Commonly used locale defaults

CLDR's supplemental data provides information on the default settings for these options for each region, alongside information on the population of the region, the languages spoken in the region, and the literary rate of the region. To get a *very* rough estimate &mdash; any real estimate would require user research &mdash; of the number of people in the world who would have the same settings for 'fw`, `hc`, and `mu`, we've multiplied the population of each region by the literacy rate of that region, and summed the literate populations of the regions which by default use those settings.  

| extension string             | population | # locales using |
|------------------------------|------------|-----------------|
| -u-fw-mon-hc-h23-mu-celsius  | 2,714,937,996 | 674             |
| -u-fw-sun-hc-h12-mu-celsius  | 1,665,105,458 | 277             |
| -u-fw-sun-hc-h23-mu-celsius  | 917,309,644  | 199             |
| -u-fw-sun-hc-h12-mu-fahrenhe | 332,515,201  | 26              |
| -u-fw-mon-hc-h12-mu-celsius  | 315,642,460  | 173             |
| -u-fw-sat-hc-h12-mu-celsius  | 224,538,941  | 53              |
| -u-fw-sat-hc-h23-mu-celsius  | 82,481,712   | 30              |
| -u-fw-fri-hc-h23-mu-celsius  | 385,633     | 2               |
| -u-fw-sun-hc-h23-mu-fahrenhe | 307,290     | 2               |
| -u-fw-mon-hc-h12-mu-fahrenhe | 81,212      | 3               |

The three strings that rarely appear in region defaults reflect, in order, the default preferences in the Maldives, the default preferences in Belize, and the default preferences in both the Cayman Islands and Palau. All of the regions other than the United States that are listed as using a string indicating a preference for the Fahrenheit scale only use Fahrenheit for referring to weather.

### Example: People who have traveled from one region to another

Consider the case of a student from the Netherlands who is spending a year at a university in Chicago. This student could avoid annoyance (and possibly error) if the university's course catalog displayed times on a 24 hour cycle instead of using the 12 hour clock common in the United States, and for the same reason would likewise prefer if calendars were displayed with Monday, rather than Sunday, as the first day of the week. Additionally, they are not acclimated to the region's extreme winters, and sometimes use the weather display on a local news site to help determine how many layers of clothes to wear. They would very much like to avoid the frustration and potential for mishap involved in converting temperatures from the unfamiliar Fahrenheit scale to the immediately comprehensible Celsius scale. 

Native applications can directly read the OS settings for preferred clock, first day of week, and temperature measurement system. However, it is not safe to directly expose this information to potentially hostile Web servers, since if our user's settings were idiosyncratic those idiosyncratic settings could easily be used to track them. It's safe to set your OS to display temperatures in Kelvin, but dangerous to tell arbitrary Web servers about it. 

The student's preferences can be expressed by the locale extension string `-u-fw-mon-hc-h23-mu-celsius`. Consulting the table above, we see that the preferences of our hypothetical student are shared with billions of other people. Revealing this set of preferences would not by itself dramatically reduce the size of the student's anonymity set.

#### Preferences in Combination with Browser Localization

Fingerprinting servers necessarily know other pieces of locale-related information that they can use in combination with the preferences string to further reduce the size of a user's anonymity set. Most obviously, servers know the contents of the `Accept-Language` header, and can also get equivalent information by consulting `navigator.languages`. For the sake of simplicity, we will refer to the language or languages made available via these two routes as the "browser locale."

The student in the example above's browser locale will almost certainly reflect either where they're from (in this case, 'nl') or where they're visiting (in this case, 'en-US'). 

1. If the student's browser locale is 'nl', the server can already determine that they most likely prefer `-u-fw-mon-hc-h23-mu-celsius` by inspecting the `Accept-Language` header. Although the guidelines in _Mitigating Browser Fingerprinting_ are clear that fingerprinting mitigation is still necessary in situations where the revealed data is already revealed elsewhere, in this case we are reasonably safe to expose this information: it is not plausible that the Web platform will ever remove the ability for users to indicate their most preferred language.

2. If the student's browser locale is 'en-US', revealing this preference string will likewise leave them with a relatively large anonymity set &mdash; most people from the regions that use those preferences by default will have left their defaults unchanged, and it is to be expected that there will be a large number of people in the same situation as our student: requesting 'en-US' with OS settings matching the defaults for the hundreds of locales that default to `-u-fw-mon-hc-h23-mu-celsius`.

In this scenario, wherein the user requests a combination of settings that differ from the defaults in the current locale but are the same as the defaults in other locales, revealing the user's settings does not dramatically reduce the size of their anonymity set. 

### Example: People with preferences that differ from *all* commonly used locale defaults

Every commonly used combination of `fw` and `hc` can be used in combination with the value `celsius` for `mu`. However, since the only sizable locale that defaults to `fahrenhe` is 'en-US' combinations of preferences involving `fahrenhe` that otherwise differ from the 'en-US' defaults are not guaranteed to be commonly used. Nevertheless, there are a number of people likely to use a browser locale of 'en-US' with preferences that differ from the United States defaults:

* People working with organizations that use a 24 hour clock.
* People with social, familial, and cultural ties to regions that use Saturday as the first day of the workweek.
* People who for reasons of simple personal preference like their calendar to have the first day of the workweek at the left-hand side of their calendars.
* People not from or in the United States who are nevertheless using 'en-US' as their browser locale.

Because of the large number of people using 'en-US', it will likely be safe to offer most of these combinations of preferences despite them not being commonly used locale defaults. User research will be required. If allowing all of these settings is not possible without making users of those settings easily individuatable, it may be possible to respect most preferences of most users while retaining larger anonymity sets by dropping support for the `fw` tag. See the table below for the number of people in locales that default to each of the possible strings with `fw` removed:

| extension string      | population | # locales using |
|-----------------------|------------|-----------------|
| -u-hc-h23-mu-celsius  | 3715114985 | 905             |
| -u-hc-h12-mu-celsius  | 2205286859 | 503             |
| -u-hc-h12-mu-fahrenhe | 332596413  | 29              |
| -u-hc-h23-mu-fahrenhe | 307290     | 2               |


## Supported tags group #2: Numbering system and calendar 

The three tags discussed above naturally provide a limited number of options. There are only seven combinations of `fw`, `hc,` and `mu` that are commonly used as region defaults: the six combinations of options with `mu-celsius`, plus the region defaults for the United States. The tags `ca` and `nu` are different, because there are potentially a large number of settings for each of these tags, with most of those settings making the user immediately individuatable. 

### Preferred numbering system

CLDR supports nearly 100 different numbering systems. However, almost all of these options only make sense in a few locales. 

We propose to restrict the number of available options for the `nu` tag to the following options:

* The default for the browser locale
* `latn` (Western Arabic), if the default is not `latn`.
* The numbering system designated as `native`, but only if that numbering system is actually in common use in that locale.

In certain regions, most notably the regions in which both Western Arabic and Eastern Arabic numerals are in common use, failing to support both of those numbering systems results in the delivery of content that is unintelligible to some users. Not being able to express a desire for a particular numbering system causes precisely the same problems that would be caused if users in locales where multiple scripts for text weren't able to select a script that's legible to them. This means that supporting the `nu` tag is a top priority, even if supporting that tag means not supporting any of the other locale preferences tags. 

See the section _Avoiding Prioritizing Larger Linguistic Communities_ for strategies on ensuring that the `nu` tag be made safely available.

### Preferred calendar

Nearly all regions use the Gregorian calendar, which is expressed as a locale extension tag as `ca-gregory`. It is likely safe in all regions to explicitly request `ca-gregory`, since texts from regions that use non-Gregorian calendars typically present dates from the Gregorian calendar alongside dates in their local calendar. 

It is safe for users to send requests for non-Gregorian calendars when the specific calendar requested is either the default for the user's locale or a commonly used alternate in that locale. However, outside of these cases requesting a non-Gregorian calendar would likely serve to immediately individuate the user. 

User research will be required to determine which alternate calendar preferences are safe to express in each locale. 

## Avoiding prioritization of larger linguistic communities.

Preserving the size of the user's anonymity set necessarily requires ensuring that they can't send rarely used combinations of preferences. However, internationalization requires respecting the needs of users even (or especially) when they're from a relatively small cultural or linguistic community. An approach that treats the combination of the user's browser localization and user preferences as the factors determining which preferences are safe to send necessarily gives more options to people from very large linguistic and cultural communities. This runs directly against the spirit of internationalization. 

If the only sources of entropy we consider are the browser locale and the set of preferences sent, then we are forced to either offer very few options to all users, or else offer fewer options to users requesting anything but the most commonly used locales. Given the choice between these two options, the first is preferable. There may, however, be strategies that allow us to offer a robust range of preferences to all users, regardless of the size of their linguistic community. 

### Considering content locale 

One distinctive feature of locale-related user information is that the individual pieces of information are often very strongly correlated. A user who prefers temperatures in Fahrenheit is very likely to prefer the other defaults for the `en-US` locale, a user with a browser locale in Europe is very likely to prefer Monday as the first day of the week, and so forth. 

In addition to the information provided in the user's `Accept-Language` header, the server also necessarily knows one other piece of locale-related information: the locale of the content it is serving. This correlates strongly with the browser locale &mdash; if someone is receiving content in 'nl', it's very likely that their `Accept-Language` contains 'nl'. If a user's browser locale matches the content's locale, it may be reasonable to allow access to more preference options. 

One potential site for further user research is determining what other combinations of browser locale and non-default user preferences are common enough to allow when viewing specific content locales. This situation could arise when, for example, the region given in the browser locale has close cultural ties to the region in the content's locale, perhaps because of a shared language, because of economic ties, or because there exist diasporic communities from one of the regions in the other.

One site for user research is determining whether significant numbers of people from other locales ultimately receive content from one of a handful of very commonly used locales. If so, users from those locales may safely transmit user preferences information while viewing content in those commonly used locales. 

There are a number of difficulties involved in taking content locale into consideration. When using the `Client Hints` architecture, steps would have to be taken to ensure that the content's locale is known to the browser before the browser can safely reveal preference information. It may be easier to find and use a rough proxy for content locale &mdash; for example, the TLD of the server.

### Options reduction

Reducing the number of options available across locales is a possible way to ensure that users of minority languages can express some preferences without necessarily exposing themselves to additional privacy loss. The removal of `fw` from the proposal, as previously mentioned, is one possibility. An additional possibility is limiting use of the `mu` tag &mdash; for example, allowing users to request the common `mu-celsius` setting, but not the rare `mu-fahrenhe` one. This would reduce the total entropy exposed, while still honoring the preferences of a wide range of users.
