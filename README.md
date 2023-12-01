## Motivation

On the Web platform, content localization is dependent only upon a user's language or region. However, this behavior can result in annoyance, frustration, offense, or even unintelligibility for some users. This proposal addresses common cases where users prefer locale-related tailorings that differ from the defaults in their locale. Consider the following problems:

1. A Spanish-speaking person from the United States has moved to Spain, and would prefer if local news sites would display weather forecasts using temperatures measured in Fahrenheit rather than Celsius.
2. An English-speaking user from Japan receives non-localized content in 'en'. They can read English, but nevertheless prefer seeing a 24-hour clock and calendars that have Monday, rather than Sunday, as the first day of the week. In general, 'en-US' is currently the typical untranslated language for software, even though 'en-US' has region-specific formatting patterns that differ from those used globally. As a result, often text with untranslated UI strings will be displayed in a language accessible to all users who speak English, but with temperatures and times represented in globally uncommon scales.
3. A user who has emigrated from one country to another sets their language dialect to one they can understand, but prefers that dates, times, and numbers be rendered according to local standards.
4. In some geographical regions multiple numbering systems are in use. For example, both Western Arabic (Latin) and Eastern Arabic numerals are in common use in Iran and Afghanistan. Users in these regions may find one or the other of these numbering systems not immediately intelligible, and therefore desire content tailored to the numbering system with which they are most familiar.
5. Western Arabic (Latin) numerals are used by default in 'hi', even though many users find Devanāgarī numerals more intelligible.

In the native environment these problems do not occur, since users can specify these desired customizations in their system settings. However, offering the full amount of flexibility allowed in the native environment is not possible in the often hostile web environment &mdash; if a user's preferences as specified in their OS settings are at all uncommon, servers can use them to fingerprint that user.

This proposal defines a mechanism to let clients to read user preferences from their operating system and then relay a subset of those preferences to servers, while refraining from sending combinations of preferences that are likely to individuate those users.  This allows for significantly more complete &mdash; but not necessarily perfect &mdash; localization while only exposing relatively coarse-grained data about users. 


## Overview 


Unicode Extensions for BCP 47 can be used to append additional information needed to identify locales to the end of language identifiers. Enabling support for a subset of BCP tags can help solve problems like the ones above.

The proposal includes two mechanisms for conveying OS settings to servers:

For **client-side applications**, A browser API that fetches this information from the different platform-specific APIs. 

For **server-side applications**, a set of `Client Hints` request header fields. Servers indicate the specific proactive content negotiation headers they accept in the `Accept-CH` response header.

The API and `Client Hints` infrastructure are straightforward: the API provides a method for accessing each individual preference, and the `Client Hints` headers each provide access to one individual preference. There is no provided way in either mechanism to request all preferences at once -- each preference must be explicitly requested.

For both of these mechanisms, the interface is deliberately simple. Most of the complexity of the proposal is concerned with determining what combinations of preferences can be safely exposed to servers without greatly increasing the available fingerprinting surface.

We support the following customization options:

* Preferred first day of week for calendars. This option corresponds to the `fw` Unicode Extensions for BCP 47 tag
* Preferred hour cycle (12-hour clock or 24-hour clock). This option corresponds to the `hc` Unicode Extensions for BCP 47 tag
* Preferred temperature measurement scale (Celsius or Fahrenheit). This option corresponds to the `mu` Unicode Extensions for BCP 47 tag.
* Preferred calendar (Gregorian, Islamic, Buddhist Solar, etc.). This option corresponds to the `ca` Unicode Extensions for BCP47 tag.
* Preferred numbering system (Devanāgari, Bengali, Eastern Arabic, etc.). This option corresponds to the `nu` Unicode Extensions for BCP47 tag.

We only support certain combinations of these settings, however -- only that subset of preference settings that when taken together match sufficiently commonly used combinations of preferences that transmitting those preferences is likely to leave the user with a relatively large anonymity set.

## Fingerprinting and Internationalization

Fingerprinting is the practice of uniquely identifying users by gathering together information about the user's system that when taken as a totality serve to distinguish that user from all others, even though none of the individual pieces of information could by themselves be used to de-anonymize the user. Fingerprinting allows sites to track users without their knowledge or consent, thereby meaningfully violating user privacy. Some uses of fingerprinting tactics "https://www.eff.org/deeplinks/2018/06/gdpr-and-browser-fingerprinting-how-it-changes-game-sneakiest-web-trackers">[may be illegal in the European Union](https://www.eff.org/deeplinks/2018/06/gdpr-and-browser-fingerprinting-how-it-changes-game-sneakiest-web-trackers) under the General Data Protection Regulation (GDPR). 

