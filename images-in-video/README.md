# Images in `<video>`

Authors: Lea Verou, Florian Rivoal

<details open>
<summary>Contents</summary>

1. [User Needs \& Use cases](#user-needs--use-cases)
2. [User research](#user-research)
   1. [Current workarounds](#current-workarounds)
   2. [Developer signals](#developer-signals)
3. [Goals](#goals)
   1. [Non-goals](#non-goals)
4. [Proposed solution](#proposed-solution)
5. [Discussion](#discussion)
   1. [Emulating current `<img>` behavior with `<video>`](#emulating-current-img-behavior-with-video)
   2. [Is it web compatible?](#is-it-web-compatible)
   3. [How to treat static images?](#how-to-treat-static-images)
   4. [Would this encourage higher usage of inefficient video formats?](#would-this-encourage-higher-usage-of-inefficient-video-formats)
   5. [What about synchronization?](#what-about-synchronization)
6. [Sample code snippets](#sample-code-snippets)
7. [Future improvements](#future-improvements)
8. [Alternatives considered](#alternatives-considered)
   1. [Making `<img>` and/or `<picture>` media elements](#making-img-andor-picture-media-elements)
   2. [Controlling image animation in CSS](#controlling-image-animation-in-css)
</details>

## User Needs & Use cases

In websites with user-generated content that support image embeds, such as social media, users often embed animated images, such as animated GIFs or APNG. By default, UAs autoplay and loop these images, which can be jarring for users, especially in use cases where there are multiple images on a single page (e.g. image galleries) and violates WCAG. Websites need to expose ways for users to control playback, especially around turning off autoplay and/or looping.

Some examples:

* Email client that wants to restrict animation for animated images sent in emails (e.g. loop once)
* Image gallery, where rather than all images animating together, users hover or focus an image to trigger autoplay
* Animated emojis sent in social media

There is a wide range of use cases and desired UIs and user experiences, so making this an automatic UA feature would not be sufficient (and is likely not web-compatible).
For example:

* Start paused and play on hover and focus
* Start paused with a play button and toggle between playing and paused states on click
* Autoplay unless `(prefers-reduced-motion)` is on
* Play once, when in view, then provide playback UI to replay

> TODO: screenshots/casts

## User research

### Current workarounds

The cornucopia of existing workarounds demonstrates demand for this feature. The significant downsides of these workarounds demonstrate why this needs to be solved in the platform.

The prominent current approaches can be summarized as:

1. Scripts or components that render image on canvas to get freeze frames, then use custom UI to toggle between them
  * Examples: `<wa-animated-image>` (and older `<sl-animated-image>`), `<gif-player>`, `react-gif-player`, `Giffer`, `Freezeframe`, `buzzfeed/libgif-js`
2. Server-side frame extraction, custom UI with JS to toggle between them
3. Server-side conversion to video, then just use `<video>` . Many large-scale gif galleries do this but such a computationally expensive solution is not always within reach for smaller publishers.

Server side solutions are computationally expensive (and thus, costly), and not always an option due to video encoding patents. Additionally converting to video is a poor solution when an alpha channel is desired, as video formats with alpha channels have less broad device/app support.

Client-side solutions are subject to same-origin restrictions, so they are a poor solution for user generated content, and also break progressive rendering. Since they depend on simply swapping static frames with the actual animated image, they have no control over looping or other aspects of playback — such control demands even more heavyweight solutions such as reimplementing image decoding on the client.

### Developer signals

While by no means unbiased or scientific, a quick social media poll shows large developer demand for this:

* [X poll](https://x.com/LeaVerou/status/1951457982916469028)
* [Mastodon poll](https://front-end.social/@leaverou/114956605810091186)
* [Bsky discussion](https://bsky.app/profile/lea.verou.me/post/3lvezoaigws2c)

## Goals

* Provide website authors playback control over animated images, allowing them (at a minimum) to control playback and looping
* Authors should not need to create their own playback UI to provide this functionality to their users
* Authors should have full styling control over playback UI, either by customizing UA-generated UI, or recreating it, and ideally both.

### Non-goals

* Controlling animation of CSS images
* Retrofitting all images on a page to not animate with no HTML changes

These are valuable (and we plan to target them separately), but out of scope for this proposal.

## Proposed solution

The proposed solution is to support images (animated or not) in `<video>` elements.

Video elements already have all the desired functionality, so this adds no new API surface. They support a rich API for controlling playback, UA-generated controls, and make it possible for authors to build their own controls. Proposals like [Pseudo elements for `<video>` content and controls](https://github.com/whatwg/html/issues/10507) will make it possible for authors to customize existing controls, without having to rebuild them, and proposals like [predefined play/pause invoker commands]() will make it easier to build custom playback UIs.

Architecturally, this seems appropriate, since animated images are essentially videos with no audio track. Their implementation details are different, but there is no difference in terms of user-facing result.

## Discussion

### Emulating current `<img>` behavior with `<video>`

To emulate the behavior of standard `<img>` elements, authors would need to specify quite a few attributes: `<video muted autoplay loop playsinline>` . However, given that the core use case is disabling autoplay and/or looping, in practice not all need to be specified.

Another issue is that there are a few `<img>` features that are not yet supported in  `<video>`:

* No `loading=lazy` though there is a proposal to add it which appears to have consensus https://github.com/whatwg/html/issues/10376
* No `fetchpriority` . `fetchpriority=high` is rarely desirable here, and `loading=lazy` can cover many of the use cases for `fetchpriority=low` .

### Is it web compatible?

Given that images in `<video>` are currently treated as an authoring mistake, we believe it’s unlikely that this could introduce significant web compat issues. `<video>` already has all the machinery to handle unknown formats.

### How to treat static images?

While at first it seems reasonable to restrict this functionality to animated images only, that would be worse for both authors and implementors:

* UAs would need to read a variable number of bytes to even determine whether some formats are animated (for example, in APNG reading the TBD chunk comes after variable length descriptive metadata)
* Authors would need to detect whether an image is animated and use different elements, which is nontrivial in most templating environments. By allowing static images as well, they can simply use `<video>` indiscriminately.

Static images could be treated as videos with one frame, and a predefined duration. Possible choices for that are:

* 0 (preferable, as it makes detection easier)
* 1 fps (for a given fps)
* NaN
* Infinity
* undefined

Ideally, controls would not be rendered for static images even when the controls attribute is present, but if that increases implementation complexity, they can be hidden in authorland.

### Would this encourage higher usage of inefficient video formats?

It is well known that gif is an inefficient animation format. APNG files are also extremely large. Would this exacerbate the problem by encouraging their use?

The vast majority of use cases are around user-generated content. Whether `<video>` supports images will not affect user behavior, as it’s an author-facing benefit. Additionally, users often upload gifs because they want the autoplay behavior. If websites were able to treat them the same as videos, users may be more encouraged to upload videos in the first place.

This feature will reduce server-side conversion to videos, but we think that the benefits of this not being required outweigh the disadvantages.

### What about synchronization?

Animated images used in CSS or `<img>` are currently synchronized. Animated images used in `<video>` would need to be treated as separate instances, as synchronization is directly at odds with playback control. However, the benefit of synchronization only matters on pages where there are multiple instances of the same animated image, which is not where animated images in `<video>` would typically be used.

## Sample code snippets

> [!NOTE]
> These are optimized for readability, and thus do not cover all edge cases.

No autoplay, show UI controls for playback:

```html
<video src="foo.gif" controls loop muted></video>
```

---

Start paused and play on hover and focus:

```html
<video src="foo.gif" loop muted tabindex="0"></video>
```

```js
video.addEventListener("pointerenter", e => e.target.play());
video.addEventListener("pointerleave", e => e.target.pause());
video.addEventListener("focus", e => e.target.play());
video.addEventListener("blur", e => e.target.play());
```

This could also be easily automated via a web component.

---

Autoplay unless (prefers-reduced-motion) is on:

```html
<video class="image" src="foo.gif" controls loop muted autoplay></video>
```

```js
if (!matchMedia("(prefers-reduced-motion)").matches) {
  for (let video of document.querySelectorAll("video.image[autoplay]") {
    video.autoplay = false; // needs to run before resource loads
  }
}
```

---

Play once when in view, then provide playback UI to replay (e.g. social media animated emoji):

```html
<video src="foo.gif" loop muted class="image"></video>
```

```js
const observer = new IntersectionObserver(entries => {
for (const {target, intersectionRatio} of entries) {
  if (intersectionRatio === 1) { // Fully visible
   target.play();
   target.controls = false;
   target.addEventListener("ended", e => e.target.controls = true);
  }
  else if (intersectionRatio === 0) { // Left the viewport
    target.pause();
    target.controls = false;
  }
 }
});

for (const video of document.querySelectorAll("video.image")) {
 observer.observe(video);
}
```

## Future improvements

This is a first step in addressing the most pressing existing pain points by making complex things possible. There is a lot to be done for making them easy:

* Images typically need much simpler player UI, which is also useful for certain types of videos as well. A new value for controls or an expansion of `controlslist` could support UA-generated simpler controls, rather than requiring authors to recreate them from scratch.
* CSS pseudo-classes could be introduced to target images separately, or target static images separately. These could be either duration based (e.g. `:min-duration() `) or for predefined things (e.g. `:static`)
* [Interest invokers](https://open-ui.org/components/interest-invokers.explainer/) could eliminate the need for JS to play/pause on hover/focus (and make the interaction more generalizable).
* Ideally, it should be possible to disable autoplay declaratively when `(prefers-reduced-motion)` is on, but that is orthogonal to this proposal, as it applies to regular videos too.

## Alternatives considered

### Making `<img>` and/or `<picture>` media elements

The obvious alternative is to retrofit `<img>` and/or `<picture>` to extend from `HTMLMediaElement` and adding appropriate attributes to them.

While this may be valuable to pursue independently at a later stage, it is a much larger scope addition.

First, there is the question of whether retrofitting `<img>` to inherit from a different superclass is web-compatible at all. How much user JS walks up the inheritance chain for HTMLImageElement instances?

Second, many of `HTMLMediaElement’s` opt-in features, are on by default in `<img>` with no way to opt-out:

|               | `<img>` | `<video>` |
|---------------|---------|---------|
| Autoplay?     | ✅ (no opt-out) | Opt-in (`autoplay` + `muted` attributes) |
| Loop?         | ✅ (no opt-out) | Opt-in (`loop` attribute) |
| Muted?        | ✅ (no opt-out) | Opt-in (`muted` attribute). |
| Playback UI   | ❌ | Opt-in (`controls` attribute) |
| Plays inline? | Always | Opt-in for mobile (`playsinline` attribute) |
| Lazy loading  | Opt-in (`loading=lazy`) | [Proposed](https://github.com/whatwg/html/issues/10376) |
| Synchronized? | ✅ (no opt-out) | ❌ |

While having different defaults is not an issue for the JS side of things, for the HTML part of the API, it would require adding new attributes (e.g. `noautoplay` , `noloop` etc), which would in turn need to be reflected in the `HTMLMediaElement` API.

### Controlling image animation in CSS

> (Florian Rivoal should probably write this)
