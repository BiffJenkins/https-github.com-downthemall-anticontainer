# Writing Plugins
AntiContainer plugins are simple [JSON](http://en.wikipedia.org/wiki/JSON) structures describing one of three plugin types.

The _redirector_ and _resolver_ types are relatively easy to master.
_sandbox_ plugins are more powerful in what they can do, however require more knowledge about Javascript and require more time to execute.

You should prefer _redirector_ and _resolver_ if possible.

* To write a new plugin open _Preferences/Anti-Container_ and select **Create Plugin**.
* To install an existing plugin use **Install Plugin** from the same tab.

_Please note_: Do not manipulate the anticontainer_plugins folder yourself. If you do, your plugins might still work, but you cannot uninstall those plugins from the interface.

All existing plugins are available as flat files from [Github](http://github.com/nmaier/anticontainer/tree/master/plugins/)

# Plugin data
As noted, plugins are just _JSON_ structures, defining a common set properties.

## Generic Plugin Properties
Each plugin must provide the following information

* `type (String)` - Type of the plugin, either _redirector_, _resolver_ or _sandbox_
* `prefix (String)` - Human readable (host) prefix, which might used throughout the user interface
* `match (String)` - Regular expression which the URL must match in order to have the plugin applied to it.

## Optional Plugin Properties (Options)
Additionally you may also specify

* `priority (Number)` - Higher priority plugins have precedence. Use this, e.g. to strip-of url redirector services.
* `useServerName (Boolean)` - If the server later proposes a new file name it will taken into account. Per default only the generated name will be used.
* `sendInitialReferrer (Boolean)` - When set to `true` the request that fetches the container page will include a http referrer if available from the download job.
* `omitReferrer (Boolean)` - When set to `true` the referrer will NOT be changed to the container page URL and left blank.
* `keepReferrer (Boolean)` - When set to `true` the referrer will NOT be changed to the container page URL and the original download referrer is kept. <sup>(1.2.4)</sup>
* `generateName (String)` - If the server and URL do not provide a sane file name you may use `generateServerName` as a last resort. A UUID will be generated and the value of generateSeverName appended, i.e. `gSN = ".jpg"` will generate something like `ec8030f7-c20a-464f-9b0e-13a3a9e97384.jpg`
* `decode (Boolean)` - When set to `true` the fetched string will be URL-decoded before generating the new URL
* `static (Boolean)` - When set to `true` this indicates that once the final URL that is retrieved is static. If the download is re-started, for example, non-static URLs are resolved again, while static URLs will be kept from the previous run. `static` is implied for _redirector_ plugins, and defaults to `false` for other plugin types.
* `noFilter (Boolean)` - When set to `true` the plugin will not be considered for automatic filter generation.

Any other properties will be ignored.

## Meta-data
You may want to provide some information as well.

* `author (String)` - The name of the plugin author to be displayed in the UI.
* `ns (String)` - Namespace URI. The `prefix` and `ns` properties together form a unique id for the plugin. If omitted the default namespace is used. If two plugins share the same unique id, then the latter will replace (as in physically override) the former.
* `updateURL (String)` - A url to update the plugin from. This value will be set automatically if the plugin is installed from the web. <sup>(future)</sup>


# Redirectors

A _redirector_ is the most simple plugin. It takes an URL and returns a modified URL without loading any pages.

## Required Properties

* `pattern (String)` - A regular expression used for the replacement
* `replacement (String)` - The replacement text

Have a look at mozilla's [replace() documentation](https://developer.mozilla.org/En/Core_JavaScript_1.5_Reference/Global_Objects/String/Replace).


## Redirector Example
```json
{
  "type": "redirector",
  "priority": 1,
  "prefix": "anonym.to",
  "match": "^http://(.*?)anonym\\.to/\\?",
  "pattern": "^http://(.*?)anonym\\.to/\\?",
  "replacement": ""
}
```


This will match anonym.to URLs simply replacing the first part of the url with nothing (effectivly removing it).

#  Resolvers

A _resolver_ is a plugin that will examine the source text of the original URL target (website) and use a regular expression to match a certain part (the URL) and return it. A builder expression will then be used to generate the new URL.

## Required Properties

* `finder (String)` - Regular expression to match a certain part of a site. Use grouping to later access the parts of the match only.
* `builder (String)` - A builder string that will assemble the new URL. See below.

See mozilla's [RegExp documentation](https://developer.mozilla.org/En/Core_JavaScript_1.5_Reference/Global_Objects/RegExp) for more information on regular expressions.

## Optional Properties

* `namer (String)` - A builder string that will be used as a suggestion for the file name. See below for builder strings. <sup>(1.2.3, DTA-3)</sup>

## Builders
Builders use a special syntax to assemble the final URL fragment.
There is no need that the result of an URL is fully qualified; you may also return relative URLs.

Beside normal string parts there are manipulators, which allow access to the numbered groups of the finder match.
A manipulator has the form:
`{func:param1,param2,...,paramN}`

Currently there are three manipulators:

1. *num* (default if `func` is omitted). Concatenates all match groups as given by parameters
    * `{1}, {num:1}`  becomes Group[1]
    * `{1,3,4}, {num:1,3,4}` becomes Group[1, 3 and 4] concatenated
2. *or*. Uses the first group as specified by parameters that is non-empty

    Example `{or:1,3}`:
    * Suppose Group is:  `{1:'', 2:'a', 3:'b'}`, then the result is: `b`, as Group[1] is empty
    * Suppose Group is: `{1:'z', 2:'a', 3:'b'}`, then the result is: `z`, as Group[1] is non-empty
3. *replace*. Replacements with a regular expression

    Example: `{replace:1,some((?=thing|when)),any$}`

    Assume Group 1 is: `"some something someone somewhen"`

    Then the result is: `"some anything someone anywhen"`


## Resolver Example
```json
{
	"type": "resolver",
	"prefix": "bildercache.de",
	"match": "^http://(?:[\\w\\d]+\\.)?bildercache\\.de/anzeige",
	"finder": "src=\"(.*?bild/.*?)\" title=\"(.+?)\"",
	"builder": "{1}",
	"namer": "{2}"
}
```
Will look for the finder expression and construct a new URL consisting of the first group (i.e. .*?bild/.*? ) and use the second group to suggest a name.

# Sandbox plugins

If neither _redirectors_ nor _resolvers_ are powerful enough for your needs then you may resort to the _sandbox_ type plugins.
With them you're allowed to perform minimalistic operations within a Javascript sandbox.

* You're allowed to use normal Javascript.
* You're furthermore provided with the source text of the page the URL originally pointed to.
* You can make your own Request

The sandbox will keep the state within an instance (i.e. per download). API for keeping global state is planned.

You may implement one of the following functions:

* process 
* resolve

Additionally you may define `finder` and `builder` (s.a.) so that `defaultResolve` can be properly generated.

Please note that sandboxed plugins are still in development and still pretty limited. Also, correctly encoding your function isn't exactly a piece of cake. I prefer coding the function as plain javascript in some html and using the output of uneval().

## Sandbox contents
The sandbox offers the following non-standard methods/properties:

* Properties
    * `prefix (String)` - Plugin prefix
    * `sendInitialReferer (Boolean)` - See above
    * `responseText (String)` - The source text as detailed above.
    * `baseURL (String)` - The original URL.
* Functions
    * `process (void)` - Your process function or the default one.
    * `resolve (void)` - Your resolve function or the default one.
    * `defaultResolve (void)` - The default resolve function
    * `setURL(url: String, [nameSuggestion: String])` - (re)sets the URL
    * `finish(void)` - must be called when done. nameSuggestion<sup>(1.2.4, DTA-3)</sup> is optional.
 * Utility functions
   * `alert(msg)` - The "usual" alert. Use only for fast debugging
   * `log(msg)` - Logs msg using the standard DownThemAll logging facility (will appear in diagnostic logs).
   * `Request/XMLHttpRequest (void)` - Aliased constructors for web requests. A Request will behave a lot like XMLHttpRequest, however there are certain differences, most importantly:
      * overrideMimeType is omitted
      * responseXML is omitted
      * onreadyStateChange is omitted. Use onload/onerror instead.
 * `makeRequest(url: String, [load: Function(request), error: Function(request), context: Object])`  - Easy access to Request. You may specify load and error callbacks (both are recommended). The callbacks will be called in scope of context or global scope if omitted. responseText will always be set accordingly (before calling the callbacks if any).


### process
This function is called directly on the URL. Scope is the global scope of the Sandbox.
Implement this to provide a more sophisticated redirector or to do some URL fixups before resolving.

Functionally equivalent to the default implementation:
`makeRequest(baseURL, resolve, resolve);`

### resolve
This function is called on the source text. Scope is the global scope of the Sandbox.
Implement this to provide a more sophisticated resolver.

Functionally equivalent to the default implementation:
`defaultResolve();`


## Sandbox Examples
_redirector_ and _resolver_ plugins can be re-written as Sandbox plugins (but shouldn't, for performance reasons)

### Sandbox as Redirector
```json
{
	"type": "sandbox",
	"prefix": "google.com",
	"match": "^http://.*google\\.com/",
	"process": "setURL('http://example.com', 'myfile.ext'); finish();"
}
```

### Sandbox as Resolver (1)
This one is functionally equivalent to a normal resolver.
```json
{
	"type": "sandbox",
	"prefix": "bayimg.com",
	"match": "^http://(?:[\\w\\d]+\\.)?bayimg\\.com/(?!image)",
	"process": "makeRequest(baseURL, resolve, resolve);",
	"resolve": "defaultResolve();",
	"finder": "src=\"(.+?)\".*?id=\"mainImage\"",
	"builder": "{1}"
}
```

### Sandbox as Resolver (2)
Will compute the image url using the same function imagefap.com used to call. (Newlines added)

```json
{
	"type": "sandbox",
	"prefix": "imagefap.com",
	"match": "^http://.*imagefap\\.com/image\\.php",
	"resolve": "var lD = function (s) {
		var s1 = unescape(s.substr(0, s.length - 1));
		var t = \"\";
		for (i = 0; i < s1.length; i++) {t += String.fromCharCode(s1.charCodeAt(i) - s.substr(s.length - 1, 1));}
		return unescape(t);
	};
	var m = /lD\\('(.*?)'\\)/.exec(responseText);
	if (m && m.length >= 2) {
		setURL(lD(m[1]));
	}
	finish();"
}
```

# Generic Plugin Features
## Cleaners
Redirectors may also "clean" file names. This is handy as a lot of hosts mess with the original file name.
The cleaners are applied after the default clean, which deletes the first 3 or 5 characters if they are followed by an underliner ('_') or space. 
`43122_imagelab.jpg -> imagelab.jpg`

Simply specify a "cleaners" property containing an array of cleaner objects like so:
```json
"cleaners": [{
	"pattern": "[\\w\\d]{3}\\.(\\w+)$",
	"replacement": ".$1"
}]
```

This example defines a single cleaner object which will strip the last 3 characters before the file extension dot.
`image1ab.jpg -> image.jpg`

## Automatic DownThemAll Filter
AntiContainer will install a DownThemAll! filter with the name "AntiContainer". It is enabled by default.

The matching pattern of this filter will be regenerated on application startup accordingly to the installed filters that do not have `noFilter = false` specified.
When you add new plugins the AntiContainer filter is not regenerated instantly. Opening the DownThemAll! manager window or DownThemAll! preferences window should trigger an update, as does restarting the application (e.g. Firefox), however.

# Contributing your plugins to AntiContainer
So you've written a plugin that other AntiContainer users might find useful as well?
Then please don't hesitate to post a pull request or file an issue.
