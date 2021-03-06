---
title: Client API | ldapjs
markdown2extras: tables
---

# ldapjs Client API

This document covers the ldapjs client API and assumes that you are familiar
with LDAP. If you're not, read the [guide](guide.html) first.

# Create a client

The code to create a new client looks like:

    var ldap = require('ldapjs');
    var client = ldap.createClient({
      url: 'ldap://127.0.0.1:1389'
    });

You can use `ldap://` or `ldaps://`; the latter would connect over SSL (note
that this will not use the LDAP TLS extended operation, but literally an SSL
connection to port 636, as in LDAP v2). The full set of options to create a
client is:

|Attribute      |Description                                                |
|---------------|-----------------------------------------------------------|
|url            |A valid LDAP URL (proto/host/port only)                    |
|socketPath     |Socket path if using AF\_UNIX sockets                      |
|log            |Bunyan logger instance (Default: built-in instance)        |
|timeout        |Milliseconds client should let operations live for before timing out (Default: Infinity)|
|connectTimeout |Milliseconds client should wait before timing out on TCP connections (Default: OS default)|
|tlsOptions     |Additional options passed to TLS connection layer when connecting via `ldaps://` (See: The TLS docs for node.js)|
|idleTimeout    |Milliseconds after last activity before client emits idle event|
|strictDN       |Force strict DN parsing for client methods (Default is true)|

## Connection management

As LDAP is a stateful protocol (as opposed to HTTP), having connections torn
down from underneath you is can be difficult to deal with.  Several mechanisms
have been provided to mitigate this trouble.


## Common patterns

The last two parameters in every API are `controls` and `callback`. `controls`
can be either a single instance of a `Control` or an array of `Control` objects.
You can, and probably will, omit this option.

Almost every operation has the callback form of `function(err, res)` where err
will be an instance of an `LDAPError` (you can use `instanceof` to switch).
You probably won't need to check the `res` parameter, but it's there if you do.




# bind
`bind(dn, password, controls, callback)`

Performs a bind operation against the LDAP server.

The bind API only allows LDAP 'simple' binds (equivalent to HTTP Basic
Authentication) for now. Note that all client APIs can optionally take an array
of `Control` objects. You probably don't need them though...

Example:

    client.bind('cn=root', 'secret', function(err) {
      assert.ifError(err);
    });

# add
`add(dn, entry, controls, callback)`

Performs an add operation against the LDAP server.

Allows you to add an entry (which is just a plain JS object), and as always,
controls are optional.

Example:

    var entry = {
      cn: 'foo',
      sn: 'bar',
      email: ['foo@bar.com', 'foo1@bar.com'],
      objectclass: 'fooPerson'
    };
    client.add('cn=foo, o=example', entry, function(err) {
      assert.ifError(err);
    });

# compare
`compare(dn, attribute, value, controls, callback)`

Performs an LDAP compare operation with the given attribute and value against
the entry referenced by dn.

Example:

    client.compare('cn=foo, o=example', 'sn', 'bar', function(err, matched) {
      assert.ifError(err);

      console.log('matched: ' + matched);
    });

# del
`del(dn, controls, callbak)`


Deletes an entry from the LDAP server.

Example:

    client.del('cn=foo, o=example', function(err) {
      assert.ifError(err);
    });

# exop
`exop(name, value, controls, callback)`

Performs an LDAP extended operation against an LDAP server. `name` is typically
going to be an OID (well, the RFC says it must be; however, ldapjs has no such
restriction).  `value` is completely arbitrary, and is whatever the exop says it
should be.

Example (performs an LDAP 'whois' extended op):

    client.exop('1.3.6.1.4.1.4203.1.11.3', function(err, value, res) {
      assert.ifError(err);

      console.log('whois: ' + value);
    });

# modify
`modify(name, changes, controls, callback)`

Performs an LDAP modify operation against the LDAP server.  This API requires
you to pass in a `Change` object, which is described below.  Note that you can
pass in a single `Change` or an array of `Change` objects.

