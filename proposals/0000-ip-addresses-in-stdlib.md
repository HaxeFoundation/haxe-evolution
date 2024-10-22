# Better support for IP addresses in the Haxe standard library

* Proposal: [HXP-0000](0000-ip-addresses-in-stdlib.md)
* Author: [Frixuu](https://github.com/Frixuu)

## Introduction

The networking-related APIs in the Haxe standard library currently differ significantly
from those found in other languages such as C#, Java, or Rust.
This proposal introduces changes to `sys.net` to align it with contemporary networking APIs
while addressing several limitations in its current implementation.

## Motivation

### IPv6 support

A significant limitation of the standard library
is its lack of cross-platform IPv6 support.
IPv6 was drafted in the late 1990s,
gained widespread operating system
and programming language support in the early 2000s,
and was integrated into the Domain Name System in 2008.
According to Cloudflare, IPv6 accounted for approximately
36% of global HTTP traffic by the end of 2023[^cf].

This limitation has been documented since 2014[^issue1]
and regularly raised since then[^issue2][^issue3].
While the C++ target received preliminary IPv6 support in Haxe 3.4.0 (January 2017),
this implementation remains undocumented
and has proven insufficient for many users[^issue4].

### Type system integration

Notably, despite Haxe's robust type system,
IPv4 addresses lack proper type representation through either a class or an abstract type.
As of version 4.3.6, the official API documentation indicates
that IP addresses are stored as plain `Int` values,
with no specification of endianness[^api-host-ip].

### DNS resolution limitations

Beyond IPv6 support, the current implementation makes incorrect assumptions about DNS resolution.
A single domain can resolve to multiple DNS A records -
a capability demonstrated by haxe.org itself through its Cloudfront configuration.
(This can be verified using `dig haxe.org A` or `nslookup haxe.org`.)

Many programming languages provide standard library support
for resolving hostnames to multiple IP addresses:

* C has `getaddrinfo()`.
* C# has `System.Net.Dns.GetHostAddresses()`.
* Go has `net.LookupIP()`.
* Java has `java.net.InetAddress#getAllByName()`.
* PHP has `gethostbynamel()`.
* Rust implements the `std::net::ToSocketAddrs` trait for `str`.

This stands in stark contrast to Haxe's `sys.net.Host` only resolving and storing a single IP address.

### Platform differences

The current implementation varies across platforms.

* On Python, `sys.net.Host` neither resolves nor stores addresses[^host-python].
* On Lua, `sys.net.Host` resolves addresses, but `sys.net.Socket` does not utilize this resolution[^host-lua].

## Detailed design

### Additions

#### IPv4 addresses

A new class `sys.net.Ipv4Address` is introduced.

```haxe
package sys.net;

final class Ipv4Address {
  // ...
```

* This class is immutable.
* The implementation details are private.
  * The address may be internally stored as a little-endian `Int`, but it is not a necessity.
* The constructor of this class accepts four octets, in a big-endian order, as its arguments.
  
  ```haxe
  public function new(a:Int, b:Int, c:Int, d:Int);
  ```

  ```haxe
  var addr = new Ipv4Address(192, 168, 0, 1);
  ```

  * Arguments outside of the range [0; 255] should cause an exception to be thrown.

* A static method is introduced to fallibly parse an IPv4 address from a string.

  ```haxe
  public static function tryParse(s:String):Null<Ipv4Address>;
  ```

  ```haxe
  var addr = Ipv4Address.tryParse("192.168.0.1");
  if (addr != null) {
    // ...
  ```

  * It must be able to parse the addresses in a quad-dotted notation, e.g. `127.0.0.1`.
  * It may accept alternative notations, like `127.0.0.1/8`, `127.1`, `0x7F000001`.

* A `toString()` implementation is present.

  ```haxe
  public function toString():String;
  ```

  ```haxe
  trace(addr.toString()); // 192.168.0.1
  ```

  * It is guaranteed to return the address in a quad-dotted notation.
* There should also be some way to check the equality of two IPv4 addresses.

  ```haxe
  // Variant 1)
  public function equals(other:Ipv4Address):Bool;

  // Variant 2)
  public static function equals(lhs:Ipv4Address, rhs:Ipv4Address):Bool;
  ```

  * If `Ipv4Address` is implemented as an abstract over a class instead,
  it might be possible to overload the `==` operator.

* Additionally, it would be nice to have:
  * Constants for commonly used addresses:
  
  ```haxe
  public static final ANY = new Ipv4Address(0, 0, 0, 0);
  public static final LOCALHOST = new Ipv4Address(127, 0, 0, 1);
  public static final BROADCAST = new Ipv4Address(255, 255, 255, 255);
  ```

  * Utility methods to check the address type:

  ```haxe
  public function isLoopback():Bool;
  public function isLinkLocal():Bool;
  public function isMulticast():Bool;
  public function isPrivate():Bool;
  ```

#### IPv6 addresses

Similarly, a new class `sys.net.Ipv6Address` is introduced.

```haxe
package sys.net;

final class Ipv6Address {
  // ...
```

* This class is immutable.
* The implementation details are private.
  * The 16 bytes of the address may be internally stored as `haxe.io.Bytes`, `haxe.ds.Vector<Int>` or two `haxe.Int64`s.
  * The scope ID may be internally stored as an `Int` (if interface names are resolved eagerly), or as a custom enum like the following:
  
  ```haxe
  private enum ScopeId {
    ByIndex(index:Int);
    ByName(name:String);
  }
  ```

* The constructor of this class takes eight 16-bit segments, in a big-endian order, as its arguments. It also optionally accepts a scope ID.
  
  ```haxe
  public function new(a:Int, b:Int, c:Int, d:Int, e:Int, f:Int, g:Int, h:Int, ?scope: ScopeId) {
    scope ??= ByIndex(0);
    // ...
  ```

  ```haxe
  var addr1 = new Ipv6Address(0x2001, 0x0db8, 0x0, 0x0, 0x0, 0x0, 0x0, 0x1);
  var addr2 = new Ipv6Address(0xfe80, 0x1, 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, ByName("enp5s0"));
  ```

  * Arguments outside of the range [0; 65535] should cause an exception to be thrown.
* A static method is introduced to fallibly parse an IPv6 address from a string.

  ```haxe
  public static function tryParse(s:String):Null<Ipv6Address>;
  ```

  ```haxe
  var addr = Ipv6Address.tryParse("2001:db8::1");
  if (addr != null) {
    // ...
  ```

  * It must be able to parse the addresses in a colon-separated hex notation, e.g. `1:2:3:4:5:6:7:8`.
  * It must be able to parse the shortened versions of the addresses, e.g. `::1`.
  * It must be able to parse segments with leading zeros, e.g. `::1` and `::0001` are equivalent.
  * It must be able to parse `%scope_id` trailing the address.
  * It may be able to parse the hybrid notation of IPv4-mapped IPv6 addresses, e.g. `::ffff:127.0.0.1`.
* The `toString()` implementation is present.
  * It is guaranteed to return the address in a colon-separated hex notation.
  * It should return the shortened version of the address (e.g. `1:0:2:0:0:0:0:3` becomes `"1:0:2::3"`).
* There should also be some way to check the equality of two IPv6 addresses.
* Additionally, it would be nice to have:
  * Constants for commonly used addresses:
  
  ```haxe
  public static final ANY = new Ipv6Address(0, 0, 0, 0, 0, 0, 0, 0);
  public static final LOCALHOST = new Ipv6Address(0, 0, 0, 0, 0, 0, 0, 1);
  ```

  * Utility methods to check the address type:

  ```haxe
  public function isIpv4Mapped():Bool;
  public function isLoopback():Bool;
  public function isMulticast():Bool;
  public function isUniqueLocal():Bool;
  ```

#### IP addresses, in general

Sometimes it is useful to hold \*an\*
address, no matter what family it belongs to.

Different languages have different ways of expressing this.

* In C# and Go, there is only one type for both IPv4 and IPv6 addresses. `System.Net.IPAddress` and `net.IP`, respectively, store raw bytes of the address.
* In Java, both IPv4 and IPv6 classes have a common ancestor, `java.net.InetAddress`.
* In vanilla JS, the addresses are stored as strings.
* In Rust, there is an `std::net::IpAddr` enum with two constructors: `V4` and `V6`.
* In Swift, you can use either an `IPAddress` protocol or a `NWEndpoint.Host` enum (can also store hostnames).

Personally, I am drawn towards the Rust solution.

* The two families have different behavior (e.g. scopes) that might be difficult to generalize.
* There is no superclass/interface for the user code to accidentally extend.
* Algebraic data types are still underused and are worth promoting as a language feature.

An enum like this would probably look similar to:

```haxe
enum IpAddress {
  V4(addr:Ipv4Address);
  V6(addr:Ipv6Address);
}
```

However, it is currently impossible to define custom methods on enums.
This would be useful for many operations, including parsing and stringifying:

```haxe
public static function tryParse(s:String):Null<IpAddress> {

  final ipv4 = Ipv4Address.tryParse(s);
  if (ipv4 != null) {
    return V4(ipv4);
  }

  final ipv6 = Ipv6Address.tryParse(s);
  if (ipv6 != null) {
    return V6(ipv6);
  }

  return null;
}
```

```haxe
public function toString():String {
  return switch (this) {
    case V4(addr):
      addr.toString();
    case V6(addr):
      addr.toString();
  };
}
```

One way to sidestep this is with `@:using` a tool class:

```haxe
package sys.net;

@:using(sys.net.IpAddress.IpAddressTools)
enum IpAddress {
  V4(addr:Ipv4Address);
  V6(addr:Ipv6Address);
}

final class IpAddressTools {
  public static function tryParse(_:Enum<IpAddress>, s:String):Null<IpAddress> {
    // ...
```

### Changes

#### Host entries

The current implementation of `sys.net.Host` supports storage of a single IPv4 address.
This update expands its capabilities through the addition of an `addresses` field:

```haxe
public var addresses(default, null):Array<IpAddress>;
```

Note: The exact access modifiers, type implementation, and reference/copy semantics remain under discussion.

#### Sockets

The `sys.net.Socket` implementation receives updates to its `bind()` and `connect()` methods
to support multiple addresses.
These changes apply consistently across all platforms:

* When `bind()` fails to listen on the first stored address,
it will now try binding to the other ones.
It will exit early on the first success.
It will only error when all addresses have been exhausted.
* When `connect()` fails to connect to the first stored socket address,
it will now try connecting to the other ones.
It will exit early on the first success.
It will only error when all addresses have been exhausted.

#### Address

The `sys.net.Address` type requires an update
to accommodate the new `IpAddress` type implementation:

```diff
-public var host:Int;
+public var host:IpAddress;
```

## Impact on existing code

The change to `sys.net.Host` breaks ABI.
To maintain API compatibility, I suggest the following non-physical property:

```haxe
@:noDoc
@:noCompletion
@:deprecated("Use the `addresses` field instead for better typing and IPv6 support")
public var ip(get, set):Int;

private function get_ip():Int {
  for (addr in this.addresses) {
    switch (addr) {
      case V4(ip):
        return ip.asLittleEndianInt();
      case _:
    }
  }
  throw new UnsupportedFamilyException("This host does not have any associated IPv4 address");
}

private function set_ip(value:Int):Int {
  this.addresses = [V4(IPv4Address.fromLittleEndianInt(value))];
  return value;
}
```

The `sys.net.Address` modification affects both ABI and API.
One way to alleviate the API change is to alias the stored address
to a different field name, e.g. `host` to `ip`.

## Drawbacks

I strongly believe these proposed changes represent necessary complexity.

* This complexity cannot be abstracted further,
as it reflects the inherent nature of modern networking protocols.
* Only networking-related code will need to be updated.
Non-networking code remains unaffected.

However, as with any change, some pain points are to be expected:

* Platform-specific implementations will need maintenance across multiple targets.
* Additional test coverage and documentation will be required.

## Alternatives

### IPv6 address scope implementation

The implementation of scoped IPv6 addresses
presents two primary approaches based on existing language implementations:

1. Core address type integration ([chosen above](#ipv6-addresses)).
   * Implemented by: C#, Java, Python, Ruby.
   * Scope is integrated into the base address type.
   * Benefits:
     * Maintains existing library type signatures. (ie. `Socket.bind/connect/peer`)
     * Simplifies parsing and string representation.

2. Socket address type integration.
   * Implemented by: C, Rust.
   * Scope included with socket address, alongside port and flow information.
   * Benefits:
     * Provides cleaner comparison and validation logic.
     * More consistent with the protocol specification.

### Enum-based address implementation

As mentioned previously, it would also be possible
to avoid using an enum for this purpose altogether,
opting for a class- or interface-based hierarchy.

However, if in the end we were to represent an arbitrary IP address as an enum,
we could also define our type as an abstract over that enum:

```haxe
enum IpAddressImpl {
  V4(addr:Ipv4Address);
  V6(addr:Ipv6Address);
}

abstract IpAddress(IpAddressImpl) from IpAddressImpl to IpAddressImpl {
  // ...
```

Advantages of such an implementation:

* It supports implicit casting from `Ipv4Address` and `Ipv6Address`.
* It enables custom equality operator implementation (`@:op(A == B)`).

This approach also has some drawbacks:

* It requires `public` implementation enum access.
* Multiple public types add cognitive overhead.
* It complicates enum constructor documentation discovery.

## Opening possibilities

### Other addressing schemes

IPv6's position as the current standard does not seem at risk.

### `Socket` class design

The introduction of abstract classes in Haxe 4.2 presents an opportunity to reconsider the inheritance structure of socket types:

* `UdpSocket` could be decoupled from `Socket`.
* `Socket` could be renamed to `TcpSocket` to better reflect its purpose.
* A new abstract base class could provide common socket functionality.
  * This could be useful for future socket families, such as Unix, Bluetooth, or AX.25,
that do not rely on IP addressing.

## Unresolved questions

### Evolution strategy

This proposal prioritizes minimal breaking changes to existing APIs. However, this conservative approach warrants discussion:

1. Does minimizing breaking changes provide the best long-term architecture for the standard library?
2. Should we consider a more comprehensive redesign to better serve future networking needs?

### Address type architecture

The current `Address` type requires architectural consideration.

Options include:

1. Renaming to `IpSocketAddress` to clarify its purpose, allowing for future `SocketAddress` variants.
2. Complete removal with corresponding adjustments to `UdpSocket`.

[^cf]: Carlos Rodrigues, *Using DNS to estimate the worldwide state of IPv6 adoption,* December 14, 2023, <https://blog.cloudflare.com/ipv6-from-dns-pov/#ipv6-adoption-on-the-client-side-from-http>
[^issue1]: *sys.net.Host not supports IPv6,* June 9, 2014, <https://github.com/HaxeFoundation/haxe/issues/3122>.
[^issue2]: *No IPv6 support for sys.net.Address\UdpSocket?,* October 29, 2015, <https://github.com/HaxeFoundation/haxe/issues/4613>.
[^issue3]: *Cannot send UDP packet to an ipv6 host,* July 23, 2018, <https://github.com/HaxeFoundation/haxe/issues/7297>.
[^issue4]: *change ipv6 first,ipv4 second.* March 29, 2017, <https://github.com/HaxeFoundation/haxe/pull/6142>
[^api-host-ip]: <https://api.haxe.org/v/4.3.6/sys/net/Host.html#ip>
[^host-python]: <https://github.com/HaxeFoundation/haxe/blob/4.3_bugfix/std/python/_std/sys/net/Host.hx>
[^host-lua]: <https://github.com/HaxeFoundation/haxe/blob/4.3_bugfix/std/lua/_std/sys/net/Socket.hx#L59>
