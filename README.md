# Document-Isolation-Policy explainer

This is the repository for Document-Isolation-Policy. You're welcome to
[contribute](CONTRIBUTING.md)!

To fill this out, please see: https://github.com/w3ctag/w3ctag.github.io/blob/master/explainers.md
## Authors:

- [Camille Lamy](https://github.com/camillelamy)

## Participate
- [Issue tracker]
- [Discussion forum]

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use cases](#use-cases)
  - [App with cross-origin popup](#app-with-cross-origin-popup)
  - [App with 3rd party iframe](#app-with-3rd-party-iframe)
  - [Embedded widget](#embedded-widget)
- [Background: Process wide XS-Leaks and the crossOriginIsolated model](#background-process-wide-xs-leaks-and-the-crossoriginisolated-model)
  - [Same-origin policy](#same-origin-policy)
  - [Process wide XS-Leaks](#process-wide-xs-leaks)
  - [The cross-origin isolation model](#the-cross-origin-isolation-model)
  - [Cross origin isolation limits](#cross-origin-isolation-limits)
    - [Deployability](#deployability)
    - [Iframes getting access to COI-gated APIs](#iframes-getting-access-to-coi-gated-apis)
  - [Out-of-process iframes evolving landscape](#out-of-process-iframes-evolving-landscape)
- [Document-Isolation-Policy](#document-isolation-policy)
  - [Document-Isolation-Policy header](#document-isolation-policy-header)
  - [Interaction with COEP](#interaction-with-coep)
  - [Interaction with COOP](#interaction-with-coop)
  - [Interaction with Permission Policy](#interaction-with-permission-policy)
  - [Interaction with the Origin-Agent-Cluster header](#interaction-with-the-origin-agent-cluster-header)
  - [Impact on agent clustering](#impact-on-agent-clustering)
    - [A note on SharedArrayBuffers and agent clusters](#a-note-on-sharedarraybuffers-and-agent-clusters)
  - [Interaction with Fetch](#interaction-with-fetch)
  - [Inheritance](#inheritance)
  - [Interactions with workers](#interactions-with-workers)
  - [Interaction with Isolated Web Apps](#interaction-with-isolated-web-apps)
  - [Reporting](#reporting)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [App with cross-origin popup](#app-with-cross-origin-popup-1)
    - [App with 3rd party iframe](#app-with-3rd-party-iframe-1)
    - [Embedded widget](#embedded-widget-1)
- [Considered alternatives](#considered-alternatives)
  - [COOP restrict-properties](#coop-restrict-properties)
  - [Origin-Agent-Cluster](#origin-agent-cluster)
  - [Keeping the status quo](#keeping-the-status-quo)
  - [Provide access to COI-gated APIs when full SiteIsolation is available](#provide-access-to-coi-gated-apis-when-full-siteisolation-is-available)
  - [Browsing context group switch instead of agent cluster keying](#browsing-context-group-switch-instead-of-agent-cluster-keying)
  - [Relying on COEP for the subresources checks](#relying-on-coep-for-the-subresources-checks)
  - [Restricting the ability to use Document-Isolation-Policy for cross-origin iframes](#restricting-the-ability-to-use-document-isolation-policy-for-cross-origin-iframes)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Developers want to build applications that are fast using [SharedArrayBuffers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer) (SAB), which can improve computation time by ~40%. But SharedArrayBuffers allow to create high-precision timers that can be exploited in a [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) attack, allowing to leak cross-origin user data. To mitigate the risk, SharedArrayBuffers are gated behind [crossOriginIsolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) (COI). CrossOriginIsolation requires to deploy both [Cross-Origin-Opener-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy) (COOP) and [Cross-Origin-Embedder-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy) (COEP). Both have proven hard to deploy, COOP because it prevents communication with cross-origin popups, and COEP because it imposes restrictions on third-party embeds. Finally, the whole COOP + COEP model is focused on providing access to SharedArrayBuffers to the top-level frame. Cross-origin embeds can only use SABs if their embedder deploys crossOriginIsolation and delegates the permission to use COI-gated APIs, making the availability of SABs in third-party iframes very unreliable.

The API proposed in this document, Document-Isolation-Policy, is proposing to solve these deployment concerns by relying on the browser [Out-of-Process-Iframe](https://www.chromium.org/developers/design-documents/oop-iframes/) capability. It will provide a way to securely build fast applications using SharedArrayBuffers while maintaining communication with cross-origin popups (needed for OAuth and payment flows) and not requiring extra work to embed cross-origin iframes. Finally, it will be available for embedded widgets as well as top-level frames, allowing to build efficient compute heavy widgets that are embedded across a variety of websites (e.g. photo library, video conference iframe, etc…).

## Goals

We want to provide end users with web applications that can perform compute heavy tasks efficiently (e.g. video games, videoconferencing, photo editing) and make use of the composability of the web (to allow for more convenient OAuth/payment flows, to embed cross-origin widgets such as a social media sharing widget,...). And we want this to happen without introducing security flaws in the web platform that would allow malicious applications to gain access to cross-origin user data.

## Non-goals

While the original [crossOriginIsolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) proposal was agnostic to the platform [SiteIsolation](https://www.chromium.org/Home/chromium-security/site-isolation/) capability, this proposal will rely on a browser's ability to isolate iframes in a different process from their embedder.

We're also only considering the case of embedded widgets that have their own document. We do not plan to give access to COI-gated APIs to third-party scripts if their execution context does not deploy COI.

## Use cases

### App with cross-origin popup

The user would like some of their compute heavy web apps to be faster (e.g. games, spreadsheet, photo editing, videoconferencing). To make these apps faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. Currently, a web app cannot use shared memory and cross-origin popups at the same time due to the restriction imposed by crossOriginIsolation. This means the web app cannot use 3rd party OAuth or payment flows.

### App with 3rd party iframe

The user would like some of their compute heavy web apps to be faster (e.g. games, spreadsheet, photo editing, videoconferencing). To make these apps faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. Currently, a web app using shared memory is limited in the 3rd party iframes that it can embed. For example, it cannot embed personalized ads unless the ads do a lot of work to support COEP themselves. Personalized ads are the revenue model for a lot of free web apps, so that means those free web apps cannot use shared memory unless the whole ad ecosystem deploys COEP.

### Embedded widget

The user would like some of the embedded widgets on the page they browse to be faster (e.g. photo library widget, map widget, video chat widget, ...). To make these widgets faster, relying on multithreading and shared memory is needed, but this means having access to SharedArrayBuffers. But access to SharedArrayBuffers is only possible if the embedder of the widget deploys crossOriginIsolation (with the constraints described in the previous use cases). So a widget embedded in a wide variety of websites cannot use shared memory to improve its performance.

## Background: Process wide XS-Leaks and the crossOriginIsolated model

### Same-origin policy

The same-origin policy is a critical security mechanism in the web that helps to isolate different websites and prevent them from accessing each other's data. It works by restricting the interaction between web pages based on their origin, which is determined by the combination of the scheme (e.g., HTTP or HTTPS), the host name, and the port number.

Under the same-origin policy, web pages can only access data from other pages that have the same origin. This helps protect user data and prevents malicious websites from accessing sensitive information, such as passwords or credit card numbers.

There are a few exceptions to the same-origin policy, such as cross-origin resource sharing (CORS). However, these exceptions are carefully designed to preserve the security of user data.

The same-origin policy is an essential part of web security, and it helps to protect users from a variety of attacks. By preventing websites from accessing each other's data, the same-origin policy helps to keep user data safe and secure, and allows the web to remain composable.

### Process wide XS-Leaks
Unfortunately, several web APIs allow an attacker to bypass the same-origin policy and to leak information from authenticated cross-origin resources. These kinds of vulnerabilities are known as XS-Leaks. In particular, we will focus on process-wide XS-Leaks, which can affect any resource loaded in the same renderer process as an attacker.

Process-wide XS-Leaks can be direct. For example, [performance.measureUserAgentSpecificMemory](https://developer.mozilla.org/en-US/docs/Web/API/Performance/measureUserAgentSpecificMemory) allows to easily leak a resource size. Leaking a resource size can be exploited in various cross-site search attacks that target resources that vary based on query parameters.

Process-wide XS-Leaks can also take the form of a [Spectre](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) attack. Spectre is a security vulnerability affecting modern processors, enabling attackers to read arbitrary data from the process they are executing in. It exploits speculative execution, where processors execute instructions before receiving all necessary data, to access and steal sensitive information. This includes authenticated cross-origin resources loaded in the process. Spectre attacks’ efficiency is correlated to timer resolution. More precise timers make Spectre attacks faster. This includes direct timers like [performance.now](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now), but also timers constructed through SharedArrayBuffers.

In the absence of [Site Isolation](https://www.chromium.org/Home/chromium-security/site-isolation/), an attacker can load a wide variety of authenticated cross-origin resources in their process.

*Without Site Isolation, all resources (documents + subresources) for a Browsing Context Group are rendered in the same process.*

![Iframes, subresources and popups are placed in the same process without SiteIsolation](/images/background1.png)

All of the cross-origin resources loaded into a process are vulnerable, unless they load through [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)[^1]. This is because cross-origin CORS requests are sent without credentials by default. In order to make credentialled requests, the cross-origin endpoint has to opt into the request being credentialled and the content shared with the cross-origin requestor.

[^1]: Note that the CORS caveat only applies to subresources directly loaded by the attacker origin. All other subresources are still at risk.

*Without Site Isolation, all non-CORS subresources and all documents are at risk from an attacker.*

![Without SiteIsolation, all resources are at risk from an attacker](/images/background2.png)

### The cross-origin isolation model
The [crossOriginIsolation](https://developer.mozilla.org/en-US/docs/Web/API/crossOriginIsolated) model was developed as a way to address the process-wide XS-Leak threat without requiring Site Isolation and CORS for all subresources.

First, cross-origin isolation identifies APIs that pose higher risk of process wide XS-Leaks and gate them behind a set of policies (COOP and COEP). These APIs are:
- [performance.measureUserAgentSpecificMemory()](https://developer.mozilla.org/en-US/docs/Web/API/Performance/measureUserAgentSpecificMemory)
- high resolution [performance.now()](https://developer.mozilla.org/en-US/docs/Web/API/Performance/now)
- [SharedArrayBuffers](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)

[Cross-Origin-Embedder-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy) (COEP) addresses the threat to subresources and embedded iframes. It requires non-CORS subresources to either opt into being loaded in the COEP context (COEP require-corp). Or it loads them without credentials (COEP credentialless).

*COEP protects subresources by asking for an opt-in or loading cross-origin resources without credentials.*

![COEP asks subresources for an opt-in or load them without credentials](/images/background3.png)

In the absence of Out-of-process-iframes, iframes are loaded in the same-process as their parent. To protect cross-origin iframes from their embedder, COEP requires cross-origin iframes embedded by a COEP document to opt into being embedded by sending a CORP header.

For the model to work, COEP needs to apply to all documents in the process. So COEP requires documents embedded in a COEP document to also deploy COEP.

Because this is complex to deploy, Chrome also introduced credentialless iframes. A credentialless iframe allows a COEP document to embed documents without COEP. However, those documents only have access to an ephemeral storage partition, initially blank. This ensures that they do not have access to stored credentials. This is safe in the XS-Leak threat model because we are concerned with personalized user data.

*COEP protects cross-origin resources of the page.*

![COEP ensures all resources of a page are protected](/images/background4.png)

[Cross-Origin-Opener-Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy) (COOP) protects other pages that would normally be loaded in the same process as an attacker without Site Isolation. COOP same-origin triggers a browsing context group switch when navigating to a page with a different top-level origin and/or COOP. Pages in a different browsing context group cannot communicate with each other, so it is safe to place them in different processes, even without Out-Of-Process-IFrames (OOPIF).

The proposed [COOP restrict-properties](https://github.com/WICG/coop-restrict-properties) restrict WindowProxy properties available to documents in pages with a different top-level origin. It also prevents synchronous DOM access between documents in pages with a different top-level origin. The latter part allows the browser to place the two pages in different processes.

*COOP protects other pages from an attacker by restricting access to pages opened.*

![COOP restricts access to popups](/images/background5.png)

When the top-level frame enforces both COOP and COEP, the page becomes cross-origin isolated. It gains access to powerful but leaky APIs.

*CrossOriginIsolation provides a secure model to gain access to leaky APIs even in the absence of OOPIF.*

![COI is a safe model for powerful APIs](/images/background6.png)

Finally, the last piece of the puzzle is a [permission policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Permissions_Policy). By default, cross-origin iframes do not have access to COI-gated APIs, even if their parent has enabled COI. The parent must delegate the permission for them to gain access to the APIs. This ensures that the parent is adequately protected against XS-Leaks exploits coming from its child frames.

### Cross origin isolation limits
If crossOriginIsolation is secure, why change the model? Essentially, because it is hard to deploy, which slows down adoption of multi-threading features (SABs, WASM threads) needed to improve web app performance.

#### Deployability
The biggest issue for COI deployment is the cascading requirements that COEP imposes. In order for a page to be crossOriginIsolated, all its frames must enforce COEP. This has proven intractable for a large number of pages.

In order to alleviate the issue, we introduced [credentialless iframes](https://developer.mozilla.org/en-US/docs/Web/Security/IFrame_credentialless). However, credentialless iframes come with a blank ephemeral storage partition. We hoped this drawback would be lessened by 3rd party cookie deprecation (3PCD). In the initial version of 3PCD, third party iframes would not have had access to credentials either, meaning that the delta between loading in a credentialless iframe and a regular iframe would have been small. But 3PCD now includes a sizable number of carve-outs or user driven opt-ins, meaning that it might actually take years for the delta to effectively reduce to a point where using credentialless iframes does not negatively impact functionality.

Finally, credentialless iframes were designed for 3rd party iframes. However, we find that large applications can struggle with getting 1st party iframes to enforce COEP. These iframes might be maintained by other parts of the organization that have different priorities from the top-level frame that want to get access to COI-gated APIs, and might not prioritize supporting COEP. This adds a lot of delay to COI deployment in the top-level frame.

#### Iframes getting access to COI-gated APIs
The crossOriginIsolation model that we built was designed to give access to leaky APIs to the top-level frame (and same-origin frames). Access to COI-gated APIs would then be granted to cross-origin child frames by the top-level frame.

COI was designed in this way because that’s the only model that works without OOPIF. At the same time, use cases that we had for SharedArrayBuffers at the time were all in the top-level frame.

However, since then, various developers have expressed to us the desire to have access to COI-gated APIs in embedded widgets. For example, a chat widget or a 3d map rendering widget might need access to SABs for more efficient calculations, even if the top-level page that embeds it had no interest in SABs itself.

In the current COI model, we have no solution for those use cases. Instead, widget developers would have to maintain two versions of their widgets, with and without SABs, and serve the SAB one only to embedders that deploy crossOriginIsolation. Maintaining two versions of a widget is impractical for most developers.

### Out-of-process iframes evolving landscape
Both of those issues can be solved if we leverage process isolation. We did not do so in the initial COI model because Chrome was the only browser vendor to ship it at the time. However, this is no longer the case.

The other issue was more limited process isolation support on Chrome on Android. However, the APIs gated behind COI are mostly meant for compute heavy websites that are looking for extra computation performance. Developers that want to use SABs are mostly looking at desktop websites, where process isolation is supported. We believe they would be able to navigate the memory cost vs improved computation performance that process isolation on Android implies.

Overall, we believe the time is right to develop a process-isolation based solution for crossOriginIsolation, which will make crossOriginIsolation much more deployable.

## Document-Isolation-Policy

We propose introducing a new security policy, Document-Isolation-Policy which will enable crossOriginIsolation for the document. The safety of the model will be backed by isolating the document when the browser is able to do so. When the browser cannot provide the necessary process isolation, COI-gated APIs will not be available.

### Document-Isolation-Policy header
We introduce a new policy Document-Isolation-Policy, backed by an HTTP header. The policy can have three values:

```
Document-Isolation-Policy: none
Document-Isolation-Policy: isolate-and-credentialless
Document-Isolation-Policy: isolate-and-require-corp
```

`none` is the default value. It does not do anything.

`isolate-and-credentialless` impacts agent cluster allocation for the document. It also forces no-cors cross-origin requests to be sent without credentials for subresources embedded by the document (like [COEP credentialless](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy#credentialless)).

`isolate-and-require-corp` impacts agent cluster allocation for the document. Additionally, the document can only load subresources from the same origin, or subresources explicitly marked as loadable from another origin. For non-cors requests, the response should have an appropriate Cross-Origin-Resource-Policy header (like [COEP require-corp](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy#credentialless)).

### Interaction with COEP
The policy exists side-by-side with [COEP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy) and does not influence it. In particular, an iframe with Document-Isolation-Policy but not COEP may not be embedded in a COEP document.

In case COEP and Document-Isolation-Policy specify different behavior for subresource checks (i.e. require-corp and credentialless), both should be applied.

### Interaction with COOP
[COOP same-origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy#same-origin) and [COOP same-origin-allow-popups](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy#same-origin-allow-popups) force browsing context group switches, which yields a new set of agent clusters. Document-Isolation-Policy would take effect after COOP has been applied.

When it comes to enabling COI, Document-Isolation-Policy can solve the problem [COOP restrict-properties](https://github.com/WICG/coop-restrict-properties) was trying to solve (communicating with cross-origin popups from a COI page). In order to solve the issue, COOP restrict-properties uses a similar mechanism of agent clustering. We believe that it is better to ship only one such mechanism. So our plan is to remove the agent clustering changes from COOP restrict-properties. This means that COOP restrict-properties will not enable COI. Instead, we plan to redesign COOP restrict-properties to focus on protecting websites against WindowProxy-based XS-Leaks (e.g. frame counting). See this [section](#COOP-restrict-properties) around alternatives considered for a longer discussion on why we’re making this choice.

### Interaction with Permission Policy
Currently, COI-gated APIs are further gated behind a [Permission Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Permissions_Policy). They are disabled by default for iframes that are cross-origin with the top-level frame. The sole reason for this situation is that the iframes might be located in the same process as the top-level frame in the absence of OOPIF. Having access to COI-gated APIs would allow them to attack the top-level frame.

If we can support process isolation of an iframe through the Document-Isolation-Policy model, there is no reason to restrict its usage of COI-gated APIs. Normally this should be the case. So a document with COEP and Document-Isolation-Policy should have access to COI-gated APIs, whether the top-level delegated the permission or not.

In some cases, the browser might not be able to support isolating the document with Document-Isolation-Policy in a different process. As discussed in the [agent clustering section](#impact-on-agent-clustering), when this happens the document gets an IsolationLevel of Logical, which does not grant access to COI-gated APIs. So exempting documents with Document-Isolation-Policy from the permission policy for COI-gated APIs is not an issue. We can simply reformulate the existing policy as a policy allowing the use of COI-gated APIs in documents without Document-Isolation-Policy.

Should we introduce a new COI-gated API that should be restricted for cross-origin iframes for reasons other than process-wide XS-Leak risks, then we will add a specific PermissionPolicy for this particular API. It will also apply to documents with Document-Isolation-Policy.

### Interaction with the Origin-Agent-Cluster header
Like COOP and COEP, Document-Isolation-Policy implies `Origin-Agent-Cluster: 1`. In particular, the [Origin-Agent-Cluster](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin-Agent-Cluster) header will be ignored if a Document-Isolation-Policy header is provided. See this [section](#origin-agent-cluster) for a longer discussion on why we chose to introduce a new header rather than extending the Origin-Agent-Cluster header.

### Impact on agent clustering
First, we leverage the [crossOriginIsolationMode](https://html.spec.whatwg.org/#cross-origin-isolation-mode) concept in the HTML spec. CrossOriginIsolationMode has three values:
- None: the default of the web.
- Logical: the context is constrained in its interactions with other contexts, but the browser is backing this isolation by process isolation.
- Concrete: the context is constrained in its interactions with other contexts and the browser is backing this isolation with process isolation. The document is eligible to get access to COI-gated APIs (if ok with Permission Policy).

We then define an IsolationKey, which is composed of:
- An origin. This is the origin of the document that requested isolation (through COOP, COEP and Document-Isolation-Policy).
- A crossOriginIsolationMode of at least Logical.

We then add an optional IsolationKey parameter to agent cluster keys. The agent cluster key then becomes:
- A site URL
- Or an origin.
- Or an origin and an IsolationKey. In this case, the origin and the origin in the IsolationKey can be different. This can be the case for documents in a page with COOP and COEP, where the origin in the isolation key is the top-level origin of the page. Whereas the origin in the agent cluster key is the origin of the document.


By default, documents get an agent cluster key that matches the IsolationLevel of their browsing context group.

If a document has a Document-Isolation-Policy of `isolate-and-credentialless` or `isolate-and-require-corp`, its agent cluster key will be:
- `{document origin, {document origin, concrete}}` if the browser can support the required process isolation.
- `{document origin, {document origin, logical}}` otherwise.

Let’s go in a bit more detail over all the possible situations depending on the IsolationLevel of the browsing context group.

**Case 1: the browsing context group has an IsolationLevel of None**

This is the default of the web. By default, agent clusters for documents have an agent cluster key of `{document origin}`. If the document sends an `Origin-Agent-Cluster: 0` header, then the agent cluster key would be `{document site}`.

If a document has a Document-Isolation-Policy of `isolate-and-require-corp` or `isolate-and-credentialless`, its agent cluster key will be:
- `{document origin, {document origin, concrete}}`if the browser can support the required process isolation.
- `{document origin, {document origin, logical}}` otherwise.

*Document-Isolation-Policy impacts agent clustering. In this example, the two same-origin frames do not have synchronous script access to one another. However, they share the same storage partition and cookie partition.*

![DIP impacts agent clustering](/images/agent-cluster1.png)

*When the browser cannot support process isolation for the document, Document-Isolation-Policy does not give access to COI-gated APIs.*

![No SiteIsolation gives no access to COI gated APIs](/images/agent-cluster2.png)

**Case 2: the browsing context group has an IsolationLevel of Logical**

In this case, all top-level documents in the browsing context group have COEP and COOP. This should normally give them access to COI-gated APIs. But the browser cannot support any form of process isolation so it’s not possible to safely give access to COI gated APIs at all. This is currently the case on Chrome Android WebView.

In this case, documents have an agent cluster key of `{document origin, {top-level origin, logical}}`. The top-level origin is guaranteed to be the same for all documents in the browsing context group due to COOP same-origin.

If a document has a Document-Isolation-Policy of `isolate-and-require-corp` or `isolated-and-credentialless`, its agent cluster key will be `{document origin, {document origin, logical}}`. If the browser cannot support page-level isolation for COI, it certainly cannot support iframe isolation either.

*In this situation, no frame gets access to COI-gated APIs because the browser cannot support the underlying process isolation. The frame with Document-Isolation-Policy does have DOM access to the other same-origin frame.*

![No access to COI gated APIs because of no process isolation](/images/agent-cluster3.png)

**Case 3: the browsing context group has an IsolationLevel of CrossOriginIsolation**

In this case, all top-level documents in the browsing context group have COEP and COOP. This gives them access to COI-gated APIs, backed by process isolation of the browsing context group.

In this case, documents have an agent cluster key of `{document origin, {top-level origin, cconcrete}}`. The top-level origin is guaranteed to be the same for all documents in the browsing context group due to COOP same-origin.

If a document has a Document-Isolation-Policy of `isolate-and-require-corp` or `isolate-and-credentialless`, and the browser can support process isolation for the document, then its agent cluster key will be `{document origin, {document origin, crossOriginIsolation}}`. Note that this may happen in a credentialless iframe, and the document will also get access to COI in this case.

If the browser cannot support process isolation for a document, and the document is same-origin with the top-level frame, then its agent cluster key is still `{document origin, {document origin, crossOriginIsolation}}` because it is the same as `{document origin, {top-level origin, concrete}}`. This allows a website to first gain access to COI using Document-Isolation-Policy before working on getting it through COOP for platforms that do not support Document-Isolation-Policy.

If the browser cannot support process isolation for a document cross-origin with the top-level frame, then we have again two options:
1. Setting its agent cluster key to `{document origin, {document origin, logical}}`. This means the document will not get access to COI-gated APIs, but will have an isolation behavior that is consistent.
2. Keeping its agent cluster key to `{document origin, {top-level origin, crossOriginIsolation}}`. In this case, the document might get access to COI-gated APIs if the permission is delegated by the parent.

We believe that option 1 is the best again, due to consistency. The document will never get access to COI gated APIs. But this case can only happen if either:
- A same-origin property decides to both get access to COI through COOP and Document-Isolation-Policy. This can easily be fixed by choosing one or the other.
- A cross-origin iframe with Document-Isolation-Policy gets embedded in a page with COOP and COEP. But it would need permission delegation to use COI-gated APIs in the first place, which is unlikely to happen.

*In this situation, the two b.com iframes without Document-Isolation-Policy have DOM access to each other, but not to the iframe with Document-Isolation-Policy. The iframes not loaded in a credentialless iframe share a StorageKey and NetworkKey.*

![Different COI iframes](/images/agent-cluster4.png)

*In this situation, the browser cannot support frame isolation. The b.com ifames without Document-Isolation-Policy have DOM access to each other. The ones not loaded in an anonymous iframe share a StorageKey and NetworkKey.*

![Only page level COI suuported](/images/agent-cluster5.png)

#### A note on SharedArrayBuffers and agent clusters
SharedArrayBuffers can be passed to other execution contexts in the same agent clutser using postMessage. So Document-Isolation-Policy impacting agent clusters will also impact the ability to post SABs to other documents. In particular, it will not be possible to share a SAB from a crossOriginIsolated document to one that isn't, because they would not be in the same agent cluster. This is the case even if they are same-origin. We should probably update the error message developers get in that case, so that it mentions other factors beyond origin as the reason the SAB could not be posted.

### Interaction with Fetch
Document-Isolation-Policy can apply checks on subresources that are similar to COEP.

`Document-Isolation-Policy: isolate-and-require-corp` requires cross-origin subresources not loaded through CORS to have a CORP header to be loaded. This is checked during the [CORP check](https://fetch.spec.whatwg.org/#cross-origin-resource-policy-header) in Fetch. The existing check for COEP should be extended so that Document-Isolation-Policy is also taken into account.

`Document-Isolation-Policy: credentialless` requires cross-origin subresource requests made without CORS to be loaded without credentials. We’ll extend the check in the [main Fetch algorithm](https://fetch.spec.whatwg.org/#http-network-or-cache-fetch) so that it also applies to Document-Isolation-policy and not just COEP credentialless.

### Inheritance
Document-Isolation-Policy will be stored in the PolicyContainer and follow the regular inheritance model for policy inheritance (unlike COOP and COEP). The underlying process isolation ensures that documents with Document-Isolation-Policy only share their process with same-origin documents that are crossOriginIsolated. So any new document created by the DIP document in the same process should also be same-origin and crossOriginIsolated. The standard behavior for policy inheritance is to inherit the origin from the creator, and its security policies. In this case, the document would inherit its origin and Document-Isolation-Policy from its creator, making crossOriginIsolated.

### Interactions with workers
Document-Isolation-State and the crossOriginIsolation status should be inherited from the creator of a DedicatedWorker or a SharedWorker.

ServiceWorkers should honor the subresource checks mandated by Document-Isolation-Policy, just as they do for COEP. ServiceWorkers can also have a COEP of their own. There is no plan to do something similar for Document-Isolation-Policy. For ServiceWorkers, COEP is sufficient to enable crossOriginIsolation and there are no deployment concerns that Document-Isolation-Policy would fix. In fact, a Document-Isolation-Policy specific to ServiceWorkers would look exactly like COEP.

### Interaction with Isolated Web Apps
TODO

### Reporting
We will provide reporting for Document-Isolation-Policy using the reporting API. We will also provide a report-only mode for the policy. This will be similar to COOP and COEP.

Example:

```
Document-Isolation-Policy: isolate-agent-cluster;report-to="endpoint"
Document-Isolation-Policy-Report-Only: isolate-agent-cluster;report-to="endpoint"
```

The browser will send reports when it detects same-origin documents with a different crossOriginIsolationMode in the browsing context group. In report-only mode, the browser sends reports when it detects same-origin documents in the browsing context group that would have a different IsolationLevel if Document-Isolation-Policy was enforced. This allows the developer to know that those documents have lost/would lose DOM access to each other.

When navigating to a document with Document-Isolation-Policy reporting enabled, the browser collects a list of all same-origin documents in the browsing context. Then, for each document:
1. Check if the crossOriginIsolationMode of the navigating document matches the one of the existing document.
2. If not:
    1. If the navigating document’s DocumentIsolationPolicy has an endpoint, send a report to it.
    2. If the existing document's DocumentIsolationPolicy has an endpoint, send a report to it.
3. Compute the crossOriginIsolationMode that would be applied to the navigating document and the existing document if their report-only DocumentIsolationPolicy was applied. If they match do not send a report. Developers will often deploy a report-only policy on several frames before actually enforcing it. This avoids sending them spurious reports.
4. Check the report-only crossOriginIsolationMode of the navigating document against the IsolationLevel of the existing document and vice versa. If any of the checks match, do not send a report.
5. If there was no match using report-only policies:
    1. If the navigating document’s report-only DocumentIsolationPolicy has an endpoint, send a report to it.
    2. If the existing document's report-only DocumentIsolationPolicy has an endpoint, send a report to it.

A report would contain the following:

```
{
reportingDocumentUrl: URL,
reportingDocumentCrossOriginIsolationMode: cross-origin-isolation-mode,
reportingDocumentIsolationPolicy: document-isolation-policy value,
otherDocumentUrl: URL,
otherDocumentCrossOriginIsolationMode: cross-origin-isolation-mode,
otherDocumentIsolationPolicy: document-isolation-policy value,
enforcement: boolean,
}
```

Because all the documents are same-origin with each other and in the same browsing context group, the report does not leak any kind of information. Should communication between same-origin frames in a browsing context group be limited due to 3PCD, we will limit the reports sent accordingly.

While it would be ideal to warn about DOM accesses that would be blocked by Document-Isolation-Policy, this would impose too much of a performance cost. So reports will be limited to warning about potential issues between frames.

For subresource checks, we will send reports similar to COEP require-corp and credentialless.

### How this solution would solve the use cases

#### App with cross-origin popup

The app can deploy a Document-Isolation-Policy of `isolated-and-credentialless` or `isolated-and-require-corp` on its documents. It will gain access to COI gated APIs, while retaining communication with cross-origin popups. It can also deploy Document-Isolation-Policy just on documents that need COI-gated APIs, but those document would lose direct DOM access to the rest of the same-origin documents of the app.

If the platform does not have enough resources to support process isolation of the documents, then the app does not gain access to COI gated APIs. However, our use case is a compute heavy app that needs multithreading for better performance. The platforms these kinds of app run on should have enough resources to support the required isolation.

#### App with 3rd party iframe

Exactly like in the app with cross-origin popup case, the app can deploy a Document-Isolation-Policy of `isolated-and-credentialless` or `isolated-and-require-corp` on its documents. If the platform can support the isolation, it will get access to COI gated APIs. In any case, there are no particular restrictions on embedding cross-origin iframes.

#### Embedded widget

An embedded iframe can deploy a Document-Isolation-Policy of `isolated-and-credentialless` or `isolated-and-require-corp` and gain access to COI-gated APIs, as long as the platform can support the necessary isolation. This is regardless of whether its embedder deployed COI or not.

## Considered alternatives

### COOP restrict-properties

[COOP restrict-properties](https://github.com/WICG/coop-restrict-properties) aims at solving the problem of deploying crossOriginIsolation while preserving communication with popups and also preventing various XS-Leaks based on windowProxy APIs (such as frame counting). It achieves the former by having a similar effect on agent clustering as Document-Isolation-Policy at a page level, and the latter by restricting WindowProxy access between pages with different top-level origins.

The pros and cons of COOP restrict-properties compared to Document-Isolation-Policy when it comes to COI deployment are the following:
Pros:
- Can be supported without having out-of-process iframes.

Cons:
- Does not solve the issue of embedding 3rd-party frames, only communication with cross-origin popups.
- Does not allow to have COI in cross-origin subframes without the parent also having COI.

Various developers have been clear that they would be fine with a solution that works only on desktop but provides much easier COI deployment. After all, several users of SABs are using the reverse OT on Chrome which provides these exact properties (Chrome desktop only with no COI deployment). Overall, this means that even if we were to release COOP restrict-properties in its current state, we would still need to release Document-Isolation-Policy. The question is then should we release COOP restrict-properties in its current state?

On the one hand, it would provide a way for pages in Chrome on low-end Android to get COI support if they have cross-origin popups. It might also be implemented by other browser vendors faster, though it's unclear if that is actually the case since the implementation of COOP restrict-properties is non-trivial, as we found out while implementing it on Chrome. While support for Safari is certainly of interest to websites that want to use SABs, support for low-end Android does not seem to be important. It seems that low-end Android cannot support heavy web apps that want to make use of SABs in the first place.

Keeping the COI part in COOP restrict-properties has a complexity cost. Spec-wise, COOP restrict-properties need to rely on an agent keying mechanism that is similar but slightly different compared to Document-Isolation-Policy. We believe that having the two existing at the same time is going to be hard for developers to wrap their heads around. And it involves extra work for Document-Isolation-Policy, as we now have to take into account the interactions with COOP restrict-properties, which are not described in this explainer.

The case for cross-origin popups is much simpler with Document-Isolation-Policy compared to COOP restrict-properties. In Chrome, we have a constraint that the initial empty document for a popup is always created in the process of its creator. With COOP restrict-properties, this is an issue when a cross-origin iframe opens a popup:

![COOP rp and cross-origin popups](/images/coop-rp-popup.png)

The initial empty document opened by a cross-origin iframe needs to be in its process, unless opened with rel no-opener Because the current model does not force process isolation, the cross-origin iframe can be in the same process as the top-level frame. Which means that a popup it opens will be in the process of the top-level frame as well. COOP same-origin avoids this issue by forcing popups opened from cross-origin frames to have rel no-opener. But that’s not practical for COOP restrict-properties which aims to preserve opener relationships. Instead, COOP restrict-properties add an origin to COOP to track which origin sent the COOP header in the first place. This origin is used to match COOP, and to restrict access to COI gated APIs when the COOP origin doesn’t match the top-level origin.

This problem simply does not exist with Document-Isolation-Policy since we’re guaranteeing that the only documents in the process of the Document-Isolation-Policy + COEP are same-origin and also have Document-Isolation-Policy + COEP. So when any of those opens an about:blank popup, it is placed in the same process and it naturally inherits origin, Document-Isolation-Policy and COEP from the PolicyContainer, which is what we want.

The COI constraints also hold back the design of COOP restrict-properties when it comes to protecting against WindowProxy-based XS-Leaks. If we do not have to uphold a particular process model in the absence of OOPIF, then we can provide more configuration options for COOP: restrict-properties. For example, applying to subframes, configuring which properties are restricted, allow-listing origins allowed to access, exempting cross-origin but same-site documents from the restrictions, …. This would make the policy more useful to users who want to use it to protect against WindowProxy XS-leaks. The fact that it no longer impacts same-origin frames makes it easier to deploy.

Overall, there are more benefits to removing the interaction between COOP restrict-properties and Document-Isolation-Policy than trying to ship it and Document-Isolation-Policy at the same time. We should pursue this instead of attempting to make the two fit together and deal with extra-complexities.

### Origin-Agent-Cluster

[Origin-Agent-Cluster](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin-Agent-Cluster) is an existing header that allows a document to specify that it wants to be placed in an agent cluster keyed on origin rather than site URL. We could have tried to piggyback on the header rather than introduce a new header. There are a few rough edges when doing that:
Origin-Agent-Cluster is meant to apply to a whole origin, and one document failing to send the header can lead to the default value being applied (no isolation). This is fine for memory optimization, but problematic for a header meant to gate access to APIs. Developers want consistent access to COI-gated APIs to avoid having to maintain two versions of their website.
Origin-Agent-Cluster is meant as a performance header rather than a security one. This means that the browser makes no security guarantee, but also that Origin-Agent-Cluster will not negatively impact the performance of the website. Here, we want to make a different tradeoff with Document-Isolation-Policy, where we guarantee security at a potential memory cost.
Origin-Agent-Cluster does not have a reporting policy, so we would have to introduce one.
Origin-Agent-Cluster only applies to agent clustering, and we also need to impact sub-resource load.

Overall, we can try to add a new value to Origin-Agent-Cluster rather than introduce a new header, but this new value will need to behave like the Document-Isolation-Policy header described here rather than the existing Origin-Agent-Cluster header. At this point, it is debatable what is clearer for developers.

### Keeping the status quo
CrossOriginIsolation defined by COOP + COEP is a model that is safe, can be supported by Chrome everywhere but on Android WebView, and could be supported by most browsers. It’s also very hard to deploy, leading to little adoption of APIs like SharedArrayBuffers (0.05% of page views instantiate a SAB in a COI environment). And we know that various developers like Google Workspace or Adobe Photoshop want to use SABs but do not manage to deploy crossOriginIsolation. SABs can improve computation time for heavy websites by ~40%, so getting access to them is needed to unlock more native-like computing capabilities for web applications. This is why the status quo is not desirable.

We designed crossOriginIsolation in a context where OOPIFs were only available in Chrome. But since then, they have shipped on Firefox. Now that the feature is available in two different browser engines, we believe it should be part of the tradeoffs we’re looking at when proposing new web platform APIs.

In this context, we think that it is better to revisit the tradeoffs we made when designing crossOriginIsolation. While Document-Isolation-Policy might not be supported on all platform initially (i.e. if memory constraints are too tight), it will greatly improve deployability on platforms where accessibility to SharedArrayBuffers matters.

### Provide access to COI-gated APIs when full SiteIsolation is available
This would mean going back to the situation prior to the introduction of crossOriginIsolation, where SABs were only available on Chrome desktop. Obviously, this makes SABs easily accessible to websites on Chrome desktop. However, compared to the status quo, this has the following drawbacks:
- It does not protect subresources.
- It does not protect cross-origin but same-site frames, unless the browser decides to use origin isolation. This is a problem for large entities which may have many different origins in the same site. One origin being compromised can then attack other origins with much higher value.
- SABs would not be available on mobile because no browser on mobile supports full Site Isolation.
- Browsers may implement process reuse mechanisms that can be abused by regular processes, so an attacker could force a cross-site document in its process. We could get around the issue by requiring a header in order to use COI-gated APIs and not follow this process reuse mechanism, but then that would essentially mean that we’re doing Document-Isolation-Policy just without the subresource part, which leaves subresources vulnerable.

Compared to Document-Isolation-Policy, just relying on SiteIsolation is easier to deploy as it does not require COEP checks on subresources (credentialless or require corp). However, it has the exact same drawbacks as compared to the status quo. In particular, we think that we can support Document-Isolation-Policy on regular Chrome Android because it only isolates documents that want to use SABs instead of requiring full SiteIsolation for all websites on the platform. Thus, we believe that Document-Isolation-Policy has better tradeoffs for solving the issue of access to SABs than relying on full SiteIsolation.

### Browsing context group switch instead of agent cluster keying
The Document-Isolation-Policy proposal relies on agent cluster keying to achieve isolation, instead of browsing context group switches. This means that it introduces a situation where two same-origin documents might find themselves in different agent clusters and be unable to have DOM access to each other. This is unprecedented in the HTML spec.

COOP restrict-properties was implemented in Chrome using the CoopRelatedGroup, which is an umbrella group of browsing context groups that still have access to each other WindowProxies, but no DOM access. In both cases, we want to have same-origin documents in a situation where they have asynchronous access through WindowProxy, but not synchronous DOM access.

We believe that agent cluster keying is more appropriate in this case as the same-origin documents that don’t have access to each other might be on the same page. It would be weird for them to be in different browsing context groups.

Additionally, the state we want to have is exactly the one of documents in different agent clusters. Rather than introducing a new concept that provide the same isolation as agent clusters, we believe that it is better to change how agent clusters are allocated.

### Relying on COEP for the subresources checks
The proposal for Document-Isolation-Policy bundles COEP-like checks on subresources with changes to agent clustering. We could imagine having the two as separate, with Document-Isolation-Policy just impacting agent clustering and waiving the COEP checks on embedded iframes. The later part is a bit complex, since we can't just take into account the COEP of the frame, but we must look at the whole ancestor chain to know if it is safe to waive the checks or not. Otherwise, we'd risk undermining the security model of COI by waiving checks in the case where we can support page-level but not frame-level COI. This complexity also translates into the reporting, which has to take into account both COEP and Document-Isolation-Policy. We believe it is simpler for developers to just have 1 header that does everything instead of two headers with complex interactions. However, that means we have to re-implement the subresources checks and reporting that are part of COEP.

### Restricting the ability to use Document-Isolation-Policy for cross-origin iframes

Document-Isolation-Policy requires process isolation, which has a memory cost. If too many iframes in a page end up being process isolated, this could have a performance cost for the page, so we could have introduced a permission policy to restrict the ability of cross-origin iframes to get process isolated using Document-Isolation-Policy. This would also restrict their ability to use COI_gated APIs, as they cannot be properly isolated. We opted against it for two reasons:

- we want to solve the use case of embedded widgets. Having their embedder be able to restrict the ability to use COI gated APIs is problematic, as this means that website developers have to maintain two versions of their widgets (with and without COI gated APIs), which is more costly and complex.
- the problem of cross-origin iframes using too much memory is a more global one, and it's not clear this particular API should be restricted as opposed to the multitude of other APIs that also use memory.

## Stakeholder Feedback / Opposition

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- [Artur Janc](https://github.com/arturjanc)
- [Charlie Reis](https://github.com/csreis)
- [David Dworken](https://github.com/ddworken)
- [Domenic Denicola](https://github.com/domenic)
- [Mike West](https://github.com/mikewest)