Example:

    var change = new ldap.Change({
      operation: 'add',
      modification: {
        pets: ['cat', 'dog']
      }
    });

    client.modify('cn=foo, o=example', change, function(err) {
      assert.ifError(err);
    });

## Change

A `Change` object maps to the LDAP protocol of a modify change, and requires you
to set the `operation` and `modification`.  The `operation` is a string, and
must be one of:

||replace||Replaces the attribute referenced in `modification`.  If the modification has no values, it is equivalent to a delete.||
||add||Adds the attribute value(s) referenced in `modification`.  The attribute may or may not already exist.||
||delete||Deletes the attribute (and all values) referenced in `modification`.||

`modification` is just a plain old JS object with the values you want.

# modifyDN
`modifyDN(dn, newDN, controls, callback)`

Performs an LDAP modifyDN (rename) operation against an entry in the LDAP
server.  A couple points with this client API:

* There is no ability to set "keep old dn."  It's always going to flag the old
dn to be purged.
* The client code will automatically figure out if the request is a "new
superior" request ("new superior" means move to a different part of the tree,
as opposed to just renaming the leaf).

Example:

    client.modifyDN('cn=foo, o=example', 'cn=bar', function(err) {
      assert.ifError(err);
    });

# search
`search(base, options, controls, callback)`

Performs a search operation against the LDAP server.

The search operation is more complex than the other operations, so this one
takes an `options` object for all the parameters.  However, ldapjs makes some
defaults for you so that if you pass nothing in, it's pretty much equivalent
to an HTTP GET operation (i.e., base search against the DN, filter set to
always match).

Like every other operation, `base` is a DN string.

Options can be a string representing a valid LDAP filter or an object
containing the following fields:

|Attribute  |Description                                        |
|-----------|---------------------------------------------------|
|scope      |One of `base`, `one`, or `sub`. Defaults to `base`.|
|filter     |A string version of an LDAP filter (see below), or a programatically constructed `Filter` object. Defaults to `(objectclass=*)`.|
|attributes |attributes to select and return (if these are set, the server will return *only* these attributes). Defaults to the empty set, which means all attributes. You can provide a string if you want a single attribute or an array of string for one or many.|
|attrsOnly  |boolean on whether you want the server to only return the names of the attributes, and not their values.  Borderline useless.  Defaults to false.|
|sizeLimit  |the maximum number of entries to return. Defaults to 0 (unlimited).|
|timeLimit  |the maximum amount of time the server should take in responding, in seconds. Defaults to 10.  Lots of servers will ignore this.|
|paged     |enable and/or configure automatic result paging|

Responses from the `search` method are an `EventEmitter` where you will get a
notification for each `searchEntry` that comes back from the server.  You will
additionally be able to listen for a `searchReference`, `error` and `end` event.
Note that the `error` event will only be for client/TCP errors, not LDAP error
codes like the other APIs.  You'll want to check the LDAP status code
(likely for `0`) on the `end` event to assert success.  LDAP search results
can give you a lot of status codes, such as time or size exceeded, busy,
inappropriate matching, etc., which is why this method doesn't try to wrap up
the code matching.

Example:

    var opts = {
      filter: '(&(l=Seattle)(email=*@foo.com))',
      scope: 'sub',
      attributes: ['dn', 'sn', 'cn']
    };

    client.search('o=example', opts, function(err, res) {
      assert.ifError(err);

      res.on('searchEntry', function(entry) {
        console.log('entry: ' + JSON.stringify(entry.object));
      });
      res.on('searchReference', function(referral) {
        console.log('referral: ' + referral.uris.join());
      });
      res.on('error', function(err) {
        console.error('error: ' + err.message);
      });
      res.on('end', function(result) {
        console.log('status: ' + result.status);
      });
    });

## Filter Strings

The easiest way to write search filters is to write them compliant with RFC2254,
which is "The string representation of LDAP search filters."  Note that
ldapjs doesn't support extensible matching, since it's one of those features
that almost nobody actually uses in practice.

Assuming you don't really want to read the RFC, search filters in LDAP are
basically are a "tree" of attribute/value assertions, with the tree specified
in prefix notation.  For example, let's start simple, and build up a complicated
filter.  The most basic filter is equality, so let's assume you want to search
for an attribute `email` with a value of `foo@bar.com`.  The syntax would be:

    (email=foo@bar.com)

