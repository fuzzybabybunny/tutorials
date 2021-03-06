## Purpose ##

I haven't been able to find a good tutorial that goes through the end-to-end setup for OrionJS, so I decided to create this tutorial both as a learning resource for others but also as a way for me to keep track of my own progress as I poke around OrionJS and figure out how to do stuff with it.

There will be cursing in this tutorial.

## Cloning Microscope ##

Most Meteor developers should be familiar with Microscope, the social news app which the user codes along with when following the [Discover Meteor](https://www.discovermeteor.com/) tutorial book. If you haven't read and coded along with this book, do it. Now.

As of this writing, Microscope does not have a backend admin system in place to manage its data, so I thought that it would be the ideal candidate for creating a tutorial on how to get OrionJS working.

First, go to https://github.com/DiscoverMeteor/Microscope and clone the repo.

```
git clone git@github.com:DiscoverMeteor/Microscope.git
cd Microscope
meteor update
```

## Download OrionJS ##

OrionJS is designed to work nicely with Bootstrap, which is perfect because Microscope does as well.

```
meteor add orionjs:core fortawesome:fontawesome orionjs:bootstrap
```

When you do `meteor list` you should see (versions may be different):

```

accounts-password            1.1.1  Password support for accounts
audit-argument-checks        1.0.3  Try to detect inadequate input sanitization
fortawesome:fontawesome      4.3.0  Font Awesome (official): 500+ scalable vector icons, customizable via CSS, Retina friendly
ian:accounts-ui-bootstrap-3  1.2.76  Bootstrap-styled accounts-ui with multi-language support.
iron:router                  1.0.9  Routing specifically designed for Meteor
orionjs:bootstrap            1.2.1  A simple theme for orion
orionjs:core                 1.2.0  Orion
sacha:spin                   2.3.1  Simple spinner package for Meteor
standard-app-packages        1.0.5  Moved to meteor-platform
twbs:bootstrap               3.3.5  The most popular front-end framework for developing responsive, mobile first projects on the web.
underscore                   1.0.3  Collection of small helpers: _.map, _.each, ...

```
## Initial Impressions ##

Great! Now that we've added all these packages, let's start up Microscope and see how screwed up we made everything.
```
meteor
```
Then point your browser to `http://localhost:3000/`

Everything looks like it should:

![enter image description here](https://lh3.googleusercontent.com/ADsFjG6v1AlTMaULUSVMG9qEBKrCQ902sUXuQI5N1dU=s0 "Screenshot from 2015-07-23 22:54:16.png")

Now let's go to `http://localhost:3000/admin`

![enter image description here](https://lh3.googleusercontent.com/yjUHPNxc1wclW3M-pPFIk1J_F8IglciV8NU85nIwNE0=s0 "Screenshot from 2015-07-23 22:56:17.png")

Looks like crap.

The reason this is here is because there's a hidden red alert box that is floating right. It's a style that was defined in Microscope so let's go remove it. Comment out that line of code.

```css
/client/stylesheets/style.css

.alert {
          animation: fadeOut 2700ms ease-in 0s 1 forwards;
  -webkit-animation: fadeOut 2700ms ease-in 0s 1 forwards;
     -moz-animation: fadeOut 2700ms ease-in 0s 1 forwards;
  width: 250px;
/*  float: right;*/
  clear: both;
  margin-bottom: 5px;
  pointer-events: auto;
}
```
This is going to make Microscope look crappier but I'm crap at CSS so screwit.

![enter image description here](https://lh3.googleusercontent.com/UwazTdhYJp0Rx_aYIbwJp52UOQbhLPfv5VvqrZJH8jo=s0 "Screenshot from 2015-07-23 23:10:09.png")

Better. The invisible alert box is still taking up space but I can't be screwed to do anything about it.

## Creating Users ##

Looks like we need to create users and log in first before we can see the OrionJS backend.

Go here and replace the original `// create two users` code with this:

(Don't just copy this code and replace everything in `fixtures.js` with this since it's just partial code.)

```javascript
/server/fixtures.js

// Fixture data
if (Posts.find().count() === 0) {
  var now = new Date().getTime();

  // create two users

  var sachaId = Accounts.createUser({
    profile: {
      name: 'Sacha Greif'
    },
    username: "sacha",
    email: "sacha@example.com",
    password: "123456",
  });

  var tomId = Accounts.createUser({
    profile: {
      name: 'Tom Coleman'
    },
    username: "tom",
    email: "tom@example.com",
    password: "123456",
  });

  var sacha = Meteor.users.findOne(sachaId);
  var tom = Meteor.users.findOne(tomId);

  var telescopeId = Posts.insert({
    title: 'Introducing Telescope',
    userId: sacha._id,
    author: sacha.profile.name,
    url: 'http://sachagreif.com/introducing-telescope/',
    submitted: new Date(now - 7 * 3600 * 1000),
    commentsCount: 2,
    upvoters: [], votes: 0
  });

  Comments.insert({
    postId: telescopeId,
    userId: tom._id,
    author: tom.profile.name,
    submitted: new Date(now - 5 * 3600 * 1000),
    body: 'Interesting project Sacha, can I get involved?'
  });

```

Shut down Meteor with `Ctrl+C` and do a `meteor reset` and start it back up again with `meteor`.

Now let's log in **as Sacha** and see what happens. Once logged in, click on the `Accounts` link on the left.

![enter image description here](https://lh3.googleusercontent.com/J5K9jjcJU_0KKQVfBcqfXiUNrqcto6slnUyOZ6O1Lks=s0 "Screenshot from 2015-07-23 23:41:12.png")

- Nice. Absolutely nothing shows up except the `Accounts` collection. Microscope has a `Posts` collection as well as a `Comments` collection.

- ... and WTF. Tom is an `admin` but Sacha is not. We didn't even tell Meteor to make Tom an `admin` in my fixtures code, so what the hell happened?

It turns out that by default, OrionJS will create a user `Role` called `admin` and if there is no `admin`,  will assign the **first** user created with `Accounts.createUser` as an `admin`. Remember this so you're not creating accidental admin users in your fixtures code.

I can't spell Sasha's name for crap so let's remove him as admin and have Tom as admin. Because ignoring your spelling weaknesses make them go away.

## Adding and Removing Roles from Users ##

Unlike other user roles packages, the `role` of the user in `OrionJS` isn't stored on the `user` object itself. You won't find something like:

```javascript
{
  username: 'cletus',
  email: 'ileik@mysister.com',
  roles: [
    'admin',
    'parent',
    'varmit_hunter'
  ]
}
```
Instead, each separate `role` is stored in a `Roles` collection and the `userId` of the user is referenced along with an array containing their `roles`.

### Getting Roles###
Let's screw around in the Chrome console before we do anything. While in the `Accounts` admin page, do:

```js
var id = Meteor.users.findOne({username: "sacha"})._id;
Roles.userHasRole(id, "admin")

true
```
It should return `true`. We got Sesha's userId and then used that ID to check if he has the role of "admin."

Likewise, each user has a `roles()` and `hasRole()` method on its object:

```js
// gets the currently logged in user, which should be Sassha
var user = Meteor.user();
user.roles();

["admin"]

user.hasRole("admin")

true
```
So that's how you check if a user has a role you want.

Currently there doesn't seem to be a method to check *which* users have a certain role.

###Setting Roles###

We need to give Tom the role of `admin`.

In Chrome console:
```
var user = Meteor.users.findOne({username: "tom"});
Roles.addUserToRoles( user._id ,  ["admin"] );

VM11445:2 Uncaught TypeError: Roles.addUserToRoles is not a function(anonymous function)
```
DUH. You can't define roles on the client. Because that's dumb.

Let's make a new file called `/server/admin.js`
```javascript
/server/admin.js

var tom = Meteor.users.findOne({username: 'tom'});
Roles.addUserToRoles( tom._id ,  ["admin"] );
```

If we go back to the OrionJS admin console we should see that now Tom and Sache are both admins.

Now we want to remove Sachet as an admin.

```javascript
/server/admin.js

var tom = Meteor.users.findOne({username: 'tom'});
Roles.addUserToRoles( tom._id ,  ["admin"] );

var nameIcantSpel = Meteor.users.findOne({username: 'sacha'});
Roles.removeUserFromRoles( nameIcantSpel._id, ["admin"] );
```
You'll notice that you can log into `OrionJS` as Sacsh, but you won't see `Accounts` on the sidebar since he's no longer an `admin`. So log in as Tom instead.

And now Tom is the only admin!

![enter image description here](https://lh3.googleusercontent.com/2cXRDE9AILHEWVhLumU2Fei-Tzf67Uwl8rOCZGfcsOk=s0 "Screenshot from 2015-07-24 00:26:13.png")

Spelling weakness successfully ignored!

##Adding Collections to OrionJS##

Here's where this tutorial gets less horrible.

Microscope has a `Posts` and `Comments` collection, but both aren't visible yet in the admin thingy. That's because they're not `Orion` collections yet.

Let's make them appear.

```javascript
// This is what it used to be:
// Posts = new Mongo.Collection('posts');

// Instead let's do:
Posts = new orion.collection('posts', {
  singularName: 'post', // The name of one of these items
  pluralName: 'posts', // The name of more than one of these items
  link: {
    // *
    //  * The text that you want to show in the sidebar.
    //  * The default value is the name of the collection, so
    //  * in this case it is not necessary.

    title: 'Posts'
  },
  /**
   * Tabular settings for this collection
   */
  tabular: {
    // here we set which data columns we want to appear on the data table
    // in the CMS panel
    columns: [
      {
        data: "title",
        title: "Title"
      },{
        data: "author",
        title: "Author"
      },{
        data: "submitted",
        title: "Submitted"
      },
    ]
  }
});

Posts.allow({
  update: function(userId, post) { return ownsDocument(userId, post); },
  remove: function(userId, post) { return ownsDocument(userId, post); },
});

Posts.deny({
  update: function(userId, post, fieldNames) {
    // may only edit the following two fields:
```
If you go back to the Microscope home page you'll see that everything appears to have remained normal. Orion collections are just extended Mongo collections.

Now it looks like we got `Posts` appearing in `OrionJS`.

![enter image description here](https://lh3.googleusercontent.com/6T99yNvDLm2qxDp-ZLF8W0V9L64Op8W8-UAvuOQlEqo=s0 "Screenshot from 2015-07-24 00:37:51.png")

Reactive search works. Sorting columns is working. Pagination is working.

But when we click on a table item we get:

![enter image description here](https://lh3.googleusercontent.com/UPlVwdKLDXf2UBsX3JqLdTpoG989DelOau61HHB84MA=s0 "Screenshot from 2015-07-24 00:39:06.png")

Errors and crap.

Luckily the error is pretty descriptive. This form needs either a schema or a collection.

##Updating Collection Documents##

When we click on one of the table items we expect to go to an update form for that particular item. `OrionJS` uses the vastly powerful `aldeed:autoform` package to generate its forms. `aldeed:autoform` in turn uses the `aldeed:simple-schema` package to know *how* to generate its forms.

###Schemas###

Schemas are these little (tee-hee) things that define how the data in your database should be. If you've got a `User` document with a `first_name` property, you'd expect the value to be a `type: String`. If having a first name is critical, you'd want it to be `optional: false`.

We use schemas to keep our data consistent. MongoDB is inherently a schema-less database. It would happily allow you to screw yourself over by storing an array of booleans inside the `first_name` property of your `user` document, for instance. And then you go to access it and your wife leaves you (probably not your husband because he's clueless).

So this is why people decided to make a schema package for Meteor. They love you and want happy families.

Let's start by defining a schema for our `Posts` collection all the way at the very bottom of `/lib/collections/posts.js`. I'm too lazy to type so read the comments.


```javascript
/lib/collections/posts.js

// Rest of the code above

      $addToSet: {upvoters: this.userId},
      $inc: {votes: 1}
    });

    if (! affected)
      throw new Meteor.Error('invalid', "You weren't able to upvote that post");
  }
});

/**
 * Now we will define and attach the schema for this collection.
 * Orion will automatically create the corresponding form.
 */
Posts.attachSchema(new SimpleSchema({
  // We use `label` to put a custom label for this form field
  // Otherwise it would default to `Title`
  // 'optional: false' means that this field is required
  // If it's blank, the form won't submit and you'll get a red error message
  // 'type' is where you can set the expected data type for the 'title' key's value
  title: {
    type: String,
    optional: false,
    label: 'Post Title'
  },
  // regEx will validate this form field according to a RegEx for a URL
  url: {
    type: String,
    optional: false,
    label: 'URL',
    regEx: SimpleSchema.RegEx.Url
  },
  // autoform determines other aspects for how the form is generated
  // In this case we're making this field hidden from view
  userId: {
    type: String,
    optional: false,
    autoform: {
      type: "hidden",
      label: false
    },
  },
  author: {
    type: String,
    optional: false,
  },
  // 'type: Date' means that this field is expecting a data as an entry
  submitted: {
    type: Date,
    optional: false,
  },
  commentsCount: {
    type: Number,
    optional: false
  },
  // 'type: [String]' means this key's value is an array of strings'
  upvoters: {
    type: [String],
    optional: true,
    autoform: {
      disabled: true,
      label: false
    },
  },
  votes: {
    type: Number,
    optional: true
  },
}));
```

I told you schemas are little, right?

Save and try clicking on a post item again.

![enter image description here](https://lh3.googleusercontent.com/xUKRDnSsT3THV3fi-HocLdl3uAwkTyt_YRC_nswc8eA=s0 "Screenshot from 2015-07-24 03:57:39.png")

Ohhhhh... crap! It's almost like I... planned... things.

Play around with this form and look at how the `schema` we defined directly correlates to how this form was generated.

- make the `Post Title` blank and save the form
- remove `http` in `URL` and save the form
- click on the `Submitted` field to see how `type: Date` works

##Adding Comments Collection##

How about we do the same thing to the `Comments` collection as we did to the `Posts` collection?

I'll race you. Ready? Go.
Done I WIN.

```javascript
/lib/collections/comments.js

Comments = new orion.collection('comments', {
  singularName: 'comment', // The name of one of these items
  pluralName: 'comments', // The name of more than one of these items
  link: {
    // *
    //  * The text that you want to show in the sidebar.
    //  * The default value is the name of the collection, so
    //  * in this case it is not necessary.

    title: 'Comments'
  },
  /**
   * Tabular settings for this collection
   */
  tabular: {
    // here we set which data columns we want to appear on the data table
    // in the CMS panel
    columns: [
      {
        data: "author",
        title: "Author"
      },{
        data: "postId",
        title: "Post ID"
      },{
        data: "submitted",
        title: "Submitted"
      },
    ]
  }
});

Meteor.methods({
  commentInsert: function(commentAttributes) {
    check(this.userId, String);
    check(commentAttributes, {
      postId: String,
      body: String
    });

    var user = Meteor.user();
    var post = Posts.findOne(commentAttributes.postId);

    if (!post)
      throw new Meteor.Error('invalid-comment', 'You must comment on a post');

    comment = _.extend(commentAttributes, {
      userId: user._id,
      author: user.username,
      submitted: new Date()
    });

    // update the post with the number of comments
    Posts.update(comment.postId, {$inc: {commentsCount: 1}});

    // create the comment, save the id
    comment._id = Comments.insert(comment);

    // now create a notification, informing the user that there's been a comment
    createCommentNotification(comment);

    return comment._id;
  }
});

/**
 * Now we will define and attach the schema for that collection.
 * Orion will automatically create the corresponding form.
 */
Comments.attachSchema(new SimpleSchema({
  postId: {
    type: String,
    optional: false,
    label: 'Post ID'
  },
  userId: {
    type: String,
    optional: false,
    label: 'User ID',
  },
  author: {
    type: String,
    optional: false,
  },
  submitted: {
    type: Date,
    optional: false,
  },
  body: {
    type: String,
    optional: false,
  }
}));
```

![enter image description here](https://lh3.googleusercontent.com/LROiQg6mMP5rpG844QNv-njdxyB5ltffUASPpbx606M=s0 "Screenshot from 2015-07-24 04:33:38.png")

![enter image description here](https://lh3.googleusercontent.com/89A5cA83MF5WZn-69hoq2yqPvh5hvllTXyTpuQMuJJU=s0 "Screenshot from 2015-07-24 04:34:43.png")

How about we do something even more CRAZY and add in a text editor for the `Body` field?

##Custom Input Types (Widgets)##

First, some background.

Forms have standard input types. Checkboxes. Radio buttons. Text. These are all supported out of the box by `aldeed:autoform`. But things like text editors (with buttons for selecting font type, size, color, etc) are custom, and `autoform` gives us a way to define our own widgets in the `schema` so that autoform can generate that form for us.

###Adding Summernote###

First do `meteor add orionjs:summernote`

Now do this to the `body` property:

```javascript
/lib/collections/comments.js

    // now create a notification, informing the user that there's been a comment
    createCommentNotification(comment);

    return comment._id;
  }
});

/**
 * Now we will attach the schema for that collection.
 * Orion will automatically create the corresponding form.
 */
Comments.attachSchema(new SimpleSchema({
  postId: {
    type: String,
    optional: false,
    label: 'Post ID'
  },
  userId: {
    type: String,
    optional: false,
    label: 'User ID',
  },
  author: {
    type: String,
    optional: false,
  },
  submitted: {
    type: Date,
    optional: false,
  },
  body: orion.attribute('summernote', {
    label: 'Body'
  }),
}));
```
Click on the comment by Sashi in the OrionJS admin panel to see what's up.

![enter image description here](https://lh3.googleusercontent.com/unDs4gjPq22hcncc11LY_Cp9eTD5RBh9G6IChwiVMYo=s0 "Screenshot from 2015-07-24 18:30:25.png")

Niiiice.

###Orion Attributes###

So what the hell is the `orion.attribute` we adding as the value for the `body` key in our schema? How about we just use our Chrome Console?
```
orion.attribute('summernote', {
  label: 'Body'
})

Object {label: "Body", type: function, orionAttribute: "summernote", ...}
autoform: Object
label: "Body"
orionAttribute: "summernote"
type: function String() { [native code] }
__proto__: Object
```
Oh. looks like by adding the `orionjs:summernote` package we got access to this method that returns a pre-made object for us that we can conveniently use in our schema. Remember that `aldeed:autoform` uses the schema to figure out *how* to generate form items, so this attribute did all the defining-the-input-widget stuff for us.

`orion.attribute('nameOfAnAttribute', optionalObjectToExtendThisAttributeWith)`

I'm going to make this comment fabulous. ROYGBIV and Comic Sans the crap out of that comment, Sechie.

![enter image description here](https://lh3.googleusercontent.com/eVMz0FVebRmy05ZPPU9p3b_91o56QjbK7GfgK0mIvaA=s0 "Screenshot from 2015-07-24 18:37:15.png")

Save it.

Remember that this comment is now in HTML, so we have to add triple spacebars `{{{ TRIPLEX #vindiesel }}}` in our `comment_item.html` to make sure Spacebars will render the HTML properly.

```html
/client/templates/comments/comment_item.html

<template name="commentItem">
  <li>
    <h4>
      <span class="author">{{author}}</span>
      <span class="date">on {{submittedText}}</span>
    </h4>
    <p>{{{ body }}}</p>
  </li>
</template>
```
Let's survey the improvements by going to the main page and clicking on the comments for the `Introducing Telescope` post.

![enter image description here](https://lh3.googleusercontent.com/8ja1upLimMWalVzUCAMWNmtULEb-xpvPzOiih5l99wg=s0 "Screenshot from 2015-07-24 18:46:03.png")

Beauty.

###Adding Images to Amazon S3 (updated 07/28/2015)###

No comment will be complete without image spamming.

Some notes:

- currently, uploading images directly from Summernote doesn't work
- images will be hosted through Amazon S3. I don't go over how to do it with `GridFS` or other filesystem packages.

`meteor add orionjs:image-attribute orionjs:filesystem orionjs:s3`

####Setting up S3####

You're going to want to follow this tutorial FIRST to set up your Amazon S3:

https://github.com/Lepozepo/S3/#amazon-s3-uploader

Make sure that you've got your S3 credentials in your `server` folder. It wouldn't hurt to add this file to your `.gitignore` file as well. On second thought, scratch what I just said and send me links to your repos instead... because... porn.

```javascript
/server/s3_credentials.js

// something like this
S3.config = {
    key: 'BVT&(Y*(H&TG*&H',
    secret: 'B^&Y*UGUFGO*(PU(/7sdfgwTVwS/',
    bucket: 'meteor.microscopelolololololol',
    region: 'us-west-1'
};
```

####Configuring OrionJS####

Create a new file:

```javascript
/lib/orion_filesystem.js

/**
 * Official S3 Upload Provider
 *
 * Please replace this function with the
 * provider you prefer.
 *
 * If success, call success(publicUrl);
 * you can pass data and it will be saved in file.meta
 * Ej: success(publicUrl, {local_path: '/user/path/to/file'})
 *
 * If it fails, call failure(error).
 *
 * When the progress change, call progress(newProgress)
 */
orion.filesystem.providerUpload = function(options, success, failure, progress) {
  S3.upload({
    files: options.fileList,
    path: 'orionjs',
  }, function(error, result) {
    debugger
    if (error) {
      failure(error);
    } else {
      success(result.secure_url, { s3Path: result.relative_url });
      result;
      debugger
    }
    S3.collection.remove({})
  });
  Tracker.autorun(function () {
    var file = S3.collection.findOne();
    if (file) {
      progress(file.percent_uploaded);
    }
  });
};

/**
 * Official S3 Remove Provider
 *
 * Please replace this function with the
 * provider you prefer.
 *
 * If success, call success();
 * If it fails, call failure(error).
 */
orion.filesystem.providerRemove = function(file, success, failure)  {
  S3.delete(file.meta.s3Path, function(error, result) {
    if (error) {
      failure(error);
    } else {
      success();
    }
  })
};
```
What this bit of code does is it defines two methods - one for uploading a file to S3 and another for removing a file from S3. It also adds a progress bar during the upload process.

Now it's schema time again since we want to add a file upload section to our Comment Update form!

```javascript
/lib/collections/comments.js

  author: {
    type: String,
    optional: false,
  },
  submitted: {
    type: Date,
    optional: false,
  },
  body: orion.attribute('summernote', {
    label: 'Body'
  }),
  image: orion.attribute('image', {
    optional: true,
    label: 'Comment Image'
  }),
}));
```

Go into a comment in the admin panel and upload something!

![enter image description here](https://lh3.googleusercontent.com/ZY2zJWFCdv25VDgbAE3yjPxD-ByUIngSW9-SEuzloaI=s0 "Screenshot from 2015-07-29 00:04:39.png")

Make sure to click on the `Save` button after you're done.  Also note that the moment you select an image the Amazon uploader will start. You'll see a progress bar.

Finally, we should go see what it looks like on the main page, but remember to modify the `comment_item` template with the new `image` key that a `comment` document now has.

```
/client/templates/comments/comment_item.html

<template name="commentItem">
  <li>
    <h4>
      <span class="author">{{author}}</span>
      <span class="date">on {{submittedText}}</span>
    </h4>
    <p>{{{body}}}</p>
    {{#if image }}
      <img src="{{image.url}}">
    {{/if}}
  </li>
</template>
```

Voila!

![enter image description here](https://lh3.googleusercontent.com/bKct-GlqtHXJYrfQkoiaZrq1XH0D1Q08UyH_PUgPZYE=s0 "Screenshot from 2015-07-29 00:05:07.png")

Sassha and Tom are going to KILL me.

##Changing Tabular Templates (updated 7/29/2015)##

If we go back to our admin panel and look at comments, we see that the table is pretty dumb:

`![enter image description here](https://lh3.googleusercontent.com/LROiQg6mMP5rpG844QNv-njdxyB5ltffUASPpbx606M=s0 "Screenshot from 2015-07-24 04:33:38.png")

1. The `Submitted` column contains WAY too much information. Something like Month-Day-Year-Time would look nicer. I'm going to completely ignore you people who do it the more logical way of Time-Day-Month-Year because, uh, freedom.

2. We also have the issue of the `Post ID` column being essentially stupid. I'd prefer if that column contained the title of the Post instead.

3. I also want a column that shows a short blurb of the comment's `body`, something like `Interesting project Sacha, can I...`

Let's tackle #1 first. If you guessed that we need to go back into our `Schema` to change this, you just WON the JACKPOT of zero money.

###orion.attributeColumn()###

We are interested in:

      orion.attributeColumn('createdAt', 'submitted', 'FREEDOM!!!'),

```javascript
/lib/collections/comments.js

Comments = new orion.collection('comments', {
  singularName: 'comment', // The name of one of these items
  pluralName: 'comments', // The name of more than one of these items
  link: {
    // *
    //  * The text that you want to show in the sidebar.
    //  * The default value is the name of the collection, so
    //  * in this case it is not necessary.

    title: 'Comments'
  },
  /**
   * Tabular settings for this collection
   */
  tabular: {
    // here we set which data columns we want to appear on the data table
    // in the CMS panel
    columns: [
      {
        data: "author",
        title: "Author"
      },{
        data: "postId",
        title: "Post ID"
      },
      orion.attributeColumn('createdAt', 'submitted', 'FREEDOM!!!'),
    ]
  }
});
```

Let's check it out!

![enter image description here](https://lh3.googleusercontent.com/YVicjxMpRLn3A5J1xNI-PRRQGWjCdtt_u6E77QHEp9A=s0 "Screenshot from 2015-07-24 20:10:05.png")

This handy-dandy method goes like this:

`orion.attributeColumn('nameOfTemplate', 'keyNameOnYourObject', 'columnLabel')`

Think about it. Meteor uses templates to display stuff. We had a crazy long date and we wanted to change the *look* of it, so using a template makes sense.

Luckily, `OrionJS` comes with some pre-made templates. One of them happens to be called `createdAt`.

`createdAt` wants a `Date` object, which happens to reside in the `submitted` key of each of your documents in the `Comments` collection. Lastly, we tell it what we want our column label to be.

Now, some freedom-hating people probably want a custom template for Time-Day-Month-Year. Let's get to that next.

###Custom Tabular Templates###

Now onto issue #2 - the `Post ID` column should be the title of the Post instead.

Open up Chrome Console and type in `Comments.findOne()`. It found a comment, right?

Now do `Posts.findOne()`. Hmmm... no post found. That's because this route isn't subscribed to anything in the `Posts` collection. If you've done Meteor before, chances are you've been using `Iron Router` to manage your data subscriptions for your routes. Unfortunately, since OrionJS is a package, it's difficult to tap into and modify the generated routes that OrionJS has already made for us. So what do we do?

####Template-Level Subscriptions####

Meteor can subscribe to data in normal template callbacks (`onRendered, onCreated`). And as it turns out, OrionJS has a very standardized template naming scheme*.

`collections.myCollection.index` - the main page that lists all the items in the collection
`collections.myCollection.create` - the form for creating a new item in the collection
`collections.myCollection.update` - the form for updating an existing item in the collection
`collections.myCollection.delete` - the form/page for deleting an existing item in the collection

These, by the way, are your standard pages for CRUD actions.

*These aren't necessarily the actual names of the templates, but the `identifier` that OrionJS uses to find the actual template.

BTW, if you want to see a list of all the route names that are registered with Iron Router, open up Chrome Console and do:

```javascript
_.each(Router.routes, function(route){
  console.log(route.getName());
});
```

Here are some more identifiers: http://orionjs.org/docs/customization#overridetemplates

So let's use this to subscribe to the `Posts` collection on the `comments.index` template.

```javascript
/client/templates/orion/comments_index.js

ReactiveTemplates.onCreated('collections.comments.index', function() {

  this.subscribe('posts', {sort: {submitted: -1, _id: -1}, limit: 0});

});
```

What this is saying is that when the template with an identifier of 'collections.comments.index' (the `Comments` index page) is created, subscribe the template to the data you specify.

Now go back to the `Comments` index page and do `Posts.find().count()` to see that you've got `Posts` now!

####Meteor Tabular Render####

So now that this page has the data we need, how do we actually change the contents of the cell itself?

`OrionJS` uses `aldeed:meteor-tabular` to show its datatables, and it just so happens that this latter package provides a way to change the cell value: https://github.com/aldeed/meteor-tabular#example

```javascript
/lib/collections/comments.js

Comments = new orion.collection('comments', {
  singularName: 'comment', // The name of one of these items
  pluralName: 'comments', // The name of more than one of these items
  link: {
    title: 'Comments'
  },
  /**
   * Tabular settings for this collection
   */
  tabular: {
    // here we set which data columns we want to appear on the data table
    // in the CMS panel
    columns: [
      {
        data: "author",
        title: "Author"
      },{
        data: "postId",
        title: "Post Title",
        render: function (val, type, doc) {
          var postId = val;
          var postTitle = Posts.findOne(postId).title;
          return postTitle;
        }
      },
      orion.attributeColumn('createdAt', 'submitted', 'FREEDOM!!!'),
    ]
  },
});
```
Go play around inside this function. `console.log` `val`, `type`, and `doc` to see what they are:

```javascript
{
  data: "postId",
  title: "Post Title",
  render: function (val, type, doc) {
    var postId = val;
    var postTitle = Posts.findOne(postId).title;
    return postTitle;
  }
}
```
`data: "postId"` is critical here because that's how `val` gets its value.

After you're through go back and look at the `Comments` index page:

![enter image description here](https://lh3.googleusercontent.com/xuT9mpGBby25enedfg64fTEyuVWIXP_jFPXokNKpkx0=s0 "Screenshot from 2015-07-29 17:06:16.png")

####Meteor Tabular with Actual Templates####

Finally, onto #3. We want a column that shows a short blurb of the comment `body`, something like `Interesting project Sacha, can I...`. This is called `truncating` a string.

I want to create an actual template for this:

```html
/client/templates/orion/comments_index_blurb_cell.html

<template name="commentsIndexBlurbCell">
  {{{ blurb }}}
</template>
```

Hold on there Skippy! Truncating a straight string that's, say, 100 characters long into one that's 15 characters long with a `...` at the end is fairly straightforward. But remember that we added Summernote and we have a *fabulous* comment?

![enter image description here](https://lh3.googleusercontent.com/8ja1upLimMWalVzUCAMWNmtULEb-xpvPzOiih5l99wg=s0 "Screenshot from 2015-07-24 18:46:03.png")

The HTML for this comment actually looks like:

```html
<p><span style=\"font-family: 'Comic Sans MS'; font-size: 18px;\"><span style=\"background-color: rgb(255, 0, 0);\">You</span> <span style=\"background-color: rgb(255, 156, 0);\">sure</span> <span style=\"background-color: rgb(255, 255, 0);\">can</span> <span style=\"background-color: rgb(0, 255, 0);\">Tom</span><span style=\"background-color: rgb(0, 0, 255);\">!!!</span></span></p>
```

Sooo... we can't just do a simple truncate of this down to 15 characters. I mean, we can...

...IF WE'RE NUBZ!

But `pathable` is not a nub:

https://github.com/pathable/truncate

Go to `/client/javascript` and literally just chuck this script's `jquery.truncate.js` file in there. Meteor will take care of minifying and loading this script automatically onto your page, as it does with *all* javascript that's not in the `/public` folder.

And now we can go ahead and create a helper for our template:

```javascript
/client/templates/orion/comments_index_blurb_cell.html

Template.commentsIndexBlurbCell.helpers({

  blurb: function(){
    var blurb = jQuery.truncate(this.body, {
      length: 15
    });
    return blurb
  }

});

```

Finally, we go back to modify our `tabular` object:

```javascript
/lib/comments.js

Comments = new orion.collection('comments', {
  singularName: 'comment', // The name of one of these items
  pluralName: 'comments', // The name of more than one of these items
  link: {
    title: 'Comments'
  },
  /**
   * Tabular settings for this collection
   */
  tabular: {
    // here we set which data columns we want to appear on the data table
    // in the CMS panel
    columns: [
      {
        data: "author",
        title: "Author"
      },{
        data: "postId",
        title: "Post Title",
        render: function (val, type, doc) {
          var postId = val;
          var postTitle = Posts.findOne(postId).title;
          return postTitle;
        }
      },{
        data: "body",
        title: "Comment",
        tmpl: Meteor.isClient && Template.commentsIndexBlurbCell
      },
      orion.attributeColumn('createdAt', 'submitted', 'FREEDOM!!!'),
    ]
  },
});
```
`data: "body"` subscribes us to the values for the `body` key so that it's available for our template.

`tmpl: Meteor.isClient && Template.commentsIndexBlurbCell` looks kind of weird but remember that this code is in `/lib`, which runs on both the client and server, and `Template` isn't defined on the server. So that's why `aldeed:meteor-tabular` requires you to do this `Meteor.isClient` thing.

WABAM!

![enter image description here](https://lh3.googleusercontent.com/_DiQmOxmyTqnDpR0lpfP3WC-D1TadvsDG08J_ApZvmg=s0 "Screenshot from 2015-07-29 17:53:03.png")

And... done.

##Dictionary (updated 7/28/2015)##

Let's go back to the Microscope main page. Say that you wanted to add a little description blurb after the word `Microscope` at the top left.

- You not only want to add a description, but you want to be able to periodically change it as well AND you want text formatting on it AND you don't want to touch any code to change it - all you want is to change it from the OrionJS admin panel from inside of an update form.

- And since I'm rolling right now, let's be able to change the `Microscope` word as well.

- AND since people like to sue, let's add a terms and conditions blurb at the bottom of every `Post Submit` page that our lawyers can periodically update with worse and worse conditions for the consumer.

Sounds like we need a collection, a schema, and a dictionary of some sort that keeps track of  lolidontknowhowtoexplainjustkeepreading.

Create a new file:
```javascript
/lib/orion_dictionary.js

orion.dictionary.addDefinition('title', 'mainPage', {
    type: String,
    label: 'Site Title',
    optional: false,
    min: 1,
    max: 40
});

orion.dictionary.addDefinition('description', 'mainPage',
  orion.attribute('summernote', {
    label: 'Site Description',
    optional: true
  })
);

orion.dictionary.addDefinition('termsAndConditions', 'submitPostPage',  
  orion.attribute('summernote', {
    label: 'Terms and Conditions',
    optional: true
  })
);
```

`orion.dictionary.addDefinition()` takes three arguments:
```
orion.dictionary.addDefinition(
   nameOfYourDictionaryItem,
   categoryOfYourDictionaryItem,
   schemaForYourDictionaryItem
);
```
You'll see how this pans out in a little bit.
```
/client/templates/includes/header.html

<template name="header">
  <nav class="navbar navbar-default" role="navigation">
    <div class="navbar-header">
      <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#navigation">
        <span class="sr-only">Toggle navigation</span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
      </button>
      <a class="navbar-brand" href="{{pathFor 'home'}}">

<!-- The third argument, `Microscope`, is the default value that will be shown if `mainPage.title` doesn't have a value set. -->
        {{ dict 'mainPage.title' 'Microscope' }}

        {{#if dict 'mainPage.description'}}
         - {{{ dict 'mainPage.description' }}}
        {{/if}}

      </a>
    </div>
    <div class="collapse navbar-collapse" id="navigation">
      <ul class="nav navbar-nav">
```

```
/client/templates/posts/post_submit.html

    <div class="form-group {{errorClass 'title'}}">
      <label class="control-label" for="title">Title</label>
      <div class="controls">
          <input name="title" id="title" type="text" value="" placeholder="Name your post" class="form-control"/>
          <span class="help-block">{{errorMessage 'title'}}</span>
      </div>
    </div>
    <input type="submit" value="Submit" class="btn btn-primary"/>
  </form>

  {{#if dict 'submitPostPage.termsAndConditions'}}
  <div>TERMS AND CONDITIONS</div>
  <div>{{{ dict 'submitPostPage.termsAndConditions' }}}</div>
  {{/if}}

</template>
```
Let's look at the damage!

Looks like a new entry called `Dictionary` was created. And `nameOfYourDictionaryItem, categoryOfYourDictionaryItem,` and `schemaForYourDictionaryItem` are being used from `orion.dictionary.addDefinition()`. Go ahead and mess around with the forms.

![enter image description here](https://lh3.googleusercontent.com/Fda7VeIxXRViSV6JjEZY3dR3A2cP2pnZ__5GjtAWdJE=s0 "Screenshot from 2015-07-28 00:39:13.png")

![enter image description here](https://lh3.googleusercontent.com/T-PkZDKttbIDYYxBldnpI-XYJJSVk6MqxaHFCpoTxNI=s0 "Screenshot from 2015-07-28 00:39:35.png")

Now for the front-end!

![enter image description here](https://lh3.googleusercontent.com/nMwCXcpq0ntrzcfVB-EB_9w1JX02jIoq3D-wE6BAfrk=s0 "Screenshot from 2015-07-28 00:40:10.png")

![enter image description here](https://lh3.googleusercontent.com/cSMgyE6l0hrFCJ8Mzuoot8teEcptPP3AFb9btg5Vq3M=s0 "Screenshot from 2015-07-28 00:39:57.png")

So PRO! The clipping-off of the T&C gives legitimacy and trustworthiness to the site.

##Relationships##

[Additional Documentation](https://github.com/orionjs/documentation/blob/master/docs/attributes/relationships.md)

OrionJS has the ability to define two types of relationships between collection objects, `hasOne` and `hasMany`. You can use these relationships to easily do CRUD between collections inside of the admin backend.

`meteor add orionjs:relationships`

In Microscope:

`Posts` has many `Comments`
`Comments` has one `Post`

###hasOne###

Let's do `hasOne` first. Just like with a traditional SQL database, the relationships are defined in the schema as well.

```
/lib/collections/comments.js

Comments.attachSchema(new SimpleSchema({
  // here is where we define `a comment has one post`
  // Each document in Comment has a postId
  postId: orion.attribute('hasOne', {
    // the label is the text that will show up on the Update form's label
    label: 'Post'
  }, {
    // specify the collection you're making the relationship with
    collection: Posts,
    // the key whose value you want to show for each Post document on the Update form
    titleField: 'title',
    // dunno
    publicationName: 'someRandomString',
  }),
  userId: {
    type: String,
    optional: false,
    label: 'User ID',
  },
  author: {
    type: String,
    optional: false,
  },
  submitted: {
    type: Date,
    optional: false,
  },
  body: orion.attribute('summernote', {
    label: 'Body'
  }),
}));
```

Now go to the admin backend and click on a comment:

![enter image description here](https://lh3.googleusercontent.com/4t50qpBophwbQ3q-gYqMaI2NDO6mJDYTlBNTgsc4Qok=s0 "Screenshot from 2015-07-28 03:24:36.png")

Will you look at that... OrionJS was able to determine the specific `post` that this comment is related to and create a dropdown menu for us to change the post if we so desire. And it works - if you change the post and click Save you'll see the comment pop up in there.

###hasMany###

One `Post` has many `Comments`. Let's go to the schema for `Posts`

```javascript
/lib/collections/posts.js

Posts.attachSchema(new SimpleSchema({

  comments: orion.attribute('hasMany', {
    // the value inside the `comments` key will be an array of comment IDs
    type: [String],
    label: 'Comments for this Post',
    // optional is true because you can have a post without comments
    optional: true
  }, {
    collection: Comments,
    titleField: 'body',
    publicationName: 'someOtherRandomString'
  }),

  // We use `label` to put a custom label for this form field
  // Otherwise it would default to `Title`
  // 'optional: false' means that this field is required
  // If it's blank, the form won't submit and you'll get a red error message
  // 'type' is where you can set the expected data type for the 'title' key's value
  title: {
    type: String,
    optional: false,
    label: 'Post Title'
  },
  // regEx will validate this form field according to a RegEx for a URL
  url: {
    type: String,
    optional: false,
    label: 'URL',
    regEx: SimpleSchema.RegEx.Url
  },
  // autoform determines other aspects for how the form is generated
  // In this case we're making this field hidden from view
  userId: {
    type: String,
    optional: false,
    autoform: {
      type: "hidden",
      label: false
    },
  },
  author: {
    type: String,
    optional: false,
  },
  // 'type: Date' means that this field is expecting a data as an entry
  submitted: {
    type: Date,
    optional: false,
  },
  commentsCount: {
    type: Number,
    optional: false
  },
  // 'type: [String]' means this key's value is an array of strings'
  upvoters: {
    type: [String],
    optional: true,
    autoform: {
      disabled: true,
      label: false
    },
  },
  votes: {
    type: Number,
    optional: true
  },
}));
```
Inside the same file disable this `deny` rule:
```javascript
// Posts.deny({
//   update: function(userId, post, fieldNames) {
//     // may only edit the following two fields:
//     return (_.without(fieldNames, 'url', 'title').length > 0);
//   }
// });
```

Let's see it!

![enter image description here](https://lh3.googleusercontent.com/uM36XR54Q94ZpFu5dwqukgRRb318eoMtvQ72Kfn4PTg=s0 "Screenshot from 2015-07-28 04:36:28.png")

Niiiice.

BUT let's talk about the limitations of defining relationships:

- If you remove a comment on the Update Post page, it will NOT automatically remove that post from the associated Update Comment page or on the main website.

- Likewise, if you change the post for a particular comment in the Update Comment page, it will also not automatically reflect in the associated Post page or on the main website.

- This is because MongoDB is inherently non-relational and implementing hard-relations like in an SQL DB requires extra code (which isn't currently available in this `orionjs:relationships` package.

- In conclusion, be very careful setting "relationships." You can easily get some marvelous data inconsistencies that will LITERALLY lead to the extinction of all cats.

##Custom Functions (none)##
