---
title: Learning Some iOS as a Kotlin Developer
tags: Swift Kotlin iOS
key: learning-io-as-a-kotlin-dev
---

I would consider myself a professional Kotlin developer during the day, and this is just a collection of some notes I have taken away from the little I learned about iOS when I tried to implement an iOS app called [Koddit](https://github.com/plusmobileapps/koddit) with Kotlin Multiplatform. 

## Immutability and Mutability

* **Immutable** - the value assigned to a variable is constant and cannot be changed or *mutated*
* **Mutable** - value assigned to a variable can be changed by reassigning another value to it

| Type | Swift | Kotlin |
|---|---|---|
| Immutable | `let` | `val` |
| Mutable | `var` | `var`|

Best practice is to always make variables immutable by default and convert to mutable variables if you find the value needing to change. Immutable variables are very useful when working from multiple threads as it will ensure values don't switch out from under you as another thread could have altered that value.


# XCode Tips

## Storyboard

### Keyboard Shortcuts

|  Action |  Keys | Description  | 
|---|---|---|
| Access libraries  | Command + Shift + L  | open the libraries popup to easily access common views such as labels and buttons  | 
| Open file in new editor  |  Option + Shift | move the cursor to the right side of current editor to open in new editor. Once two editors are open, you can move the cursor into which ever windo to open the file in that window.   |  

### Outlets and Actions

**Outlets** - reference to view in code to make dynamic changes to at run time from code

**Action** - reference to a piece of code that will run when the user interacts with a control

To connect a storyboard view to a specific part of code, open the `ViewController` file in a separate editor. Then `control + click and drag` to the `ViewController`. Then in the submenu that pops up, you should have the option to select the type of connection you want to make (action or outlet). You can also assign it a label to be generated in code. Note there is a little circle that appears next to the connection, when it is filled this indicates that the variable is actually referencing something from the storyboard. If you delete the view, you will see that circle no longer be filled indicating that it is not referencing anything. 

![Connecting actions and outlets](/assets/images/actions-outlets.gif)