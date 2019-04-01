![true\[X\] logo](media/truex.png)

# TruexAdRenderer Roku Documentation

Version 1.0.0

## Contents

* [Overview](#overview)
* [Product Flows](#product-flows)
* [How to use TruexAdRenderer](#how-to-use-truexadrenderer)
    * [When to show True\[X\]](#when-to-show-truex)
    * [Handling Events from TruexAdRenderer](#handling-events-from-truexadrenderer)
        * [Terminal Events](#terminal-events)
    * [Handling Ad Elimination](#handling-ad-elimination)
* [TruexAdRenderer Roku API](#truexadrenderer-roku-api)
    * [Reference to true\[X\] Component Library](#reference-to-the-truex-component-library)
    * [`TruexAdRenderer` Input Events](#truexadrenderer-input-events)
        * [`init`](#init)
        * [`start`](#start)
    * [`TruexAdRenderer` Output Events -- Main Flow](#truexadrenderer-output-events----main-flow)
        * [`adFetchCompleted`](#adfetchcompleted)
        * [`adStarted`](#adstarted)
        * [`adCompleted`](#adcompleted)
        * [`adError`](#aderror)
        * [`noAdsAvailable`](#noadsavailable)
        * [`adFreePod`](#adfreepod)
        * [`userCancelStream`](#usercancelstream) 
    * [`TruexAdRenderer` Output Events -- Informative](#truexadrenderer-output-events----informative)
        * [`optIn`](#optin)
        * [`optOut`](#optout) 
        * [`skipCardShown`](#skipcardshown) 
        * [`userCancel`](#usercancel) 
        * [`videoEvent`](#videoevent) 

## Overview

In order to support interactive ads on Roku, true\[X\] has created a renderer component library that can renderer true\[X\] ads natively, which interfaces with a hosting channel, as well as its existing ad server and content delivery mechanism (e.g. SSAI).

With this library, the host player channel can defer to the TruexAdRenderer when it is required to display a true\[X\] ad.

For simplicity, publisher implemented code will be referred to as "channel code" while true\[X\] implemented code will be referred to as "renderer code".

true\[X\] will provide a Roku `TruexLibrary` component library that can be loaded into the channel. This library will offer a component, `TruexAdRenderer`, that will need to be instantiated, initialized and given certain commands (described below in [TruexAdRenderer Input Events](#truexadrenderer-input-events)) by the channel code.

At this point, the renderer code will take on the responsibility of requesting ads from true\[X\] server, creating the native UI for the true\[X\] choice card and interactive ad unit, as well as communicating events to the channel code when action is required.

The channel code will still need to parse out the SSAI ad response, detect when a true\[X\] ad is supposed to display, pause the stream, instantiate `TruexAdRenderer` and handle any events emitted by the renderer code.

It will also need to handle skipping ads in the current ad pod, if it is notified to do so.


## Product Flows

There are two distinct product flows supported by `TruexAdRenderer`: Sponsored Stream (full-stream ad-replacement) and Sponsored Ad Break (mid-roll ad-replacement).

In a Sponsored Ad Break flow, once the user hits a mid-roll break with a true\[X\] tag flighted, they will be shown a "choice-card" offering them the choice between watching a normal set of video ads or a fully interactive true\[X\] ad:

![choice card](media/choice_card.png)

***Fig. A** example true\[X\] mid-roll choice card*

If the user opts for a normal ad break, or if the user does not make a selection before the countdown timer expires, the true\[X\] UI will close and playback of normal video ads can continue as usual.

If the user opts to interact with TrueX, an interactive ad unit will be shown to the user:

![ad](media/ad.png)

***Fig. B** example true\[X\] interactive ad unit*

The requirement for the user to "complete" this ad is for them to spend at least the allotted time on the unit and for at least one interaction (e.g. navigating anywhere through the ad).

![true attention timer example](media/true-attention-timer-example.png)

***Fig. C** example true\[X\] attention timer*

Once the user fulfills both requirements, a "Watch Your Show" button will appear in the bottom right, which the user can select to exit the true\[X\] ad. Having completed a true\[X\] ad, the user will be returned directly to content, skipping the remaining ads in the current ad pod.

The Sponsored Stream flow is quite similar. In this scenario, a user will be shown a choice-card in the pre-roll:

![choice card](media/choice_card.png)

***Fig. D** example true\[X\] pre-roll choice card (full-stream replacement)*

Similarly, if the user opts-in and completes the true\[X\] ad, they will be skipped over the remainder of the pre-roll ad break. However, every subsequent mid-roll break in the current stream will also be skipped over. In this case instead of the regular pod of video ads, the user will be shown a "hero card" (also known as a "skip card"):

![skip card](media/skip_card.png)

***Fig. E** example true\[X\] mid-roll skip card*

This messaging will be displayed to the user for several seconds, after which they will be returned directly to content.

## How to use TruexAdRenderer 

### When to show True\[X\]

Upon receiving an ad schedule from your SSAI service, you should be able to detect whether or not true\[X\] is returned in any of the pods. true\[X\] ads should have `apiFramework` set to `VPAID` or `truex`.

SSAI vendors differ in the way they convey information back about ad schedules to clients. Certain vendors such as Verizon / Uplynk expose API’s which return details about the ad schedule in a JSON object. For other vendors, for instance Google DAI, the true\[X\] payload will be encapsulated as part of a companion payload on the returned VAST ad. The Roku RAF library also exposes various wrappers which encapsulate vendor specific logic into a simple interface. Please work with your true\[X\] point of contact if you have difficulty identifying the right approach to detecting the true\[X\] placeholder, which will be the trigger point for the ad experience.

Once the player reaches a true\[X\] placeholder, it should pause, instantiate the `TruexAdRenderer` and immediately trigger `init` followed by `start` events.

Alternatively, you can instantiate and `init` the `TruexAdRenderer` in preparation for an upcoming placeholder. This will give the `TruexAdRenderer` more time to complete its initial ad request, and will help streamline true\[X\] load time and minimize wait time for your users. Once the player reaches a placeholder, it can then call `start` to notify the renderer that it can display the unit to the user.

### Handling Events from TruexAdRenderer

Once `start` has been called on the renderer, it will start to emit events (see [`TruexAdRenderer` Output Events -- Main Flow](#truexadrenderer-output-events----main-flow) and [`TruexAdRenderer` Output Events -- Informative](#truexadrenderer-output-events----informative)).

One of the first events you will receive is `adStarted`. This notifies the channel that the renderer has received an ad for the user and has started to show the unit to the user. The channel does not need to do anything in response, however it can use this event to facilitate a timeout. If an `adStarted` event has not fired within a certain amount of time, the host channel can proceed to normal video ads.

At this point, the channel code must listen for the renderer's *terminal events* (described below), while paying special attention to the `adFreePod` event. A *terminal event* signifies that the renderer is done with its activities and the channel may now resume playback of its stream. The `adFreePod` event signifies that the user has earned a credit with true\[X\] and all linear video ads remaining in the current pod should be skipped. If the `adFreePod` event did not fire before a terminal event is emitted, the channel should resume playback without skipping any ads, so the user receives a normal video ad payload.

#### Terminal Events

The *terminal events* are:
* `adCompleted`: the user has exited the true\[X\] ad
* `userCancelStream`: the user intends to exit the current stream entirely
* `noAdsAvailable`: there were no true\[X\] ads available to the user
* `adError`: the renderer encountered an unrecoverable error

It's important to note that the player should not immediately resume playback once receiving the `adFreePod` event -- rather, it should note that it was fired and continue to wait for a terminal event.


### Handling Ad Elimination

Skipping video ads is completely the responsibility of the channel code. The SSAI API should provide enough information for the channel to determine where the current pod end-point is, and the channel, when appropriate, should fast-forward directly to this point when resuming playback.

## TruexAdRenderer Roku API

This is an outline of `TruexAdRenderer` input and output events. Input events are assigned on the `action` field from the interface of the `TruexAdRenderer`, while output events are emitted against the `event` field on the same component.

### Reference to the true\[X\] Component Library

The true\[X\] interactive ad component and its rendering logic are distributed as part of a component library. It is required for the hosting channel to add a reference to the component library in order for it to be used, for instance via the following reference added to your channel’s main scene:

    <ComponentLibrary id="TruexAdLibrary" uri="http://static.truex.com.s3.amazonaws.com/roku/TruexAdRenderer-Roku-v0.9.0.pkg"/>

### TruexAdRenderer Input Events

#### `init`

```
m.tar = m.top.createChild("TruexAdLibrary.TruexAdRenderer")
m.tar.observeFieldScoped("event", "handleTarEvent")
m.tar.observeFieldScoped("request", "handleTarPlaybackRequest")

m.tar.action = {
    type : "init",
    adParameters : "<Ad parameters associative array as returned by SSAI>",
    slotType : "<the type of the current ad pod>",
    supportsUserCancelStream : <optional; set to true to enable the userCancelStream event>
}
```

This event will be triggered by the channel code in order to initialize the `TruexAdRenderer`. The renderer will parse out the `adParameters` and `slotType` passed to it and make a request to the true\[X\] ad server to see what ads are available.

You may initialize `TruexAdRenderer` early (a few seconds before the next pod even starts) in order to give it extra time to make the ad request. The renderer will output an `adFetchCompleted` event at completion of this ad request. This event can be used to facilitate the implementation of a timeout or loading indicator, and when to make the call to `start`.

The parameters for this method call are:

* `adParameters`: AdParameters as returned by SSAI. In the example of Uplynk, this would correspond to `response.ads.breaks[0].ads[0].adParameters`
* `slotType`: the type of the current ad pod, `PREROLL` or `MIDROLL`
* `supportsUserCancelStream`: optional -- set to `true` to enable the [userCancelStream](#usercancelstream) event


#### `start`

```
m.tar.action = {
    type : "start"
}
```

This method should be called by the channel code when the channel is ready to display the true\[X\] unit to the user. This can be called anytime after the unit is initialized.

The channel should have as much extraneous UI hidden as possible, including player controls, status bars and soft buttons/keyboards.

After calling `start`, the channel code should wait for a [terminal event](#terminal-events) before taking any more action, while keeping track of whether or not the [`adFreePod`](#adfreepod) event has fired.

In a non-error flow, the renderer will first wait for the ad request triggered in `init` to finish if it has not already. It will then display the true\[X\] unit to the user in a new component (added to the `TruexAdRenderer` parent component) and then fire the [`adStarted`](#adstarted) event. `adFreePod` and other events may fire after this point, depending on the user's choices, followed by one of the [terminal events](#terminal-events).

### `TruexAdRenderer` Output Events -- Main Flow

The following events signal the main flow of the `TruexAdRenderer` and may require action by the host channel:


#### `adFetchCompleted`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "adFetchCompleted"
' }
```

This event fires in response to the `init` method when the true\[X\] ad request has successfully completed and the ad is ready to be presented. The host channel may use this event to facilitate a loading screen for pre-rolls, or to facilitate an ad request timeout for mid-rolls.

For example: `init` is called for the pre-roll slot, and the channel code shows a loading indicator while waiting for `adFetchCompleted`. Then, either the event is received (and the channel can call `start`) or a specific timeout is reached (and the channel can remove the `TruexAdRenderer` component and resume playback of normal video ads).

Another example: `init` is called well before a mid-roll slot to give the renderer a chance to preload its ad. If `adFetchCompleted` is received before the mid-roll slot is encountered, then the channel can call `start` to immediately present the true\[X\] ad. If not, the channel can wait for a specific timeout (if not already reached, in case the user has seeked to activate the mid-roll) before removing the `TruexAdRenderer` component and resuming playback with normal video ads.


#### `adStarted`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "adStarted",
'     campaignName : <string representing the campaign name>
' }
```

This event will fire in response to the `start` input event when the true\[X\] UI is ready and has been added to the component hierarchy.

The parameters for this event are:

* `campaignName`: The name of the ad campaign available to the user (e.g. "*Storebought Coffee - Sample Video Ad #1 - Q1 2017*")

#### `adCompleted`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "adCompleted",
'     timeSpent : <integer representing the amount of time spent>
' }
```

This is a [terminal event](#terminal-events). This event will fire when the true\[X\] unit is finished with its activities -- at this point, the channel should resume playback and remove the `TruexAdRenderer` component from the Scene Graph.

Here are some examples where `adCompleted` will fire:

* The user opts for normal video ads (not true\[X\])
* The choice card countdown runs out
* The user completes the true\[X\] ad unit
* After a "skip card" has been shown to a user for its duration

The parameters for this event are:

* `timeSpent`: The amount of time (in seconds) the user spent on the true\[X\] unit

#### `adError`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "error",
'     errorMessage : <string representing the error message>
' }
```

This is a [terminal event](#terminal-events). This event will fire when the true\[X\] unit has encountered an unrecoverable error. The channel code should handle this the same way as an `adCompleted` event - resume playback and remove the `TruexAdRenderer` component from the Scene Graph.

The parameters for this event are:

* `errorMessage`: A description of the cause of the error.


#### `noAdsAvailable`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "noAdsAvailable"
' }
```

This is a [terminal event](#terminal-events). This event will fire when the true\[X\] unit has determined it has no ads available to show the current user. The channel code should handle this the same way as an `adCompleted` event - resume playback and remove the `TruexAdRenderer` component from the Scene Graph.


#### `adFreePod`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "adFreePod"
' }
```

This event will fire when the user has earned a credit with true\[X\]. The channel code should notate that this event has fired, but should not take any further action. Upon receiving a [terminal event](#terminal-events), if `adFreePod` was fired, the channel should skip all remaining ads in the current slot. If it was not fired, the channel should resume playback without skipping any ads, so the user receives a normal video ad payload.


#### `userCancelStream`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "userCancelStream"
' }
```

This is a [terminal event](#terminal-events), and is only enabled when the `supportsUserCancelStream` property is set to `true` when triggering the [`init`](#init) action.

When enabled, the renderer will fire this event when the user intends to exit the stream entirely. The channel, at this point, should treat this the same way it would handle any other "exit" action from within the stream -- in most cases this will result in the user being returned to an episode/series detail page.

If this event is not enabled, then the renderer will emit an [`adCompleted`](#adcompleted) event instead.

### `TruexAdRenderer` Output Events -- Informative

All following events are used mostly for tracking purposes -- no action is generally required.


#### `optIn`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "optIn"
' }
```

This event will fire if the user selects to interact with the true\[X\] interactive ad.

Note that this event may be fired multiple times if a user opts in to the true\[X\] interactive ad and subsequently backs out.

#### `optOut`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "optOut"
'     userInitiated : <true or false>
' }
```

This event will fire if the user opts for a normal video ad experience.

The parameters for this event are:

* `userInitiated`: This will be set to `true` if this was actively selected by the user, `false` if the user simply allowed the choice card countdown to expire.


#### `skipCardShown`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "skipCardShown"
' }
```

This event will fire anytime a "skip card" is shown to a user as a result of completing a true\[X\] Sponsored Stream interactive in an earlier pre-roll.


#### `userCancel`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "userCancel"
' }
```

This event will fire when a user backs out of the true\[X\] interactive ad unit after having opted in. This would be achieved by tapping the "Yes" link to the "Are you sure you want to go back and choose a different ad experience" prompt inside the true\[X\] interactive ad. The user will be subsequently taken back to the Choice Card (with the countdown timer reset to full).

Note that after a `userCancel`, the user can opt-in and engage with an interactive ad again, so more `optIn` or `optOut` events may then be fired.


#### `videoEvent`

```
function handleTarEvent(evt as Object) as Void
adEvent = evt.getData()

' adEvent : {
'     type : "videoEvent",
'     subType : "<the subType -- see below>",
'     videoName : "<the name of the video, if available>",
'     url : "<the source URL of the video, if available>"
' }
```

A `videoEvent` is emitted when noteworthy events occur in a video within a true\[X\] unit. These events are differentiated by their `subType` value as follows:

* `started`: A video has started
* `firstQuartile`: A video has reached its first quartile
* `secondQuartile`: A video has reached its second quartile
* `thirdQuartile`: A video has reached its third quartile
* `completed`: A video has reached completion

The parameters for this event are:

* `subType`: The subType of the emitted video event
* `videoName`: The name of the video that emitted the event, if available
* `url`: The source URL of the video that emitted the event, if available
