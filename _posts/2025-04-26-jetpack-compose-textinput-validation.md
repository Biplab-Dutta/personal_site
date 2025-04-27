---
author: Biplab
title: Effective TextField validation in Jetpack Compose using Kotlin Flows and Coroutines
date: 2025-04-26 3:00:00 +545
categories: [Jetpack Compose]
tags: [android, jetpack compose, kotlin, flow, coroutine, viewmodel]
image:
  path: https://raw.githubusercontent.com/Biplab-Dutta/personal_site/main/preview_images/jetpack-compose-form-validation.png
  alt: Preview Image Credit - ChatGPT
---

Implementing text field validation in a Jetpack Compose app can be tricky. Launching suspending function calls to apply validation as soon as the text input updates, can lead to unexpected behaviour. There is a great [documentation][documentation] available from the jetpack compose team regarding the same topic which makes use of [MutableState][MutableState] to represent text input. My article will make use of [StateFlow][StateFlow] which is generally discouraged to use for text inputs unless you really know how couroutine and observing a flow works in Kotlin. We will see how we can reactively update our state and perform an email validation.

> _To follow along, clone the starter project in your local system. Checkout the [init][starter_project] branch for the initial setup._
{: .prompt-info }

To begin, create a package under `textinputvalidation` as `text_input`. Create 3 kotlin files inside `text_input` -- `TextInputScreen.kt`, `TextInputState.kt` and `TextInputViewModel`.

The `TextInputScreen.kt` file contains the UI code.
The `TextInputState.kt` file contains the class the represents our state.
The `TextInputViewModel` contains our state transformation logic.

Let's begin with our state class. Paste the following code in the `TextInputState.kt`.

```kotlin
data class TextInputState(
    val text: String = "",
    val hasError: Boolean = false,
)
```

It's a simple class that represents our state. The `text` property represents the text entered in our TextField composable whereas the `hasError` property denotes if there's any error after some validation logic has been performed.

Next, we can work on our view model.

```kotlin
class TextInputViewModel : ViewModel() {
    private val _state = MutableStateFlow(TextInputState())
    val state = _state.asStateFlow()

    fun onInputChanged(input: String) {
        _state.update {
            it.copy(text = input)
        }
    }
}
```

Here, we have a state variable that represents our textfield state. And as mentioned earlier, we are using `StateFlow` to hold our state information instead of `MutableState`.
The `onInputChanged` is a method that is triggered whenever user updates the text in the text field.
For a simple TextField implementation, this code sample works just fine. But with this code, we don't have validation setup. For this article, we want the validation to kick in as soon as the user begins typing something in the TextField.

The validation logic in our case will be a simple pattern check for an email.

```kotlin
  private fun validateInput(input: String): Boolean {
    return Patterns.EMAIL_ADDRESS.matcher(input).matches()
  }
```

Now, where should we call our `validateInput` method? It's tempting to think that since we want validation to be performed as soon as the user updates a text in the textfield, we should keep it inside of out `onInputChanged` method. But doing so comes with an issue.

The validation logic can be a suspending function which requires the result to come from an API of some sort. Such suspending function can take some time to compute. As a result of which, synchronization issue is occurred.

To read more about the synchronization issue, checkout this amazing [blog][blog].

Coming back to the solution. If we can't place our validation logic in the `onInputChanged` method, where should we actually do it? The answer to that is simple. We first create a separate method as `observeTextInput`. This method listens for every text input and performs the necessary validation.

```kotlin
  @OptIn(ExperimentalCoroutinesApi::class)
  private fun observeTextInput() {
      state
          .mapLatest { it.text }
          .drop(1)
          .onEach {
              val isInputValid = validateInput(it)
              _state.update {
                  it.copy(hasError = !isInputValid)
              }
          }.launchIn(viewModelScope)
  }
```
First, since we are only interested in the `text` property of our state class to listen the value from, we get that using `state.mapLatest { it.text }`. Then we ignore the first emission of our state using `drop`. I will tell you later why this is necessary. Then to run the validation logic after each text input change, we use the `onEach` lambda. Finally, the `launchIn` is a terminal flow operator the helps to run the flow collection in a `viewModelScope`.

Now with all that done, where should we call the `observeTextInput` method? For our usecase, we want to start listening to the text change as soon as the screen is rendered.

Our state is a flow which needs to be collected before operating on it. Flow collection for a state happens generally inside of our composables because that's where we need our app state. So, if we could write our code in such a way that says, as soon as our `TextInputState` is collected, we kickin the text change observation logic for validation, our work would be done.

Let's update the `TextInputState`.

```kotlin
class TextInputViewModel : ViewModel() {
    private val _state = MutableStateFlow(TextInputState())
    val state = _state
        .onStart { observeTextInput() } // The onStart is triggered as soon as the flow that's calling it (_state) is collected.
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5000),
            TextInputState()
        )

    fun onInputChanged(input: String) {
        _state.update {
            it.copy(text = input)
        }
    }

    @OptIn(ExperimentalCoroutinesApi::class)
    private fun observeTextInput() {
        state
            .mapLatest { it.text }
            .drop(1)
            .onEach {
                val isInputValid = validateInput(it)
                _state.update {
                    it.copy(hasError = !isInputValid)
                }
            }.launchIn(viewModelScope)
    }

    private fun validateInput(input: String): Boolean {
        return Patterns.EMAIL_ADDRESS.matcher(input).matches()
    }
}
```

