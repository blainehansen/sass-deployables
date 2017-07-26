# sass-deployables

> A library for creating object-oriented sass components.

---

Deployables allow you to define a component with internal state properties, which you can then reference from the content of the component. Then you can define different versions and transformations of the component, with different values of those state properties, and the changes you make will cascade to all the references.

Here's a basic example.

```sass
.deployable-component
  // activates this selector as a deployable. only valid on a single simple selector
  $component: dy-deployable()
  // creates a new internal property, and sets the default to red
  $component: dy-define-state($component, color, red)

  // normal css attributes are of course fine
  width: 100%

  // uses the color internal property, outputting its value to border-color 
  $component: dy-output($component, border-color, color)

  &.primary
    // creates a new version, with blue as its state
    $component: dy-define-version($component, color, blue)

  &:hover
    // creates a new transform,
    // saying that every version should have a lightened hover state
    $component: dy-define-transform($component, lighten, 'this.color', 10%)

  // builds the deployable
  +dy-build($component)
```

This will output something equivalent to this:

```sass
.deployable-component
  border-color: red

  width: 100%

  &:hover
    border-color: lighten(red, 20%)

  &.primary
    border-color: blue

    &:hover
      border-color: lighten(blue, 20%)
```

See how the Deployable remembered the reference (`dy-output`), and output an adjusted version of it for `.primary`? And notice how the hover transformation was only defined once but was output for both versions?

This gets much more powerful when you have more references, versions, and transforms.

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  width: 100%

  $component: dy-output($component, color)
  .nested-block
    $component: dy-output($component, border-color, color)
    $component: dy-output-function($component, background-color, adjust-hue, 'this.color', 180)

  &.primary
    $component: dy-define-version($component, color, blue)
    
  &.success
    $component: dy-define-version($component, color, green)

  &:hover
    $component: dy-define-transform($component, lighten, 'this.color', 10%)

  +dy-build($component)
```
Compiles to something like:

```sass
.deployable-component
  color: red
  width: 100%

  .nested-block
    border-color: red
    background-color: adjust-hue(red, 180)

  &:hover
    color: lighten(red, 20%)

    .nested-block
      border-color: lighten(red, 20%)
      background-color: adjust-hue(lighten(red, 20%), 180)

  &.primary
    color: blue

    .nested-block
      border-color: blue
      background-color: adjust-hue(blue, 180)

    &:hover
      color: lighten(blue, 20%)

      .nested-block
        border-color: lighten(blue, 20%)
        background-color: adjust-hue(lighten(blue, 20%), 180)

  &.success
    color: green

    .nested-block
      border-color: green
      background-color: adjust-hue(green, 180)

    &:hover
      color: lighten(green, 20%)

      .nested-block
        border-color: lighten(green, 20%)
        background-color: adjust-hue(lighten(green, 20%), 180)
```

Using Deployables will help you cut down on a huge amount of redundancy.

## API

There are six functions to create Deployables, and one mixin to build them.

- `@function dy-deployable($parent-component: null)`
- `@function dy-define-state($component, $name, $value)`
- `@function dy-define-version($component, $name, $value)`
- `@function dy-define-transform($component, $name, $func, $func-args...)`
- `@function dy-output($component, $css-prop, $var-name: null)`
- `@function dy-output-function($component, $css-prop, $func, $func-args...)`
- `@mixin dy-build($component)`

Thorough explanations are given below.

## Concepts

Deployables are made of a few core concepts:

- States. Internal variables the component has.
- Versions. Distinct types of the component. Each version should have at least one state variable that is different than all the others.
- Transforms. Changes that should be applied to the state variables no matter what they are.
- References, or the Content. The uses of the state variables, made in the content of the component.

Read on to understand each in more depth.


## States

Defining a new state variable is simple. Just call `dy-define-state` in the base of the component, in the "default" version.

```sass
.deployable
  // initialize
  $component: dy-deployable()
  // everything at the base of the component is the "default" version
  $component: dy-define-state($component, color, gray)
```

Now that variable can be referred to in the content, given different values in different versions, and changed in transforms.

**Note:** you can't call `dy-define-state` in versions, transforms, or nested blocks.

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  .nested-block
    // -> ERROR, invalid location to set deployable state
    $component: dy-define-state($component, color, green)
    $component: dy-output($component, color)

  &.primary
    // -> ERROR, can't call dy-define-state in version
    // use dy-define-version instead
    $component: dy-define-state($component, color, red)
```


## Versions

A Version is any different kind of the component that is distinct from all the others, and doesn't overlap with them.

A common example is different kinds of buttons:

