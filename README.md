# Element Timing: Explainer

The **Element Timing** API enables monitoring when large or developer-specified image elements or groups of text nodes are displayed on screen.


### Objectives

1.  Inform developers when specific 'hero' elements are first displayed on the screen. We support the following image elements: \<img\>, \<image\> inside \<svg\>, and background-images. We also support observing certain groups of text nodes. Web developers can better understand which are the critical images and text of their sites, so after annotating them the browser can provide timing information about those.
1.  Enable analytics providers to measure display time of key images or text, without explicit opt in from web developers. In many cases it's not feasible for developers to modify their HTML just to get better performance insights, so it's important to provide basic information even for websites that cannot annotate their elements.


### How do we register elements for observation?

There are two ways an element can be registered for observation. The first way is explicitly via the `elementtiming` HTML attribute. Having this attribute set will signal the browser to expose render timing information for this element (its image and/or text). It should be noted that setting the `elementtiming` attribute does not work retroactively: once an element has loaded and is rendered, setting the attribute will have no effect. Thus, it is strongly encouraged to set the attribute before the element is added to the document (in HTML, or if set on Javascript, before adding it to the document). Having the attribute implies that:

* If the element is an image, there will be an entry for the image.
* If the element is affected by (possibly multiple) background images, there will be one entry for each of those background images.
* If the element is associated to at least a text node and it is implicitly registered for observation, there will be one entry for the associated text nodes.

The second way is implicitly: when the element content takes a large portion of the viewport when it is first displayed. That is, when an image (which could also be a background-image) occupies a large portion of the viewport, then an entry is dispatched for it. Similarly, when the area occupied by the text nodes associated to an element is large, then an entry is dispatched for that group of text. We register a subset of images and text by default to allow RUM analytics providers to gather information without having to request HTML changes from sites.

TODO: fix implicit registration.

### What information is exposed?

A `PerformanceElementTiming` entry has the following attributes:
* `name`: the initial URL for the resource request if the entry is for an image, or the first characters of the text if it is text.
* `entryType`: it will always be the string "element".
* `startTime`: the rendering timestamp.
* `duration`: it will always be set to 0.
* `intersectionRect`: the display rectangle of the image or text content within the viewport.
* `responseEnd`: if the entry is for an image, the timestamp of when the last byte of the resource response was received, same as ResourceTiming's [responseEnd](https://w3c.github.io/resource-timing/#dom-performanceresourcetiming-responseend). Otherwise, 0.
* `identifier`: the value of the `elementtiming` attribute of the element.
* `naturalWidth`: the [intrinsic](https://drafts.csswg.org/css2/conform.html#intrinsic) width of the image. It matches with the corresponding DOM [attribute](https://html.spec.whatwg.org/multipage/embedded-content.html#dom-img-naturalwidth) for img. 0 for text.
* `naturalHeight`: the [intrinsic](https://drafts.csswg.org/css2/conform.html#intrinsic) height of the image. It matches with the corresponding DOM [attribute](https://html.spec.whatwg.org/multipage/embedded-content.html#dom-img-naturalheight) for img. 0 for text.
* `id`: the element's ID.
* `element`: points to the element. This will be "null" if the element is [disconnected](https://dom.spec.whatwg.org/#connected).

Note: for background images, the element is the one being affected by the background image style.

Sample code:

```
<img src="my_image.jpg" elementtiming="foobar">

const observer = new PerformanceObserver((list) => {
  let perfEntries = list.getEntries().forEach(function(entry) {
      // Send the information to analytics, or in this case just log it to console.
      // |entry.startTime| contains the timestamp of when the image is displayed.
      console.log("My image took " + entry.startTime + " to render!");
   });
});
observer.observe({entryTypes: ['element']});
```

#### Origin restrictions

Allowing third-party origins to measure the time an arbitrary image resource takes to render could expose certain private content such as whether a user is logged into a website. Therefore, for privacy and security reasons, only rendering timing for entries corresponding to resources that pass the [timing allow check](https://w3c.github.io/resource-timing/#dfn-timing-allow-check) are exposed.

However, to enable a more holistic picture, the rest of the information is exposed for arbitrary images. That is, its `startTime` will be 0 but the other attributes will be set as described above.

### Questions

#### What about invisible or occluded elements?

The entry creation might be affected by _visibility_: for instance, elements are not exposed if the style visibility is set to none, or the opacity is 0. However, occlusion will not affect entry creation: an entry is seen if the element is there, but hidden by a full-screen pop-up on the page.

