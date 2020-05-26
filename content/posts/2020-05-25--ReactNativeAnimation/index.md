---
title: React Native Animation
category: "React Native Animation"
cover: hummingbird.jpg
author: Anthony Humphreys
---

# Creating Animated Experiences in React Native

The following examples will all use variations of a simple UI element, a card, slowly adding features and interactions to extend the UX. The following examples only use React Native components and APIs, there are lots of UI and Animation libraries available which I'll touch upon [later](#Gotchas-and-Libraries).

## Simple Animations

For the simplest example, I created a simple card component using `<Animated.View>` and `<Text>` components from React Native.

### Creating a card

```JSX
<Animated.View
  style={[
    styles.card,
    {
      opacity: fadeAnim
    }
  ]}
>
  <Text style={styles.text}>React Native Animations</Text>
  <Text style={styles.text}>Some more text</Text>
</Animated.View>
```

Before looking at the animation, lets have a quick look at how this component is put together.

`<Animated.View>` is a component provided by React Native which is simply a `<View>` wrapped by `createAnimatedComponent`, which you can read about [here](https://reactnative.dev/docs/animated#createanimatedcomponent).

Styles are provided to the components using `StyleSheet` - a React Native API for creating style objects. I didn't want to use 3rd party libraries for this tutorial, and it is bad practice to use inline styles in react native for a number of performance and other reasons which have been pretty well covered elsewhere.

You'll notice an array being passed to the style prop. Normally if I was passing multiple styles I would use StyleSheets `compose` API, but this would rather defeat the object here, as I want to be able to pass a dynamic style while keeping the performance benefits and optimizations from StyleSheet - the only way to achieve this is by passing an array.

### Animating the card

For a simple introduction, I thought a fade-in would be a useful example as it is a fairly typical interaction animation.

To animate a single value, I need to initialize a variable, `fadeAnim` which will subsequently be used to drive the opacity of the card component.

```javascript
const fadeAnim = useRef(new Animated.Value(0)).current;
```

Next, I use a `useEffect` to trigger the animation.

```javascript
useEffect(() => {
  Animated.timing(fadeAnim, {
    toValue: 1,
    duration: 1500,
    useNativeDriver: true
  }).start();
}, [fadeAnim]);
```

This will cause the effect to run when fadeAnim is initialised on first render, fading the card into view.

In this example we are using a simple, linear transition, from 0 to 1, and that value is passed to the opacity prop. To make this look slightly more natural, you could user the `interpolate` [method](https://reactnative.dev/docs/animations#interpolation) - I won't be covering interpolation in this post, as it is a little out of scope and it doesn't add much to these examples.

## Moving on a bit

The next step is to look at how to trigger an animation programatically. In this case we will extend the previous example to include a fade out animation for the same card we faded in previously.

To achieve this, I added a `useCallback` to trigger the fade out animation.

```javascript
const fadeOut = useCallback(() => {
  Animated.timing(fadeAnim, {
    toValue: 0,
    duration: 1500,
    useNativeDriver: true
  }).start();
}, [fadeAnim]);
```

This `fadeOut` function is called from a `<Button>` which I added to the card.

```JSX
<Button color="#FFFFFF" title="Fade Out" onPress={fadeOut} />
```

Althought a pretty simplistic example, hopefully this demonstrates the UX patterns which you can achieve with relatively little code.

## Composing Animations

Next, we look at composing animations. For this example, we add a further animated value,

```javascript
const heightAnim = useRef(new Animated.Value(100)).current;
```

we also add some state, to reflect whether the card is `expanded` or not

```javascript
const [expanded, setExpanded] = useState(false);
```

the `heightAnim` variable is used to drive the height of the card, while the `expanded` state variable is used to show and hide some additional text, which only shows when the card is in the `expanded` state. This is not necessarily the best way to construct this UI, this is purely for demonstration purposes. In this instance, the `fadeAnim` variable is used to drive the opacity of the text, not the whole card.

```JSX
<Animated.View
  style={[
    styles.card,
    {
      height: heightAnim
    }
  ]}
>
  <Text style={styles.text}>React Native Animations</Text>
  <Button color="#FFFFFF" title={expanded ? "Close" : "Open"} onPress={expanded ? close : open} />
  {expanded && (
    <Animated.Text style={[styles.text, { opacity: fadeAnim }]}>
      Some more text, hidden by default
    </Animated.Text>
  )}
</Animated.View>
```

### Open - Expand the card and fade in the text

The following snippet calls `setExpanded` before using another `Animated` API, `Animated.parallel`. As the name suggests, this API allows you to execute multiple Animations in parallel, i.e. triggered at the same time.

```javascript
const open = useCallback(() => {
  setExpanded(true);
  Animated.parallel([
    Animated.spring(heightAnim, {
      toValue: 150,
      duration: 100,
      useNativeDriver: false
    }),
    Animated.timing(fadeAnim, {
      toValue: 1,
      duration: 1000,
      useNativeDriver: false
    })
  ]).start();
}, [fadeAnim, heightAnim]);
```

### Close - Hide the text and collapse the card

This next snippet utilises the same `Animated.parallel` API, but this time you'll notice the `.start()` call contains a callback. This is so that we can call `setExpanded`, setting the state value to false _only_ once the animations have completed, preventing unexpected UI behaviour.

```javascript
const close = useCallback(() => {
  Animated.parallel([
    Animated.timing(fadeAnim, {
      toValue: 0,
      duration: 100,
      useNativeDriver: false
    }),
    Animated.spring(heightAnim, {
      toValue: 100,
      duration: 1000,
      useNativeDriver: false
    })
  ]).start(() => setExpanded(false));
}, [fadeAnim, heightAnim]);
```

Try out the example and you'll see how this allows us to create more refined and quite natural interactions. Having the text 'snap' in and out of existence is quite unsatisfactory and indeed jarring to the end user.

## Layout Animation API

The layout animation API gets an honourable mention for simple layout animations, but I would personally ignore it and use the `Animated` API directly instead, as it gives you control over the interactions and allows richer experiences. There are some nuances to setting up the Layout Animation API across platforms discussed in the [documentation](https://reactnative.dev/docs/animations#layoutanimation-api), and I am not a fan of additional boilerplate.

## Accessibility Considerations

When designing and building User Interfaces, accessibility should always be at the front of your mind. If you have to 'make your interface accessible', I would argue it has been poorly designed and implemented. Accessibility should be given the same consideration as performance and other considerations.

You can detect this quite easily, thanks to React Native's `AccessibilityInfo.isReduceMotionEnabled`, [documented here](https://reactnative.dev/docs/accessibilityinfo#isreducemotionenabled). This returns a bool which you can easily use alongside your animation implementations to determine whether to dial down animations or omit them entirely.

## Gotchas and Libraries

There are a few aspects of React Native Animation that trip me up from time to time. The first one is `useNativeDriver`, this has a few benefits, as outlined in the [documentation](https://reactnative.dev/docs/animations#using-the-native-driver)

> By using the native driver, we send everything about the animation to native before starting the animation, allowing native code to perform the animation on the UI thread without having to go through the bridge on every frame. Once the animation has started, the JS thread can be blocked without affecting the animation.

The documentation outlines some caveats, and shows that not all scenarios are appropriate for `useNativeDriver`.

From a UX perspective, its really easy to get animation very wrong (looking at you most early 2000s websites...). There is a delicate balance where animation is concerned. I would highly recommend [the case for whimsy](https://www.youtube.com/watch?v=Z2d9rw9RwyE) - along with pretty much any of Joshua Comeau's talks for how to use animation creatively, but without it being offputting or obnoxious.

There are a few libraries out there, though I haven't had experience with them so it wouldn't be fair for me to pass comment on them here!

## Videos

I'm currently working on a series of short tutorials covering this content in a little more depth and talking through some animation examples. I'll update this post once they're ready. In the meantime, if you have any particular aspects of React Native animation that you would like me to cover, please get in touch [via Twitter](https://twitter.com/aphumphreys) or at [contact@anthonyhumphreys.co.uk](mailto:contact@anthonyhumphreys.co.uk?subject=React%20Native%20Animation%20Tutorial%20Request)
