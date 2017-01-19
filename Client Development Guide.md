##Standard File — Client Development Guide

This guide walks through the essentials of building an application that consumes a Standard File API. This code is based on the Standard Notes client, and uses a non-compileable Javascript-ish pseudocode.

For the full specification, see the [Standard File guide](http://standardfile.org/).

### Authentication
To authenticate a user, we need to make a request to the server to retrieve the password derivation parameters used to generate the user's password. This information is public, and can be retrieved by a GET call to `/auth/params` with `email` as a paramter:

```
getAuthParamsForEmail(email, callback) {
	var request = Restangular.one("auth", "params");
	request.get({email: email}).then(function(response){
	  callback(response.plain());
	})
}
```

Auth params include:

- pw_func (only PBKDF2 is currently supported)
- pw_alg (can be either sha256 or sha512)
- pw_salt (fed to pw_func)
- pw_cost (the number of iterations for the kdf)
- pw_key_size (the output size of the key in bits)

Next, compute the user's password and encryption keys using the user's text field inputted password:

```
generatePasswordAndKey(password, pw_salt, pw_func, pw_alg, pw_cost, pw_key_size) {
	 var algMapping = {
	   "sha256" : CryptoJS.algo.SHA256,
	   "sha512" : CryptoJS.algo.SHA512
	 }
	 var fnMapping = {
	   "pbkdf2" : CryptoJS.PBKDF2
	 }
		
	 var alg = algMapping[pw_alg];
	 var kdf = fnMapping[pw_func];
	 var output = kdf(password, pw_salt, { keySize: pw_key_size/32, hasher: alg, iterations: pw_cost }).toString();
		
	 var outputLength = output.length;
	 var pw = output.slice(0, outputLength/2);
	 var mk = output.slice(outputLength/2, outputLength);
	 return [pw, mk]
}
```

Here the return values are `pw`, which is the password that will be sent to the server, and `mk`, which is the master encryption key, which is never sent to the server.

From here, it's basic authentication as usual:

```
var request = Restangular.one("auth/sign_in");
var params = {password: pw, email: email};
request.post().then(function(response){
  var jwt = response.token;
  localStorage.setItem("jwt", response.token);
  callback(response);
})
```

Here we capture `jwt`, which is a JSON Web Token, and save it locally. This token must be passed to any future API calls to authenticate the user.

You send this token via an HTTP header:
`Authorization: Bearer _jwt_value_`

### Registration

On registration, the client must decide the auth params themselves. There are recommended defaults available [here](http://standardfile.org/#key-generation). As of the time of this writing, they are:

| Type | Value |
| --- | --- |
| Function | PBKDF2 |
| Algorithm | SHA512 |
| Cost | 60,000 |
| Key size | 512 |

To register, the client must also generate a salt. For logging in, the salt is returned by the server, but for registering, it must be done locally.

**Generating a salt:**

1. Generate a random key `pw_nonce`.
2. Compute `salt = SHA1(email + "SN" + pw_nonce)`

Then, call the same `generatePasswordAndKey` function you used for logging in. Then, when registering, make a POST request to `auth` with the auth params you used, including the salt **and** `pw_nonce`:

```
  var request = Restangular.one("auth");
  var params = {
  				password: pw, email: email, pw_salt: pw_salt, 
  				pw_nonce: pw_nonce, pw_func: pw_func, pw_alg: pw_alg, 
  				pw_cost: pw_cost, pw_key_size: pw_key_size
  				};
  request.post().then(function(response){
    localStorage.setItem("jwt", response.token);
    callback(response);
  }) 
```

Save the JWT.

## Architecture

What follows is a recommended set up for how models and controllers should be structured on the client. This can however be done in whatever way the developer prefers.

### Models

Create a class called `Item`. This model should have all metadata related fields of items returned from the server, including:

**Abstract `Item` class:**

- uuid
- content
- enc_item_key
- content_type
- presentation_name
- url
- created_at
- updated_at
- deleted

Then, create subclasses of `Item` for structures that your application supports. In the case of the Standard Notes app, that will be a `Note` class and a `Tag` class:

**Note extends Item:**

- title
- text

**Tag extends Item:**

- title

### Controllers

Three main controllers are relied on for a clean separation of concerns:

- **ItemManager**: responsible for mapping items returned from the server to local models.
- **ApiController**: responsible for communicating and retrieving changes from the server, as well as faciliating encryption/decryption with the help of CryptoHelper.
- **CryptoHelper**: decrypts/encrypts outgoing and incoming items from ApiController.

Note that encryption and decryption should be handled by the ApiController _before_ it passes it off to the ModelManager. It should be treated as an API level thing, and should not be intertwined with normal application logic.

First, let's handle fetching new items. 

**ApiController**

```
refreshItems() {
	let request = Restangular.one("items/sync");
	if(self.syncToken) {
		request.syncToken = self.syncToken;
	}
	request.post().then(function(response){
		self.syncToken = response.sync_token;
		self.handleResponseItems(response.retrieved_items);
	})
}
```

```
handleResponseItems(responseItems) {
	CryptoHelper.decryptResponseItems(responseItems);
	ItemManager.sharedInstance.mapResponseItemsToLocalItems(responseItems);
}
```

**CryptoHelper**

```
// Modifies response items in place
decryptResponseItems(responseItems) {
	for item in responseItems {
		self.decryptItem(item);
	}
}
```
We'll go over the `decryptItem` function further below.

**ItemManager**

```
mapResponseItemsToLocalItems(responseItems) {
	var mappedItems = [];
	for responseItem in responseItems {
		if responseItem["deleted"] == true {
			let item = Database.find(uuid: responseItem["uuid"]);
			if(item) {
				Database.remove(item);
			}
			continue;
		}
		
		let item = Database.findOrCreate(uuid: responseItem["uuid"]);
		item.updateFromJSON(responseItem);
		self.resolveReferencesForItem(item);
	}
}
```

**Item**

```
updateFromJSON(jsonDic) {
    self.uuid = jsonDic["uuid"]
    self.contentType = jsonDic["content_type"]
    self.presentationName = jsonDic["presentation_name"]
    self.encItemKey = json["enc_item_key"]
    self.url = jsonDic["presentation_url"]
    self.createdAt = dateFromString(jsonDic["created_at"])
    self.updatedAt = dateFromString(jsonDic["updated_at"])

    self.content = jsonDic["content"] // this is just a string
    
    let contentObject = JSON.parse(self.content)
    mapContentToLocalProperties(contentObject)
}
```

Then each subclass will override their own implementation of `mapContentToLocalProperties`:

**Note**

```
mapContentToLocalProperties(contentObject) {
	// super has no role here, but we'll call it in case we decide to make changes
	super.mapContentToLocalProperties(contentObject)
	self.title = contentObject["title"]
	self.text = contentObject["text"]
} 
```

**Tag**

```
mapContentToLocalProperties(contentObject) {
	super.mapContentToLocalProperties(contentObject)
	self.title = contentObject["title"]
} 
```

Each Item's content field can have an array of referenced items in its `references` field. This can look like this:

Item: 

```
{
  "uuid": "ec549665-824d-4078-a65a-8e1e223e33bf",
  "content_type": "Note",
  "content": {
    "references": [
      {
        "uuid": "cb92d48d-536b-40f8-be79-95de7ec1dff5",
        "content_type": "Tag"
      }
    ],
    "title": "My Note",
    "text": "Hello world."
  },
}
```

`ItemManager.resolveReferencesForItem` simply sets up the local relationships for the models you just created or updated:

```
resolveReferencesForItem(item) {
	item.clearReferences()
	let references = item.contentObject["references"]
	for reference in references {
		let referencedItem = Database.find(reference["uuid"]
		if referencedItem {
			item.addItemAsRelationship(referencedItem)
		}	
	}
}
```

And then in the individual model classes, you'd do something like this:

**Note**

```
addItemAsRelationship(item) {
	if item.contentType == "Tag" {
		self.addTag(item);
	}
	super.addItemAsRelationship(item);
}
```

Next, let's handle saving items to the server.

Add a field to your local models `dirty`. Anytime you make a change to an item, set `dirty` equal to `true`.

Then ask the ApiController to upload changes:

ApiController:

```
saveDirty() {
	let dirty = ItemManager.sharedInstance.fetchDirty()
	let itemParams = _map(dirty, function(item){
		return self.paramsForItem(item);
	})
	
	let request = Restangular.one("items/sync");
	request.items = itemParams;
	request.post().then(function(response(){
		let savedItems = response.saved_items;
		self.handleResponseItems(savedItems, metadataOnly: true)
	})
}
```

Upon completion, the server will return your saved items under the `saved_items` param. You don't want to merge the `content` from these items with your local models because your local models could have updated content. For example, imagine you're dealing with a Note. 

The user types the text "Hello", the app sync those changes. By the time the request is complete, the user has typed "World". If you merge the saved_items with your local items, then it will delete "World". For this reason, you should only merge metadata from saved items. This means every field _except_ for `content`, `enc_item_key`, and `auth_hash`.

**ApiController**

```
paramsForItem(item) {
	var params = {}
	params["content_type"] = item.contentType
	params["uuid"] = item.uuid
	params["presentation_name"] = item.presentationName
	params["deleted"] = item.deleted
	
	if item.isPublic() {
		// send decrypted
		params["enc_item_key"] = null
		params["auth_hash"] = null
		params["content"] = "000" + CryptoHelper.base64(message: item.createContentJSONStringFromProperties();
	} else {
		// send encrypted
		let encryptedParams = Crypto.encryptionParamsForItem(item);
		params.merge(encryptedParams);
	}
}
```

**Item**

```
createContentJSONStringFromProperties() {
	var params = self.structureParams()
	return JSON.stringify(params)
}
```

The base class Item will implement the `structureParams()` method, along with subclasses. The Item class will handle setting up references, while the individual subclasses will handle properties specific to that class.

**Item**

```
structureParams() {
	return ["references" : referencesParams()]
}

func referencesParams() {
    fatalError("This method must be overridden")
}
```

Then in individual subclasses, like **Note**:

```
structureParams() {
	let params = {
		"title" : self.title,
       "text" : self.text
	}

	// must merge with super
	params.mergeWith(super.structureParams())
	
	return params
}
    
// Returns an array of dictionaries of related item metadata (uuid and content type only).
referencesParams() {
    var references = []
    for tag in self.tags {
        references.append(["uuid" : tag.uuid, "content_type" : tag.contentType])
    }
    return references
}
```


Note that when sending an Item decrypted, you should null out `enc_item_key` and `auth_hash`. Make sure that these keys are present when sent to the server, but have a nil value.

**CryptoHelper**

```
encryptionParamsForItem(item) {
	var params = {}
	let message = item.createContentJSONStringFromProperties()
	if item.encItemKey == nil {
		self.createKeyForItem(item)
	}
	let keys = self.keysForItem(item)
	params["content"] = "001" + self.encrypt(message, keys["ek"])
	params["enc_item_key"] = item.encItemKey
	params["auth_hash"] = self.computeAuthHash(params["content"], keys["ak"])
	return params
}
```

When sending `content` unencrypted, we append a "000" to the base64 string to indicate that it is unencrypted. When sending encrypted, we append a "001" to the base64 string.

```
createKeyForItem(item) {
	let hex = self.generateRandomHexKey(size: 512)
	let encryptedKey = self.encrypt(hex, User.masterKey)
	item.encItemKey = encryptedKey
}

keysForItem(item) {
	let decryptedKey = self.decrypt(item.encItemKey, User.masterKey)
	let ek = decryptedKey.firstHalfOfString();
	let ak = decryptedKey.secondHalfOfString();
	return {"ek" : ek, "ak" : ak}
}
```

Here `ek` stands for encryption key, and `ak` stands for auth key, which is used to generate the auth_hash string, which verifies the authenticity of the message (that it hasn't been tampered with).

```
encrypt(message, hexKey) {
	let keyData = hexKey.toHexadecimalData()
	let resultData = AES128CBCEncrypt(message.toData(encoding: utf8), keyData);
	// encode the result in base64
	return resultData.toBase64String()
}
```

AES128CBCEncrypt is the encryption algorithm, which will depend on platform. 128 represents the block size. The parameters used for the AES function are:

- Mode: CBC
- Padding: Pkcs7

Note that no IV is used, since our keys are random and long enough. If your platform algorithm requires an IV be passed, just pass a blank string, or its data equivalent.

Next let's compute the authentication hash:

```
computeAuthHash(message, key) {
	let messageData = message.toData(encoding: utf8)
	let keyData = hexKey.toHexadecimalData()
	let result = HMAC256(messageData, keyData)
	return result.hexEncodedString()
}
```

Here `HMAC256` is just your platform's implementation of an HMAC function using SHA256.

That's it for saving items. 

Let's go over decryption for incoming items.

**CryptoHelper**

```
decryptResponseItems(responseItems) {
	for item in responseItems {
		if item["deleted"] == true {
			continue
		}
		
		let contentString = item["content"]
		
		// check if its encrypted - has "001" prefix
		if contentString.hasPrefix("001") {
			let keys = self.keysForItem(item["enc_item_key"])
			let ek = keys["ek"]
			let ak = keys["ak"]
			
			// verify auth hash before decrypting
			let computedAuthHash = HMAC256(contentString.toData(encoding: utf8), ak.toHexadecimalData())
			if computedAuthHash != item["auth_hash"] {
				print("Auth hash does not match, continuing")
				continue
			}
			
			let contentToDecrypt = contentString.substringFrom(3)
			let decryptedContent = self.decrypt(contentToDecrypt, ek)
			item["content"] = JSON.parse(decryptedContent)
		} else {
			// decrypted, but has "000" prefix
			// strip out prefix, then decode base64
			let contentToDecode = contentString.substringFrom(3)
			item["content"] = JSON.parse(Crypto.base64decode(contentToDecode))
		}	
	}
}
```

```
decrypt(base64String, hexKey) {
    let base64Data = Data(base64Encoded: base64String)
    let resultData = AES128CBCDecrypt(base64Data, hexKey.toHexadecimalData())
    let resultString = String(data: resultData!, encoding: .utf8)
    return resultString!
}  
```

And that's it for encryption/decryption.

Next, let's merge `refreshItems()` and `saveDirty()` into one function called `sync()`:

**ApiController**

```
sync() {
	let request = Restangular.one("items/sync");
	
	let dirty = ItemManager.sharedInstance.fetchDirty()
	let itemParams = _map(dirty, function(item){
		return self.paramsForItem(item);
	})
	
	request.items = itemParams;
	
	if(self.syncToken) {
		request.syncToken = self.syncToken;
	}
	
	request.post().then(function(response){
		self.syncToken = response.sync_token;
		self.handleResponseItems(response.retrieved_items);
		self.handleResponseItems(response.saved_items, metadataOnly: true)
	})
}
```

Done.

### Deleting an item

**ItemManager**

```
setItemToBeDeleted(item) {
    item.deleted = true
    item.dirty = true
}
```

Then sync.

After successful sync, delete the item from your local database.

### Presentations

Finally, let's talk about presentations. An item that is presented means it is public, and should not be encrypted.

Any item can be shared, or presented, simply by assigning it a `presentation_name` value. The server decides what the URL for this item would be given a presentation name. 

It is up to the client however to decide which items are related to a presented item, and therefore must also be decrypted.

For example, if you share a Tag, that Tag must be sent to the server decrypted, but also, any Notes belonging to that tag should also be sent decrypted.

To handle this logic, we simply define a function on the Item class:

**Item**

```
isPublic() {
    return self.presentationName != nil
}
```

Subclasses however should override this function if their definition of public is different. In the case of a Note, it is public if it has a presentation_name, OR if any of the tags it belongs to is public.

**Note**

```
isPublic() {
    return super.isPublic || self.hasOnePublicTag()
}

hasOnePublicTag() {
    for tag in self.tags {
        if tag.isPublic {
            return true
        }
    }
    return false
}
```

A Note can be public if it belongs to a public Tag, but it can also be public because it is shared individually AND it belongs to a private tag. To make sure you're displaying the right URL, you can also implement this function on a Note object:

**Note**

```
isSharedIndividually() {
	return self.presentationName != nil
}
```

That should be it.

## Next Steps

Join the [Slack](https://slackin-ekhdyygaer.now.sh/) group to discuss implementation questions.

You can also email [standardnotes@bitar.io](mailto:standardnotes@bitar.io) for any questions.

Check out the source code for other completed clients:

iOS: [https://github.com/standardnotes/iOS]()

Web: [https://github.com/standardnotes/web]()

For reference, you can also see the source for a standard Standard File server: 
[https://github.com/standardfile/ruby-server]()

Follow [@standardnotes](https://twitter.com/standardnotes) for updates and announcements.