---
title: "Godot Web progress report #9: Godot Scripts <-> JavaScript Interface"
excerpt: "No need to \"eval\"! A whole new interface to interact with JavaScript from Godot Scripts in Web exports, and a new API to prompt the user to download a file generated by Godot."
categories: ["progress-report"]
author: Fabio Alessandrelli
image: /storage/app/uploads/public/60c/dfe/795/60cdfe7956fca023522866.png
date: 2021-06-30 16:20:00
---

Howdy Godotters!

It hasn't been long since the [last Web progress report](https://godotengine.org/article/godot-web-progress-report-8), but it's finally time for the blog entry you have probably been waiting for... time to talk about **integrating Godot with third-party JavaScript APIs on the Web**.

### Motivation

Sometimes, when exporting Godot for the Web, it might be necessary to interface with external JavaScript code. Things like third-party SDKs, libraries, or simply accessing browser features that are not directly exposed by Godot.

Historically, this has been done in Godot via the [`JavaScript.eval`](https://docs.godotengine.org/en/stable/classes/class_javascript.html#class-javascript-method-eval) method. This relied on the JavaScript [`eval()` function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval), which beside being dangerous when misused, is also [quite cumbersome to use](https://docs.godotengine.org/en/3.3/getting_started/workflow/export/exporting_for_web.html#calling-javascript-from-script).

For these reasons, a **new interface** has been developed. This new interface feels more natural in the context of Godot scripting (e.g. GDScript and C#).

### JavaScriptObject

A new `JavaScriptObject` class has been added that wraps around native JavaScript objects and allows to call JavaScript methods and retrieve object properties.

You can access a JavaScript object via two new methods in the [`JavaScript`](https://docs.godotengine.org/en/latest/classes/class_javascript.html) singleton:

* `JavaScript.get_interface()`: Retrieves and wraps around an object in the global scope by calling a new method.

```gdscript
extends Node

func _ready():
    # Retrieve the `window.console` object.
    var console = JavaScript.get_interface("console")
    # Call the `window.console.log()` method.
    console.log("test")
```

* `JavaScript.create_object()`: Create an object via the JavaScript `new` constructor.


```gdscript
extends Node

func _ready():
    # Call the JavaScript `new` operator on the `window.Array` object.
    # Passing 10 as argument to the constructor:
    # JS: `new Array(10);`
    var arr = JavaScript.create_object("Array", 10)
    # Set the first element of the JavaScript array to the number 42.
    arr[0] = 42
    # Call the `pop` function on the JavaScript array.
    arr.pop()
    # Print the value of the `length` property of the array (9 after the pop).
    print(arr.length)
```

As you can see, thanks to Godot's internal [Object](https://docs.godotengine.org/en/stable/classes/class_object.html) design, you can interact with JavaScript objects obtained that way like they were native Godot objects, calling their methods, and retrieving (or even setting) their properties.

Base types (int, floats, strings, booleans) are automatically converted (floats might lose precision when converted from JavaScript to Godot the first time). Anything else (i.e. objects, arrays, functions) are seen as `JavaScriptObject`s themselves.

If you wonder where the magic lies and don't mind digging into the C++ codebase, it was "just" a matter of creating a new [`Object` child class](https://github.com/godotengine/godot/blob/3.x/platform/javascript/javascript_singleton.cpp) and overriding the `_get`, `_set`, `setvar`, `getvar`, and `call` methods... plus a painful amount of [JavaScript glue code](https://github.com/godotengine/godot/blob/3.x/platform/javascript/js/libs/library_godot_javascript_singleton.js).

But there's more! Let's talk about...

### Callbacks

Calling JavaScript code from Godot is nice, but sometimes you need to call a Godot function from JavaScript instead.

This case is a bit more complicated. JavaScript relies on garbage collection, while Godot uses reference counting for memory management. This means you have to explicitly create callabacks (which are returned as `JavaScriptObject`s themselves) and you have to keep their reference.

Arguments passed by JavaScript to the callback will be passed as a single Godot `Array`.

But don't worry if this sounds a bit technical, it's not too hard.

Here is an example to set the JavaScript [`window.onbeforeunload`](https://developer.mozilla.org/en-US/docs/Web/API/WindowEventHandlers#properties) property which asks a user for confirmation when leaving the web page:

```gdscript
extends Node

# Here create a reference to the `_my_callback` function (below).
# This reference will be kept until the node is freed.
var _callback_ref = JavaScript.create_callback(self, "_my_callback")

func _ready():
    # Get the JavaScript `window` object.
    var window = JavaScript.get_interface("window")
    # Set the `window.onbeforeunload` DOM event listener.
    window.onbeforeunload = _callback_ref

func _my_callback(args):
    # Get the first argument (the DOM event in our case).
    var js_event = args[0]
    # Call preventDefault and set the `returnValue` property of the DOM event.
    js_event.preventDefault()
    js_event.returnValue = ''
```

Here is another example that asks the user for the [Notification permission](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API) and waits asynchronously to deliver a notification if the permission is granted:

```gdscript
extends Node

# Here create a reference to the `_on_permissions` function (below).
# This reference will be kept until the node is freed.
var _permission_callback = JavaScript.create_callback(self, "_on_permissions")

func _ready():
    # NOTE: This is done in `_ready` for semplicity, but SHOULD BE done in response
    # to user input instead (e.g. during `_input`, or `button_pressed` event, etc.),
    # otherwise it might not work.

    # Get the `window.Notification` JavaScript object.
    var notification = JavaScript.get_interface("Notification")
    # Call the `window.Notification.requestPermission` method which returns a JavaScript
    # Promise, and bind our callback to it.
    notification.requestPermission().then(_permission_callback)

func _on_permissions(args):
    # The first argument of this callback is the string "granted" if the permission is granted.
    var permission = args[0]
    if permission == "granted":
        print("Permission granted, sending notification.")
        # Create the notification: `new Notification("Hi there!")`
        JavaScript.create_object("Notification", "Hi there!")
    else:
        print("No notification permission.")
```

### But can I use library X?

You most likely can, and it shouldn't be too cumbersome. First, you have to include your library in the page. You can simply customize the `Head Include` during export (see below), or even [write your own template](https://docs.godotengine.org/en/stable/tutorials/platform/customizing_html5_shell.html).

In the example below, we customize the **Head Include** to add an external library ([axios](https://axios-http.com/)) from a content delivery network, and a second `<script>` tag to define our own custom function:

```html
<!-- Axios -->
<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
<!-- Custom function -->
<script>
function myFunc() {
    alert("My func!");
}
</script>
```
![HTML5 export head include setting](/storage/app/uploads/public/60c/ddb/7a5/60cddb7a5c331340661868.png)

We can then access both the library and the function from Godot, like we did in previous examples:

```gdscript
extends Node

# Here create a reference to the `_on_get` function (below).
# This reference will be kept until the node is freed.
var _callback = JavaScript.create_callback(self, "_on_get")

func _ready():
	# Get the `window` object, where globally defined functions are.
	var window = JavaScript.get_interface("window")
	# Call the JavaScript `myFunc` function defined in the custom HTML head.
	window.myFunc()
	# Get the `axios` library (loaded from a CDN in the custom HTML head).
	var axios = JavaScript.get_interface("axios")
	# Make a GET request to the current location, and receive the callback when done.
	axios.get(window.location.toString()).then(_callback)

func _on_get(args):
	OS.alert("On Get")
```

And one last treat...

### Downloading files

A lot of you asked for an easy way to "download files" (e.g. game saves) from the Godot HTML5 export to the user computer.

This has been done with `eval()` before, and can also be done with the new interface described above.

Given it has a very common use case though, we decided to expose this functionality to scripting via a dedicated `JavaScript.download_buffer()` function which lets you download any generated buffer.

Here is a minimal example on how to use it:

```gdscript
extends Node

func _ready():
    # Asks the user download a file called "hello.txt" whose content will be the string "Hello".
	JavaScript.download_buffer("Hello".to_utf8(), "hello.txt")
```

And here is a more complete example on how to download a previously saved file:

```gdscript
extends Node

# Open a file for reading and download it via the JavaScript singleton.
func _download_file(path):
	var file = File.new()
	if file.open(path, File.READ) != OK:
		push_error("Failed to load file")
		return
	# Get the file name.
	var fname = path.get_file()
	# Read the whole file to memory.
	var buffer = file.get_buffer(file.get_len())
	# Prompt the user to download the file (will have the same name as the input file).
	JavaScript.download_buffer(buffer, fname)

func _ready():
	# Create a temporary file.
	var config = ConfigFile.new()
	config.set_value("option", "one", false)
	config.save("/tmp/test.cfg")

	# Download it
	_download_file("/tmp/test.cfg")
```

### Future work

Working on the Godot HTML5 export for more than a year has been a great experience, and I love seeing the many developers releasing their Godot games and apps on the Web, even with the [limitations this platform still has](https://www.youtube.com/watch?v=RCUj6Rlq2mw). I like to think Godot can help revitalize the web games ecosystem.

While I will keep maintaining the HTML5 platform, and there will be some more news in the next months beside the usual bug fixing, it's time to get back at the other aspect of Godot that's been overlooked in the last year... Networking and Multiplayer! Those of you following GitHub development might have noticed something already, but there's much more to come.

So stay tuned! :)

### References

- [Godot <-> JavaScript interface](https://github.com/godotengine/godot/pull/487190) and [3.x version](https://github.com/godotengine/godot/pull/48691) ([original proposal](https://github.com/godotengine/godot-proposals/issues/1852))
- [Download API](https://github.com/godotengine/godot/pull/48881) and [3.x version](https://github.com/godotengine/godot/pull/48929)
