## DBus design recommendations
### General
* What do you use as a default value for an optional object path?
  * DBus does not have the notion of a 'Nullable' type, use '/' the root path as the default value.
* What are the pitfalls of writable properties?
  * They don't have consistent failure modes in client side bindings
     * Clients ignore errors on writes
     * Clients set values asynchronously, ignoring failures
  * Some DBus services ignore invalid input
* Read only properties are the best practice, use methods to change state
* Do not create a tree structure or object hierarchy to represent relationships/associations between objects, use object properties on objects to form associations
* When an object is removed and you emit the signal InferfacesRemoved, you provide the object path and an array of all interfaces the object supported.
* For InterfacesRemoved and for GetManagedObjects you need not include the standard interfaces, ref. http://lists.freedesktop.org/archives/dbus/2015-March/016626.html
* Always make sure new objects are available before you put the object path in a
signal, or return the object path from a method.
* Object paths are opaque and should not contain any information about the object itself.  It's merely a pointer to an object.  Avoid using properties of the object or anything a user could possible change which would effect the object path.


### Maintaining backwards compatibility
* In general you **cannot**
  * Remove a method
  * Remove a property
  * Change a method signature
  * Change a signal signature
  * Change a property return type
  * Behavioral changes
    * Which signals are emitted and the order in relation to method returns etc.

* Be extremely careful when a property returns an enumerated value or one of a possible set of values.  If you add a new one existing clients could potential break, so you you need to clearly state that values will be added over time.

* Do add new properties over time, never remove or change existing.
* To allow methods which have a large number of parameters or need the ability to change over time, the typical convention is to use a 'a{sv}' as an options argument to a method.  However, please use best judgment on when this is needed and not add it for every possible method just to be 'safe'.
   * To aide it it's use, document the the options available in the DBus interface XML in a comment.
* More
    * http://0pointer.de/blog/projects/versioning-dbus.html
    * https://developer.gnome.org/gio/stable/gdbus-codegen.html#gdbus-code-stability
    * http://dbus.freedesktop.org/doc/dbus-api-design.html
