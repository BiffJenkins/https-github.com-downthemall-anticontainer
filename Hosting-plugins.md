# Hosting plugins

The most simple way for an author to publish her plugins is to send plugins around and let users install them manually.

However there is also a way to let users install plugins directly from your website, provided you can set the `Content-Type` of the plugin files to `application/x-anticontainer-plugin`.

Web installation furthermore has the added benefit that `updateURL` is set accordingly if the plugin doesn't yet define one.

## Apache example
You can achieve this in Apache for example by putting the following directive in your .htaccess file:

* `AddType application/x-anticontainer-plugin json`

## nginx example
In nginx you may use the types directive (within a location) to specify the correct mime type:

* `types { application/x-anticontainer-plugin json; }`

# Screenshot
The browser will then render the installable plugin like this:

![Web install in action](/nmaier/anticontainer/wiki/webinstall.png)