```sass
.button
  $component: dy-deployable()

  // the default value
  $component: dy-define-state($component, color, black)

  $component: dy-output($component, color)
  $component: dy-output($component, border-color, color)
  $component: dy-output-function($component, background-color, lighten, 'this.color', 90%)

  &.primary
    $component: dy-define-version($component, color, blue)
    
  &.success
    $component: dy-define-version($component, color, green)

  &.danger
    $component: dy-define-version($component, color, red)

  +dy-build($component)
```

Each of those types (`.primary`, `.success`, `.danger`) can't coexist with the others. A button can't be both `.success` and `.danger`. That's what makes them Versions, their separateness.


### Interleaving Versions

Even though Versions shouldn't overlap, you can have different *kinds* of Versions changing different qualities. As long as they don't conflict with each other everything will work. A simple example is a button with different Versions for colors, and another set of Versions for sizes.

```sass
.button
  $component: dy-deployable()

  // states, outputs, and versions for coloring
  $component: dy-define-state($component, color, black)
  $component: dy-output($component, color)
  $component: dy-output($component, border-color, color)

  &.primary
    $component: dy-define-version($component, color, blue)
    
  &.success
    $component: dy-define-version($component, color, green)


  // states, outputs, and versions for sizing
  $component: dy-define-state($component, font-size, 16px)
  $component: dy-output($component, font-size)
  $component: dy-output($component, line-height, font-size * 1.5)

  &.big
    $component: dy-define-version($component, font-size, 20px)

  &.small
    $component: dy-define-version($component, font-size, 12px)

  +dy-build($component)
```

Now when you make buttons in your templates, you can use Versions for colors and for sizes together, like `.button.primary.big`. As long as these interleaved Versions don't operate on the same States, they'll work great.


### Version Selector Limitations

Right now, the rules for versions are pretty specific. A valid version is any selector that starts with `&`. Here are some examples of valid versions.

- Compounding selectors: `&#id`, `&.big`
- Pseudo-elements: `&::before`
- Attributes or psueudo-classes: `&:active`, `&[target=_blank]`
- Suffixes: `&-suffix`


<!-- ### Output Order

Deployables can't guarantee that 

Although it's not a good idea to have versions that could potentially conflict with each other, it's technically allowed by this library. 

no guarantee of output order.


Even though 

When different versions conflict with each other, the one defined most recently wins. So in the above example, if the element was both `.open` and `:first-child`, it would be green since `:first-child` was defined last.

You can create more specific versions to handle these kinds of situations.

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  $component: dy-output($component, color)

  &.open:first-child
    $component: dy-define-version($component, color, silver)

  &.open
    $component: dy-define-version($component, color, blue)

  &:first-child
    $component: dy-define-version($component, color, green)

  +dy-build($component)
``` -->


## Transforms

You can define Transforms on the component. These work very similarly to Versions, but instead of setting the state directly, you define a change to be made to it.

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  $component: dy-output($component, color)

  &.success
    $component: dy-define-version($component, color, green)

  &:hover
    $component: dy-define-transform(color, fade-out, 'this.color', 0.2)

  +dy-build($component)
```
```sass
.deployable-component
  color: red

  &:hover
    color: fade-out(red, 0.2)

  &.success
    color: green

    &:hover
      color: fade-out(green, 0.2)
```

Transforms will be applied to all versions, so you only have to declare them once.


### Compounding Transforms

Right now, each transform is only applied once to each version, and they aren't compounded on top of each other. This means that transforms are only applied one at a time.

Here's an example:

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  $component: dy-output($component, color)

  &:hover
    $component: dy-define-transform($component, color, fade-out, 'this.color', 0.2)

  &.open
    $component: dy-define-transform($component, color, adjust-hue, 'this.color', 180deg)

  +dy-build($component)
```
```sass
.deployable-component
  color: red

  // see how neither of these has the other one defined for it?
  // which one of these will be applied is dependent on css specificity
  &:hover
    color: fade-out(red, 0.2)

  &.open
    color: adjust-hue(red, 180deg)
```

Allowing transforms to compound on top of each other is a feature planned for the next version of this package.

For now, you can just create more specific transforms.

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  $component: dy-output($component, color)

  &:hover
    $component: dy-define-transform(color, fade-out, 'this.color', 0.2)

  &.open
    $component: dy-define-transform(color, adjust-hue, 'this.color', 180deg)

  &.open:hover
    $component: dy-define-transform(color, darken, 'this.color', 30%)

  +dy-build($component)
```
```sass
.deployable-component
  color: red

  &.open
    color: adjust-hue(red, 180)

  &:hover
    color: fade-out(red, 0.2)

  &.open:hover
    color: darken(red, 30)
```

### Mingling Versions and Transforms