The `onStart` method is what gets triggered whenever our state is collected from the UI. The `stateIn` method helps in converting our cold Flow into a hot StateFlow that is required by the UI.

The main idea here is to not run any validation logic direcly from the event handler (`onInputChanged`). Rather, create a separate observer and have it executed whenever the screen is created.

There is also a debate in the android community on where we should execute certain logic as soon as the screen is composed. In this demo, I used `onStart` and `stateIn` in my viewmodel. Many would prefer to use the `init` block of the viewmodel. To know about why I didn't use the `init` block of the viewmodel, you can read the following articles by Jaewoong Enum.

  - [Loading Initial Data in LaunchedEffect vs. ViewModel][blog1]
  - [Loading Initial Data on Android Part 2: Clear All Your Doubts][blog2]

Now, we can work on our UI. Our UI would be a screen with a TextField in the center. Nothing too fansy. Inside the `TextInputScreen` file

```kotlin
@Composable
fun TextInputScreenRoot(
    modifier: Modifier = Modifier,
) {
    val textInputViewModel = viewModel<TextInputViewModel>()
    val state by textInputViewModel.state.collectAsStateWithLifecycle()
    TextInputScreen(
        modifier = modifier,
        state = state,
        onValueChange = textInputViewModel::onInputChanged,
    )

}

@Composable
private fun TextInputScreen(
    modifier: Modifier = Modifier,
    state: TextInputState,
    onValueChange: (String) -> Unit,
) {
    Box(
        modifier = modifier.fillMaxSize(),
        contentAlignment = Alignment.Center,
    ) {
        OutlinedTextField(
            modifier = Modifier
                .fillMaxWidth()
                .padding(horizontal = 20.dp),
            value = state.text,
            onValueChange = onValueChange,
            label = { Text("Email") },
            isError = state.hasError,
            keyboardOptions = KeyboardOptions(
                keyboardType = KeyboardType.Email
            ),
            singleLine = true,
            supportingText = {
                if (state.hasError && state.text.isEmpty()) {
                    Text("Enter an email")
                }

                if (state.hasError && state.text.isNotEmpty()) {
                    Text("Invalid email")
                }
            }
        )

    }
}
```
The `TextInputScreenRoot` composable collects the state from the viewmodel which in turn starts observing the textfield changes.

Now finally, update the `MainActivity.kt`.
```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            TextInputValidationTheme {
                Scaffold(
                    modifier = Modifier.fillMaxSize(),
                ) { innerPadding ->
                    TextInputScreenRoot(modifier = Modifier.padding(innerPadding))
                }
            }
        }
    }
}
```
The reason we used `drop(1)` in our `observeTextInput` in the viewmodel is so that we don't start displaying error message as soon as the screen is rendered i.e before the user interacts with the TextField.

Now, with all of our setup and code, we should be able to see our form validation in action without having to worry about any scenarios.

<img src="https://raw.githubusercontent.com/Biplab-Dutta/personal_site/refs/heads/main/blog_assets/form-validation-jetpack-compose.gif" alt="jetpack compose form validation demo" width=300 height="500"/>

I hope that this blog post was helpful for the readers in some ways. Like I stated, in most of the blogs that I found on the internet on this topic, I kept seeing implementation of `MutableState` APIs. The `StateFlow` flow API is easier to use and it can be used in building performant text input validation with proper knowledge of Kotlin Flows and coroutines.

You can find the [full source code][final-source-code] for this project on my Github in the main branch.

If you wish to see some more projects of mine, follow me on [GitHub][GitHub-Biplab]. I am also active on Twitter [@b_plab][Twitter-Biplab] where I tweet about Flutter and Android.

**My Socials:**

|[GitHub][GitHub-Biplab]|[LinkedIn][LinkedIn-Biplab]|[Twitter][Twitter-Biplab]|

Until next time, happy coding!!! 👨‍💻

— Biplab Dutta


<!-- Hyperlinks -->
[MutableState]: <https://developer.android.com/reference/kotlin/androidx/compose/runtime/MutableState>
[StateFlow]: <https://developer.android.com/kotlin/flow/stateflow-and-sharedflow#stateflow>
[documentation]: <https://developer.android.com/develop/ui/compose/quick-guides/content/validate-input>
[starter_project]: <https://github.com/Biplab-Dutta/Text-Input-Validation-Jetpack-Compose/tree/init>
[blog]: <https://medium.com/androiddevelopers/effective-state-management-for-textfield-in-compose-d6e5b070fbe5>
[blog1]:<https://proandroiddev.com/loading-initial-data-in-launchedeffect-vs-viewmodel-f1747c20ce62>
[blog2]: <https://proandroiddev.com/loading-initial-data-part-2-clear-all-your-doubts-0f621bfd06a0>
[Twitter-Biplab]: https://twitter.com/b_plab98
[GitHub-Biplab]: https://github.com/Biplab-Dutta/
[LinkedIn-Biplab]: https://www.linkedin.com/in/biplab-dutta-43774717a/
[final-source-code]: <https://github.com/Biplab-Dutta/Text-Input-Validation-Jetpack-Compose>
