# Energy Saver Mode ⚡🔋

## Authors:

- Anssi Kostiainen

## Participate
- GitHub issue https://github.com/w3c/battery/issues/9

## Introduction

>**Energy conservation** is the effort to reduce wasteful energy consumption by using fewer energy services. This can be done by **using energy more effectively** or **changing one's behavior** to use less service. ([Wikipedia](https://en.wikipedia.org/wiki/Energy_conservation))

This document discusses an energy saver mode feature that is an integral part of modern operating systems. This feature provides a hint to native applications that they should use energy more effectively or change their behavior to avoid doing something or postpone non-critical tasks. This information would be equally beneficial to web apps to enable more energy-efficient web experiences.

## Goals

The end user expects modern web applications to use less energy when the OS-level energy saver mode setting is turned on. Web authors have currently no means to apply such optimization on demand informed by this signal because it does not exist.

Below we expand on use cases that may make use of context-sensitive [energy saving strategies](#energy-saving-strategies-for-web-applications) with the help of a [proposed new API](#proposed-new-api) that exposes the OS-level energy saver mode state and a corresponding signal.

<details>
<summary>Use cases for Energy Saver Mode</summary>

- Web app disables CSS animations and stops video playback when the energy saver mode is turned on.

- Web app adjusts the amount/number of data/messages it fetches over the network as well as the polling frequency depending on the energy saving mode state.

- If the user has put the device in energy saving mode, the webmail client will only poll for new data if the user explicitly interacts with the app and clicks an "Update" button. If the energy saving mode is turned off, the app fetches data without user interaction.

- Web-based navigation app will only start an energy-intensive operation that must run to completion if the device is not in energy saving mode. For example, when starting a long-running navigation, the user may be recommended to turn on the energy saver mode (if the battery is also low).

- The mapping app will show a simpler 2D map if the energy saving mode is on, and only turn the WebGL-powered 3D map on by default if the energy saver mode is off.

- Web app will only start a long-running energy-intensive operation if the energy saving mode is turned off. For example, download high-resolution textures in a web-based game, or enable extra video feeds in a web-based video conferencing app.

- The user manually turns on the OS energy saving mode even if the battery level is 95% to ensure the device will survive a long-haul flight. While on the plane, the user browses the web and uses his favourite web apps using the airliner's Wi-Fi service. The web-based apps in use automatically enable optimizations to prolong the battery life: service worker caching approach is switched to "cache as needed", features using WebGL are disabled, and automatic polling for new data over network is stopped.

- The user is laying in her bed before going to sleep. The phone's battery is below 15% and the energy saver mode is automatically turned on by the OS. Knowing the phone will soon be able to recharge over night, the user will turn off the energy saver mode and the web-based email client starts to fetch new emails automatically while she is working to reach inbox zero. (The OS also returns the screen brightness to its normal setting.)

- A web developer conducts an A/B test of a new version of a web application to ensure all of its diverse userbase still receives a good experience. The performance metric data retrieved via [Performance Timeline](https://www.w3.org/TR/performance-timeline-2/) is adjusted to consider the energy saver mode state.

</details>

## Non-goals

- Allow the script to toggle the power saving mode.

- Expose the battery level. This requirement is already addressed by the existing [Battery Status API](https://www.w3.org/TR/battery-status/). The proposed API will work together with the Battery Status API, but also addresses many use cases on its own.

## User research

A [research paper](https://www.researchgate.net/publication/341123776_Investigating_the_Correlation_between_Performance_Scores_and_Energy_Consumption_of_Mobile_Web_Apps) discovered a statistically significant negative correlation between performance scores and energy consumption of mobile web apps. The results from a controlled experiment involving 21 real third-party web app implies that an increase in the performance score tends to lead to a decrease in energy consumption.

Many tools exists for optimizing the initial page load performance. For example, Google’s [Lighthouse](https://developer.chrome.com/docs/lighthouse/), [PageSpeed Insights](https://pagespeed.web.dev/) and guidance from [Core Web Vitals](https://web.dev/vitals/) provide insights for improving the user experience of the initial load.

This demonstrates that performance optimization of web apps is a well-understood craft with many tools and APIs available to assist web authors. However, explicit energy efficiency optimizations have been left as an implementation detail of the browser, at most exposed through the _browser settings_ to users, see e.g. [Chrome Energy Saver](https://blog.google/products/chrome/new-chrome-features-to-save-battery-and-make-browsing-smoother/) and [Edge Efficiency Mode](https://techcommunity.microsoft.com/t5/articles/efficiency-mode-in-microsoft-edge-save-even-more-battery-life/m-p/3651853). Such optimizations in the browser implementation, however, while useful, cannot take into account the context-sensitive requirements of web apps. This limits the optimization opportunity space in browser implementation to operations that are not time or context sensitive. For example, the browser can batch or delay push message delivery because it is [specified](https://www.w3.org/TR/push-api/) they can be sent at any time. On the other hand, the browser cannot know whether it can disable [transparency in canvas](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas#turn_off_transparency) since it can be crucial to the user experience.

Because energy-saving measures often have a user visible impact, anecdotal evidence suggests web authors tend to optimize for a better user experience over energy efficiency. Web authors are also lacking tools to measure the energy use and APIs to understand when energy-efficient operation is acceptable, or desired. This explainer discusses the latter problem and explores possible solutions to it.

All major desktop and mobile OSes have integrated energy saver mode features that are under the user's control. Arguably users are already trained to understand the impact on the user experience of such a feature: users' expectations are lowered and more modest UX is accepted when such a mode is enabled.

<details>
<summary>
Other interesting tidbits
</summary>

- Some estimates put the global carbon emissions of the internet’s 5 billion worldwide users on par with that of the airline industry. ([Ecograder](https://ecograder.com/how-it-works))

- The internet consumes a lot of electricity. 416.2 TWh per year to be precise. To give you some perspective, that's more than the entire United Kingdom. ([Website Carbon Calculator](https://www.websitecarbon.com/))

- The web must be an environmentally sustainable platform. ([W3C TAG Ethical Web Principles](https://www.w3.org/TR/ethical-web-principles/#sustainable))

- The web design industry has taken early steps to define [strategies](https://sustainablewebdesign.org/strategies/) for delivering sustainable web design projects and developed tools such as Ecograder that crawl websites to give an emissions estimate.

</details>


## Detailed design discussion

### Energy consumption timeline

The tools and best practices discussed in [user research](#user-research) are focused on optimizations _during the initial load_. Lighthouse, for example, measures performance metrics such as _First Contentful Paint_, _Time to interactive_ and _First CPU Idle_. These are events that happen early in the lifetime of the webpage, before the [current document readiness](https://html.spec.whatwg.org/multipage/dom.html#current-document-readiness) is "complete". As discussed, these optimizations will, by proxy, often lead to lower energy use too.

However, the overall energy consumption after the initial load (when the current document readiness is "complete") is more significant (TODO: add data) for a long-running web experience. Such experiences are often implemented following an architectural patterns called [single-page application](https://en.wikipedia.org/wiki/Single-page_application) (SPA) popularized by major [JavaScript frameworks](https://en.wikipedia.org/wiki/Single-page_application#JavaScript_frameworks). SPA is also a [preferred architectural pattern](https://web.dev/learn/pwa/architecture/) for progressive web applications (PWA) with mostly atomic in-page updates.

This unearths an impactful opportunity to change these JS frameworks' behavior to be more energy efficient: SPAs in particular benefit from runtime energy efficiency optimization that lower the overall energy consumption _during the entire lifetime_ of the web application and not just during the initial load.

### Energy saving strategies for web applications

A number of energy saving strategies are implementation details of the *web app* and context sensitive. That is, the browser implementation cannot apply these optimizations without breaking the UX. This implies the energy saver mode state information must be exposed to web apps to enable these highly application-specific and context-sensitive energy optimizations.

<details>
<summary>
Examples of context-sensitive energy optimizations
</summary>

- stop or limit [animations](https://developer.mozilla.org/en-US/docs/Web/Performance/Animation_performance_and_frame_rate) via SVG, JavaScript, canvas and WebGL, CSS animation, video, animated gifs and PNGs
- stop or limit script-driven background activity
- do not automatically poll (or push) for new data over the network, fetch data only when requested by the user (e.g. hit "get new messages" in an email client)
- increase the polling frequency of new data to a reasonable level (e.g. update stock ticker data hourly instead of every minute)
- adjust the amount or number of messages or data in a single fetch
- avoid eager pre-loading and fetch only minimal assets required upon load
- use lower quality assets, images, and videos, or use placeholder images in place of non-cricial ones
- throttle or disable secondary content e.g. optional third-party embeds
- selectively employ [optimizations for canvas](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas)
- reduce or do not use canvas, WebGL, WebGPU or other energy-intensive web capabilities
- prefer "cache as needed" over "precaching" as a [service worker caching approach](https://web.dev/learn/pwa/assets-and-data/#frequently-used-cache-approaches)
- use a darker theme
- stop rendering (non-critical) audio
- release the screen lock to allow the OS to turn the screen off sooner
  - (TODO: interaction with the [Screen Wake Lock API](https://www.w3.org/TR/screen-wake-lock/))
</details>



## Operating system APIs and the native-land

All major OSes provide user controls to toggle "Energy Saving Mode" state or equivalent: [Windows](https://support.microsoft.com/en-us/windows/battery-saving-tips-for-windows-a850d64d-ee8e-c8d2-6c75-8ffe6ea3ea99), [Chrome(OS)](https://blog.google/products/chrome/new-chrome-features-to-save-battery-and-make-browsing-smoother/#ug3p3), [macOS](https://support.apple.com/en-gb/guide/mac-help/mh35848/mac#mchlp7877a91), [Android (various)](https://support.google.com/android/answer/7664692#zippy=%2Cturn-on-battery-saver-or-low-power-mode) and [iOS](https://support.apple.com/en-us/HT205234).

This state information is also exposed to native app developers through native APIs, for example:
- Android [isPowerSaveMode](https://developer.android.com/reference/android/os/PowerManager#isPowerSaveMode()) boolean
- Windows [EffectivePowerMode*](https://learn.microsoft.com/en-us/windows/win32/api/powersetting/ne-powersetting-effective_power_mode) enum values
- macOS and iOS [lowPowerModeEnabled](https://developer.apple.com/documentation/foundation/nsprocessinfo/1617047-lowpowermodeenabled) boolean

These native APIs are arguably exposed to native apps, since the OS cannot act on behalf of native apps to apply context-sensitive optimizations. Similarly, the browser cannot act on behalf of web applications to enable similar optimization. The browser can however, similarly to the OS, enable generic optimizations such as suspend inactive background tabs or reduce the framerate of [animation frame](https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering) generation. These browser-level optimizations are complementary to the application-level optimizations.

## Considered alternatives

### Proposed new API

The closest existing interface to extend would be the `BatteryManager` to allow these two APIs to interact in advanced use cases. A simple event (`energysavingmodechange`) and boolean (`battery.energySavingMode`) pair may be sufficient to enable the majority of the use cases.

```
let b = await navigator.getBattery();

if (b.energySaverMode) {
    saveEnergy();
}

b.energysavermodechange = () => {
    if (this.energySaverMode) saveEnergy();
}
```

This API could hang off of its own interface if future extensibility is expected. Or if we expect implementers to support only this API and not the rest of the `BatteryManager`, or if we want to permission-gate the two APIs separately.

### New metadata element attribute "battery-savings"

If a site wanted to allow reduced framerate if it saved battery, they would add this to their page:

```
<meta name="battery-savings" content="allow-reduced-framerate">
```
If a site wanted to allow generic slowdown of script execution:

```
<meta name="battery-savings" content="allow-reduced-script-speed">
```

Source: [Explainer: web page settings to save battery](https://github.com/chrishtr/battery-savings/blob/master/explainer.md)

### New HTTP header "Save-Energy" network client hint

A new HTTP header similar to [`Save-Data: on`](https://wicg.github.io/savedata/#save-data-request-header-field):

```
GET /document HTTP/2
Host: www.example.com
Accept-Encoding: *
Save-Energy: On
```

Source: [Power-saving detection / coarser low-power insights](https://github.com/WebKit/explainers/issues/47)

### New CSS media query "prefers-energy-saving"

A new CSS media feature "prefers-energy-saving" similar to ["prefers-reduced-data"](https://www.w3.org/TR/mediaqueries-5/#prefers-reduced-data).

The prefers-energy-saving media feature could be used to detect if the user has a preference for being served energy saving content that uses less energy for the page to be rendered.

```
.image {
  background-image: url("images/heavy.jpg");
}

@media (prefers-energy-saving: reduce) {
  .image {
    background-image: url("images/light.jpg");
  }
}
```

Source: https://github.com/w3c/battery/issues/9#issuecomment-657527653


## Stakeholder Feedback / Opposition

- WebKit: https://github.com/WebKit/explainers/issues/47

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Peter Beverloo
- J. S. Choi
- Gilles Dubuc
- Robert Linder
- Sangwhan Moon
- Luc Potage
- Noam Rosenthal
- Joseph Scott
- Jimmy Wärting