It isn't allowed to mix versions and transforms in the same block.

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  $component: dy-output($component, color)

  &.success
    $component: dy-define-version($component, color, green)

    // -> ERROR, can't mingle versions and transforms
    $component: dy-define-transform(color, saturate, 'this.color', 10%)

  &:hover
    $component: dy-define-transform(color, fade-out, 'this.color', 0.2)
    
    // -> ERROR, can't mingle versions and transforms
    $component: dy-define-version($component, color, gray)

  +dy-build($component)
```



## Content References

In order to output the state values in a way that will be remembered and used for every Version, you can use the `dy-output` and `dy-output-function` functions. You can even call these functions in nested blocks, and they'll still work exactly as expected.

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  // has two forms
  // one that outputs to the css property with the same name
  $component: dy-output($component, color)
  // -> color: the value of color

  // and one that outputs to a named css property
  $component: dy-output($component, border-color, color)
  // -> border-color: the value of color


  // supply the css property to be output to,
  // the function name,
  // and the list of arguments,
  // using the format 'this.statevar' to refer to state properties
  $component: dy-output-function($component, background-color, adjust-hue, 'this.color', 180)


  .nested-block
    // every version will have a .nested-block with the appropriate background-color
    $component: dy-output($component, background-color, color)


  // ... versions and transforms ...

  +dy-build($component)
```


### Content Overrides

In different Versions and Transforms, you can override the content outputs, or add new ones.

```sass
.deployable-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, red)

  $component: dy-output($component, color)
  $component: dy-output($component, border-color, color)

  &.primary
    $component: dy-define-state($component, color, green)

    // this will override the normal "border-color: color" output
    $component: dy-output-function($component, border-color, lighten, 'this.color', 10%)

    // this will only output for this version
    $component: dy-output($component, background-color, color)
    

  +dy-build($component)
```
Compiles to something like:

```sass
.deployable-component
  color: red
  border-color: red

  &.primary
    color: green
    border-color: lighten(green, 10%)
    background-color: green
```


## Inheritance

You can create secondary Deployables that inherit the states, versions, and transforms of existing Deployables. Just make the component object available to other scopes, and pass it to `dy-deployable`. 

In order to also get the benefits of Sass `@extend`, call the `dy-extend` mixin. Fixing the inconvenience of having to call two different things in order to extend a component is one of the features planned for the next version.

```sass
.parent-component
  $component: dy-deployable()
  $component: dy-define-state($component, color, black)

  // here's a normal css attribute
  // which will be carried over by dy-extend
  width: 100%

  $component: dy-output($component, color)

  &:hover
    $component: dy-define-transform(color, fade-out, 'this.color', 0.2)

  +dy-build($component)

  // this is necessary to make the $component available to other scopes
  $parent: $component !global

.child-component
  $component: dy-deployable($parent)
  +dy-extend($parent)

  &.primary
    $component: dy-define-version($component, color, blue)

  &.success
    $component: dy-define-version($component, color, green)

  +dy-build($component)
```

This is just as useful with extend-only selectors. Just skip the `dy-build` call.

```sass
%hoverable
  $component: dy-deployable()

  &:hover
    $component: dy-define-transform(color, fade-out, 'this.color', 0.2)

  $hoverable: $component !global

.deployable-component
  $component: dy-deployable($hoverable)
  +dy-extend($hoverable)

  // ... etc ...

  +dy-build($component)
```

### Extendable Libraries

Since Deployables are essentially just maps stored in a variable like `$component`, they can be shared in libraries that would allow users to extend existing components. As long as the variables are made global and available, anyone can work with them.

```sass
// in a package called 'cool-buttons'
// 'main.sass'
%cool-btn
  $component: dy-deployable()

  $component: dy-define-state($component, color, black)
  $component: dy-output($component, color)
  $component: dy-output($component, border-color, color)

  &.primary
    $component: dy-define-version($component, color, blue)
    
  &.success
    $component: dy-define-version($component, color, green)

  $cool-btn-component: $component !global


// in user code
@import 'cool-buttons/main'
.extended-btn
  $component: dy-deployable($cool-btn-component)
  +dy-extend($cool-btn-component)

  // overriding an old Version
  &.primary
    $component: dy-define-version($component, color, cyan)

  // creating a new Version
  &.danger
    $component: dy-define-version($component, color, red)

  +dy-build($component)
```


## Future Plans

**Features**

- [ ] transform compounding
- [ ] hidden `$component` object saving, so the calls aren't so cluttered
- [ ] getting rid of `dy-extend` and calling `@extend` in `dy-deployable`
- [ ] more flexible version definitions, allowing the `&` to be in different places
- [ ] "related" versions, ones that are derived from a base value

**Administrivia**

- [ ] robust testing
- [ ] thorough errors and warnings


## Contributing

If you'd like to contribute, just fork the project and then make a pull request!