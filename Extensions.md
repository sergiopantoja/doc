# Extensions

Extensions allow users to customize their app experience without depending on the app's core developer. For example, suppose many users demanded an "export to PDF" feature in a notes app built on top of Standard File. The developer of the notes app does not wish to implement this feature, as it would make the codebase larger, more complex, and less sustainable, and would rather keep just the essentials.

Instead, an outside developer can decide to build a "Export to PDF" extension. Any app that supports extensions can now support Export to PDF functionality without needing to build it themselves, or even understand how it works.

## Reference Implementations

[Dropbox Sync Extension](https://github.com/standardnotes/website/blob/master/app/controllers/dropbox_controller.rb)

[Standard Journal Extension](https://github.com/standardnotes/standard-journal)

## Specification
The core interface of an extension is a primary URL that supports GET requests. An extension developer can implement an extension system however they choose, as long as the end result is a URL the end user can plug in to applications.

When a user enters the URL into the client application, the client will perform a GET request to that URL. The server will return actions available to the extension.

**Request:**

`GET extension_url`
Parameters:

- item_uuid (optional)
- content_type (required if item_uuid supplied, otherwise optional. The content_type of the item. The server can use this to determine if this content type is supported by this extension.)

**Response:**

The extension's name, supported content types, and:

-  If content_type or item_uuid was provided in the params, then the response will be an array of Actions that apply to this item or this item type (nested under "actions").

### The `Action` Model

An action model has the following fields:

- **label**: what should be displayed to the user.
- **url**: the URL the app should perform
- **type**: the type of the URL action. This could be one of ["get", "post", "show", "delete"]. `show` opens the URL in a browser. The rest are API requests.
- **structures**: an array of Structure metadata that describes what is required to be sent with the request. For `content` properties, the requested fields should be a key path.

 ```
 [
	 	"structure" : "Note",
	 	"fields": [
	 		{
	 			"name": "uuid",
	 			"modifies": false,
	 			"mod_type" : nil
	 		},
	 		{
	 			"name": "created_at",
	 			"modifies": false,
	 			"mod_type" : nil
	 		},
 	 		{
	 			"name": "content.title",
	 			"modifies": true,
	 			"mod_type" : replace
	 		},
	 	]
 ]
 ```


Upon completion of this action, apps should either merge or append the values of keys present in this array. In this example, after the app performs this action and receives a response, it should merge or append the values of the response for these keys with the local model, and then sync data with user's server.

The app should display this list of actions and perform the respective URL of that action upon selection.

## Example

Let's try an example with a "Calculate Average" extension that calculates the mean of numbers in a Note structure.

As an extension developer, I set up a simple server that can generate unique URLS and also respond to them. I also put up a small landing page:

___
_Welcome to my extension! Use this URL in your app to install this extension_

**https://myextension.me/note/calcaverage**
___

Now, as a user, I'll copy that URL and paste it into the main client.

The client performs a GET request to https://myextension.me/calcaverage without any parameters and receives a response:

```
{
	"name" : "Average Calculator",
	"supported_types" : [ "Note" ],
	"actions" : [
		{
			"label" : "Calculate Average",
			"url" : "https://myextension.me/calcaverage",
			"type" : "post",
			"structures" : [
				{
				 	"type" : "Note",
				 	"fields": [
			 	 		{
				 			"name": "content.text",
				 			"modifies": true,
				 			"mod_type" : "insert"
				 		}
				 	]
				 }
			}
		]
	]
}
```

The client now displays this action.

The user clicks on "Calculate Average".

The client will perform a POST request:

URL: `https://myextension.me/calcaverage`

Parameters:

```
 {
 	"item" : {
 		"content.text" : "1, 3, 5, 7, 11, 13, 17, 19, 23, 29"
 	}
 }
```

Client receives response JSON:

```
{
	"item" : {
		"content.text" : "The average is: 12.8"
	}
}
```

Since the original action mod type was "insert", we'll insert this result to this Note's content.text field and sync.


## File Attachment Example

Let's do another example where we have a FileAttachment extension.

As an extension developer, I'll build a website that can handle user registration and sign in. When a user registers, I'll provider the user with a unique, secret URL that identifies their account:

> https://fileattacher.me/extension/jfd9r3ajfht478f

Next, as a user, I'll enter this URL into the main app.

The client app performs a GET request to https://fileattacher.me/extension/jfd9r3ajfht478f without any parameters and receives a response:

```
{
	"name" : "File Attacher",
	"supported_types" : [ "Note" ]
}
```

Because this extension supports the Note type, the app will display this extension in the extensions section of any Note item. And because no Actions are returned with this response, this is an Item specific extension.

Next, as a user, I select a note. I click on "Extensions" in the menu bar. The client now performs a GET request to the extension's URL in order to retreive available actions for this item.

**Request:**

`GET https://fileattacher.me/extension/jfd9r3ajfht478f`

Parameters:

```
{"item_uuid" : "439ecf9b-788f-470f-9559-65ac5179981a", "content_type" : "Note}
```

**Response:**

```
{
	"name" : "File Attacher",
	"supported_types" : [ "Note" ],
	"actions" : [
		{
			"label" : "Attach new file",
			"url" : "https://fileattacher.me/extension/jfd9r3ajfht478f/439ecf9b-788f-470f-9559-65ac5179981a/attach",
			"type" : "show",
			"structures" : nil
		}
	]
}
```
The client now displays this action.

When the user selects "Attach new file", the client app notices the `type` field is `show`, so it will simply open that URL with the user's default browser. The browser now loads the extension's website via the unique URL and shows an "Add attachment" button. The user proceeds to add an attachment titled "Expenses(1).pdf", and closes the window.

The user re-opens the extension menu in the main app. The client performs the same GET request to the extension's URL with the item's UUID and content_type to reload the available actions, and receives the following response:

```
{
	"name" : "File Attacher",
	"supported_types" : [ "Note" ],
	"actions" : [
		{
			"label" : "Attach new file",
			"url" : "https://fileattacher.me/extension/jfd9r3ajfht478f/439ecf9b-788f-470f-9559-65ac5179981a/attach",
			"type" : "show",
			"structures" : nil
		},
		{
			"label" : "Download Expenses(1).pdf",
			"url" : "https://fileattacher.me/extension/jfd9r3ajfht478f/439ecf9b-788f-470f-9559-65ac5179981a/download/i39fe9fj4d",
			"type" : "show",
			"structures" : nil
		}
	]
}
```

The client app now displays both these actions, and the user can download the attachment they uploaded via the second action.

## Third Party Sync Example

Let's say you want to sync all your notes to a third party cloud provider, Dropsquare.

First you'd visit dropsquare.com, login, and press "Generate Secret Standard File Extention URL". You receive this result:

> https://dropsquare.com/ext/ifj93jfnbuf72scm4865u29fj

Next, you'd open your main app and click "Add new extension", and you'd type in that URL.

The client app now performs a GET request to that URL, and receives this response:

```
{
	"name" : "Dropsquare Sync",
	"supported_types" : [ "Note" ],
	"actions" : [
		{
			"label" : "Auto-Push Changes",
			"url" : "https://dropsquare.com/ext/ifj93jfnbuf72scm4865u29fj/push",
			"type" : "watch:post:30",
			"structures" : [
				{
					"type" : "Note",
				 	"fields": [
			 	 		{
				 			"name": "uuid",
				 			"modifies": false,
				 		},
			 	 		{
				 			"name": "content.title",
				 			"modifies": false,
				 		},
			 	 		{
				 			"name": "content.text",
				 			"modifies": false,
				 		}
				 	]
				}
			]
		}
	]
}
```

The client notices the first action has a type `watch:post:30`.

- the `watch` means the client should watch the fields provided in the structures array for changes.

- the `post` means whenever a change is made to one of those fields, the client should `post` to the URL in that action.

- the `30` is the watch interval, or how often this event should be executed, at minimum. This means if the user changes the title of the note once, if another change is made within 30 seconds, the sync is queued and not executed until 30 seconds have passed from the previous request.

The user changes the title of a note.

The client observes the change and notifies the extension by making a POST request to `https://dropsquare.com/ext/ifj93jfnbuf72scm4865u29fj/push` with the following parameters:

```
{
	"items" : [
		{
			"content_type" : "Note",
			"uuid" : "055fb8e0-cbef-4dfa-aa09-b7fea8d3555b",
			"content" : {
				"title" : "Hello World.",
				"text" : "It's me, 42."
			}
		}
	]
}
```

Dropsquare receives those items and saves them, and responds with:

```
{
	"response_id" : "e238y492c3849t0"
}
```

The client stores the response id and sends it to any future requests to the same URL as `previous_response_id`.

The above example handles auto-saving changes. What if we wanted to auto-pull changes from Dropsquare? Our main data repository in this app is still the Standard File server. We just want to use Dropsquare as a backup data store for peace of mind.

The below scenario wouldn't make sense to use in this kind of application, but we delineate it below to demonstrate how it would be performed nonetheless.

We do a GET request to the extensions URL, and notice a new action:

```
{
	"label" : "Auto-Pull Changes",
	"url" : "https://dropsquare.com/ext/ifj93jfnbuf72scm4865u29fj/pull",
	"type" : "poll:get:10",
	"structures" : [
		{
			"type" : "Note",
		 	"fields": [
	 	 		{
		 			"name": "uuid",
		 			"modifies": true,
		 			"direction" : "in"
		 		},
	 	 		{
		 			"name": "content.title",
		 			"modifies": true,
		 			"direction" : "in"
		 		},
	 	 		{
		 			"name": "content.text",
		 			"modifies": true,
		 			"direction" : "in"
		 		}
		 	]
		}
	]
}
```

This action is a `poll` action. This means it executes something every x seconds. In this case, it will execute a GET request to `https://dropsquare.com/ext/ifj93jfnbuf72scm4865u29fj/pull` every 10 seconds.

The client looks at the structures, and notices that all the fields have a `direction` of "in", which means these paramters come in from the request, and are not going out from the app.

The server will respond with:

```
{
	"response_id" : "5ufj39fdj3",
	"items" : [
		{
			"content_type" : "Note",
			"uuid" : "...",
			"content" : {
				"title" : "Expense Report",
				"text" : "Spent lots of money today."
			}
		}
	]
}
```

The client will merge this item with local items. It will also store the `response_id` and send it with subsequent requests to the URL. The server can use this ID as a sync marker, to determine which changes should be pushed.