ldapjs requires all filters to be surrounded by '()' blocks. Ok, that was easy.
Let's now assume that you want to find all records where the email is actually
just anything in the "@bar.com" domain and the location attribute is set to
Seattle:

    (&(email=*@bar.com)(l=Seattle))

Now our filter is actually three LDAP filters.  We have an `and` filter (single
amp `&`), an `equality` filter `(the l=Seattle)`, and a `substring` filter.
Substrings are wildcard filters. They use `*` as the wildcard. You can put more
than one wildcard for a given string. For example you could do `(email=*@*bar.com)`
to match any email of @bar.com or its subdomains like "example@foo.bar.com".

Now, let's say we also want to set our filter to include a
specification that either the employeeType *not* be a manager nor a secretary:

    (&(email=*@bar.com)(l=Seattle)(!(|(employeeType=manager)(employeeType=secretary))))

The `not` character is represented as a `!`, the `or` as a single pipe `|`.
It gets a little bit complicated, but it's actually quite powerful, and lets you
find almost anything you're looking for.

## Paging
Many LDAP server enforce size limits upon the returned result set (commonly
1000).  In order to retrieve results beyond this limit, a `PagedResultControl`
is passed between the client and server to iterate through the entire dataset.
While callers could choose to do this manually via the `controls` parameter to
`search()`, ldapjs has internal mechanisms to easily automate the process.  The
most simple way to use the paging automation is to set the `paged` option to
true when performing a search:

    var opts = {
      filter: '(objectclass=commonobject)',
      scope: 'sub',
      paged: true,
      sizeLimit: 200
    };
    client.search('o=largedir', opts, function(err, res) {
      assert.ifError(err);
      res.on('searchEntry', function(entry) {
        // do per-entry processing
      });
      res.on('page', function(result) {
        console.log('page end');
      });
      res.on('error', function(resErr) {
        assert.ifError(resErr);
      });
      res.on('end', function(result) {
        console.log('done ');
      });
    });

This will enable paging with a default page size of 199 (`sizeLimit` - 1) and
will output all of the resulting objects via the `searchEntry` event.  At the
end of each result during the operation, a `page` event will be emitted as
well (which includes the intermediate `searchResult` object).

For those wanting more precise control over the process, an object with several
parameters can be provided for the `paged` option.  The `pageSize` parameter
sets the size of result pages requested from the server.  If no value is
specified, it will fall back to the default (100 or `sizeLimit` - 1, to obey
the RFC).  The `pagePause` parameter allows back-pressure to be exerted on the
paged search operation by pausing  at the end of each page.  When enabled, a
callback function is passed as an additional parameter to `page` events.  The
client will wait to request the next page until that callback is executed.

Here is an example where both of those parameters are used:

    var queue = new MyWorkQueue(someSlowWorkFunction);
    var opts = {
      filter: '(objectclass=commonobject)',
      scope: 'sub',
      paged: {
        pageSize: 250,
        pagePause: true
      },
    };
    client.search('o=largerdir', opts, function(err, res) {
      assert.ifError(err);
      res.on('searchEntry', function(entry) {
        // Submit incoming objects to queue
        queue.push(entry);
      });
      res.on('page', function(result, cb) {
        // Allow the queue to flush before fetching next page
        queue.cbWhenFlushed(cb);
      });
      res.on('error', function(resErr) {
        assert.ifError(resErr);
      });
      res.on('end', function(result) {
        console.log('done');
      });
    });

# starttls
`starttls(options, controls, callback)`

Attempt to secure existing LDAP connection via STARTTLS.

Example:

    var opts = {
      ca: [fs.readFileSync('mycacert.pem')]
    };

    client.starttls(opts, function(err, res) {
      assert.ifError(err);

      // Client communication now TLS protected
    });


# unbind
`unbind(callback)`

Performs an unbind operation against the LDAP server.

Example:

    client.unbind(function(err) {
      assert.ifError(err);
    });
