Role-based access control is suitable for applications that limit access to
records based on the role of the user sending the request.

If your app is configured to use role-based access control, you need to define
what roles your app is needed. Instead of granting access to users on a per-user
basis, you grant accesses to users on a per-role basis. Each user of the same
role can access records that you grant access to that role.

In this guide, we are going to walk you through making a blogging application
in which many authors can contribute articles to the same blog while
others can post comments to the articles.

## Define roles for your app

During the bootstrapping phrase of your app, you should define the roles that
your app is going to need. Once the app is freezed into production mode, you
cannot grant access to roles you have not defined.

In the case of a simple blogging application, an application might need
an `webmaster` role, an `author` role and a `reader` role. Users with the `author`
role can post articles, while users with the `reader` role can post comments.

The role of an `webmaster` is to assign roles to existing users.

To tell Skygear Server you have these roles, you should define them during the
bootstrapping phrase:

```javascript
const Webmaster = skygear.Role.define('webmaster');
const Author = skygear.Role.define('author');
const Visitor = skygear.Role.define('visitor');
```

At this pont, the `webmaster` role is an ordinary role without special powers
from the view of Skygear. To actually give the role the power to assign
roles to other users, you have to designate those as admin roles.

```javascript
skygear.setAdminRole([Webmaster]);
```

It is possible to designate multiple roles to have admin privilege.

Finally, you have to tell Skygear Server what the default role a new user should have.
To do this, you have to set `visitor` as the default role. New user will
automatically gain the `visitor` role.

```javascript
skygear.setDefaultRole([Visitor]);
```

Remember that the above definitions are freezed at the time the application
is switched to production mode.

## Access types

There are two access types: read and write.

A user having read access to a record will be able to fetch and query the
record. This includes getting all fields of the record as well as the access
control settings for the record.

A user having write access to a record will be able to save and delete the
record. This includes adding, updating and removing all fields of the record
as well as the access control settings for the record.

A user having write access cannot change the ownership of the records.

## Set role-based access for each record

In role-based access control, you add access to a record by specifying which
roles have access to it, and which type of access is allowed.

There are three types of access types: no access, read only and read write.
A role having read only access can fetch and query the record from the database.
A role having read write access can save and delete the record in addition
to fetch and query..

You can set access by calling these methods on a record object:
* `setNoAccessForRole(...)` - remove all access of a record to the specified role
* `setReadOnlyForRole(...)` - set read only access of a record to the specified role
* `setReadWriteAccessForRole(...)` - Add read write access of a record to the specified role

In our example,
when you create an article, you want `visitor` to see it, but only
only `webmaster` and `author` will be able to change it. You can add
access by calling these functions on a record object:

```javascript
const article = new Article(params);

article.setReadWriteAccessForRole(Webmaster);
article.setReadWriteAccessForRole(Author);
article.setReadOnlyForRole(Visitor);

skygear.publicDB.save(article);
```

Role-based access control are applied to each record individually. In other
words, access control applied to a record does not affect access control
applied to other records.

Note that the owner of a record always have access to a record.

Although the above scenario will work, it is recommended to modify access
control of a record in client-side hooks. By doing this, you put the access
control logic in one place, and that access control settings are applied
consistently.

```javascript
Article.beforeSave((article) => {
  if (article.isLocked) {
    article.setReadOnlyForRole(Author);
  } else {
    article.setReadWriteAccessForRole(Author);
  }
});
```

## Setting creation access

In the above examples, you add or remove access for each individual record,
but such access control does not apply when creating a new instance of
the record. At this point a `visitor` cannot modify an existing article, but
a `visitor` can create a new instance of article.

To restrict record creation to a certain role, you can call
`setRecordCreateAccess` to tell Skygear Server which roles can create which record
instance in the bootstrapping phrase.

```javascript
skygear.setRecordCreateAccess(Article, [Webmaster, Author])
skygear.setRecordCreateAccess(Comment, [Webmaster, Author, Visitor])
```

Note that creation access is freezed in production mode.

## Default setting for access control

If you do not add access control on a record, the record will inherit
the default setting for access control. The default setting is that record
is public readable.

The default setting can be changed by calling `setDefaultACL` function:

```javascript
const acl = skygear.defaultACL;
acl.setPublicReadOnly();
acl.setReadOnlyForRole(Webmaster);
skygear.setDefaultACL(acl);
```

In the above example, each newly created record is readable by public and it
can be modified by a `webmaster`. If you add access control to a specific
record, the above settings will be ignored for that record.

## Changing role of a user

A user with an admin role is able to change roles of other users. In
previous section, we defined a `webmaster` will have admin privilege, and this
user can promote a `visitor` to an `author`.

This is done by calling `addRole` on a user object.

```javascript
skygear.getUserByEmail('johndoe@example.com').then((john) => {
  john.addRole(Author);
  john.removeRole(Visitor);
  skygear.saveUser(john).then((john) => {
    console.log('John has author role now');
  }, () => {
    console.log('Unable to promote John');
  });
}, (error) => {
  console.log('John is not found');
});
```
