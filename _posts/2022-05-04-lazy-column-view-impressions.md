---
title: How to track view impressions in a Jetpack Compose Lazy Column
tags: Kotlin Android Coroutines StateFlow Jetpack-Compose
key: lazy-column-view-impressions
---

If you have ever shipped a feature with a scrolling list, a product manager will usually ask you to track when an item in that list is viewed by the user. With Jetpack Compose being somewhat new, I was curious how to solve this problem with respect to a `LazyColumn` so let's learn how to know the second the eyeballs see items as they scroll into view!

![](/assets/images/spongebob-eyes.gif)

<!--more-->

## UI

```kotlin
@Composable
fun ListView(
    people: List<Person>,
    onDeleteClicked: (Person) -> Unit,
    onItemViewed: (Person) -> Unit
) {
    val lazyListState = rememberLazyListState()
    LazyColumn(state = lazyListState) {
        items(people.size, key = { people[it].key }) {
            val person = people[it]
            PersonRow(lazyListState, person, onDeleteClicked, onItemViewed)
        }
    }
}
```

```kotlin
@Composable
fun PersonRow(lazyListState: LazyListState, person: Person, onDeleteClicked: (Person) -> Unit, onItemViewed: (Person) -> Unit) {
    ItemImpression(key = person.key, lazyListState = lazyListState) {
        onItemViewed(person)
    }
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(24.dp)
    ) {
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text(text = person.name)

            IconButton(onClick = { onDeleteClicked(person) }) {
                Icon(Icons.Default.Delete, contentDescription = "Delete")
            }
        }
    }
}
```

```kotlin
@Composable
fun ItemImpression(key: Any, lazyListState: LazyListState, onItemViewed: () -> Unit) {
    val isItemWithKeyInView by remember {
        derivedStateOf {
            lazyListState.layoutInfo
                .visibleItemsInfo
                .any { it.key == key }
        }
    }
    if (isItemWithKeyInView) {
        LaunchedEffect(Unit) { onItemViewed() }
    }
}
```

## ViewModel

```kotlin
class MainViewModel : ViewModel() {

    private val tracker = AnalyticsTracker()

    private var _state: MutableStateFlow<List<Person>> = MutableStateFlow(people)
    val state: StateFlow<List<Person>> get() = _state

    fun onDeleteClicked(person: Person) {
        _state.value = _state.value.toMutableList().also { it.remove(person) }
    }

    fun onPersonViewed(person: Person) {
        tracker.onPersonViewed(person)
    }
}
```

## Analytics Tracker 

```kotlin
class AnalyticsTracker {

    private val recordedPeople = hashSetOf<String>()

    fun onPersonViewed(person: Person) {
        if (recordedPeople.contains(person.key)) return
        recordedPeople.add(person.key)
        Log.d("Item Impression", person.toString())
    }
}
```

## Resources 

* [Github Repository](https://github.com/plusmobileapps/lazycolumn-view-impressions)
* [Stack Overflow Answer](https://stackoverflow.com/a/70951303/7900721) - an efficient way to check when a specific LazyColumn item comes into view