The [Mitigating Browser Fingerprinting in Web Specifications](https://www.w3.org/TR/fingerprinting-guidance/#detectability) WICG Interest Group Note observes that there exists no plausible way to eliminate fingerprinting. The best we can do is mitigate fingerprinting, either by reducing the available surface for fingerprinting (i.e. revealing less information) or by ensuring that whatever fingerprinting occurs is in some way observable ("active fingerprinting") instead of invisible to the user ("passive fingerprinting"). 

Fingerprinting is a particularly diabolical problem in the context of internationalization, because requesting properly tailored localized content requires exposing a significant amount of sensitive information about the user, information that is not just useful for individuating the user but also reveals details of the identity categories the user falls into.

We have two mandates that we must fulfill if we want to make Web content accessible worldwide:

1. We must provide content localized in a way such that the content is legible to all users, culturally sensitive to all users, with a presentation quality that is consistent across languages and locales.
2. We must, to the best of our abilities, ensure the privacy of all users.

This proposal involves exposing data about the OS settings of users, which necessarily exposes an attack surface for fingerprinting. However, we have made this surface as small as possible by carefully avoiding sending combinations of settings that would likely result in the user become distinguishable. All fingerprinting surfaces made available as the result of this proposal are active fingerprinting surfaces. Any attempt to gather unnecessary user data will require the server make additional requests for each individual preference setting.

_Mitigating Browser Fingerprinting_ makes clear that fingerprint mitigation requires minimizing the fingerprinting surface even when the information that could be gathered is already exposed by other parts of the Web platform. This is because, among other reasons, the Web platform is not static: if the same bits of entropy are exposed via two means, it becomes harder to remove the fingerprinting surface in the future. This spec aims to adhere to this as much as possible, with the (somewhat arbitrary) goal of exposing *less* information than one entry in the `Accept-Language` preferred language list. 

### Example Problem #1: 

Consider the case of a student from the Netherlands who is spending a year at a university in Chicago. This student could avoid annoyance (and possibly error) if the university's course catalog displayed times on a 24 hour cycle instead of using the 12 hour clock common in the United States, and for the same reason would likewise prefer if calendars were displayed with Monday, rather than Sunday, as the first day of the week. Additionally, they are not acclimated to the region's extreme winters, and use the weather display on a local news site to help determine how many layers of clothes to wear. They would very much like to avoid the frustration and potential for mishap involved in converting temperatures from the unfamiliar Fahrenheit scale to the immediately comprehensible Celsius scale. 

Native applications can directly read the OS settings for hour cycle, first day of week, and temperature measurement system. However, it is not safe to directly expose this information to potentially hostile Web servers, since if our user's settings were idiosyncratic those idiosyncratic settings could easily be used to track them. It's safe to set your OS to display temperatures in Kelvin, but dangerous to tell arbitrary Web servers about it. 

The student's preferences can be expressed with the following [locale extension string](http://www.unicode.org/reports/tr35/#Locale_Extension_Key_and_Type_Data): 

`-u-fw-mon-hc-h23-mu-celsius`

CLDR's supplemental data provides information on the default settings for these options for each region, alongside information on the population of the region, the languages spoken in the region, and the literary rate of the region. To get a *very* rough estimate -- any real estimate would require user research -- of the number of people in the world who would send that same locale extension string, I've multiplied the population of each region by the literacy rate of that region, and summed the literate populations of the regions which by default use those settings.  

| -u-fw-mon-hc-h12-mu-fahrenhe | 81212      |
| extension string             | population | # locales using |
|------------------------------|------------|-----------------|
| -u-fw-mon-hc-h23-mu-celsius  | 2,714,937,996 | 674             |
| -u-fw-sun-hc-h12-mu-celsius  | 1,665,105,458 | 277             |
| -u-fw-sun-hc-h23-mu-celsius  | 917,309,644  | 199             |
| -u-fw-sun-hc-h12-mu-fahrenhe | 332,515,201  | 26              |
| -u-fw-mon-hc-h12-mu-celsius  | 315,642,460  | 173             |
| -u-fw-sat-hc-h12-mu-celsius  | 224,538,941  | 53              |
| -u-fw-sat-hc-h23-mu-celsius  | 82,481,712   | 30              |
| -u-fw-fri-hc-h23-mu-celsius  | 385633     | 2               |
| -u-fw-sun-hc-h23-mu-fahrenhe | 307290     | 2               |
| -u-fw-mon-hc-h12-mu-fahrenhe | 81212      | 3               |
