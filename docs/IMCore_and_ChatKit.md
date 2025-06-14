# IMCore and ChatKit
&nbsp;&nbsp;&nbsp;&nbsp; I'm writing this to detail what I've learned about IMCore and ChatKit over the course of this project. When I started this project (the situation my be different now), there were no modern open-source projects that correctly used IMCore and ChatKit to accomplish anything similar to what I was trying to do. So I hope that this can act as a guide for anyone hoping to accomplish a similar thing. \
&nbsp;&nbsp;&nbsp;&nbsp; There will be a good amount of code in this article; it'll be in [Logos](http://iphonedevwiki.net/index.php/Logos), since that's what I used when writing this project. You may want to learn its syntax first to fully understand what I'm trying to explain in this article.  \
&nbsp;&nbsp;&nbsp;&nbsp; I would also like to clarify, before starting this, that these are almost definitely not the best methods to accomplish these things. They are just the ones that I found work well for me. There are almost definitely better methods out there, but I have yet to discover them, and the following work well enough for me. With that out of the way, let's get into it!

## Connecting to IMCore
&nbsp;&nbsp;&nbsp;&nbsp; When starting this project, I initially set up an IPC Center in the `MobileSMS.app` on iOS to send texts (using [libmryipc](https://github.com/muirey03/mryipc)), since it seemed to be the only process that could do what I needed (e.g. access `[CKConversationList sharedConversationList]`, call `[CKConversation sendMessage:]`, etc). However, when iOS 14 rolled around, this started causing some issues, and I couldn't send any texts or get any of my commands to be processed by `MobileSMS.app` until I re-entered the app. After some testing and help from other developers (shoutout to @ericrabil on twitter), I realized that this was because `MobileSMS.app` was being suspended in the background, which is why it wasn't doing anything when I sent a message to the IPC Center. \
&nbsp;&nbsp;&nbsp;&nbsp; After some more poking around, I found [this conversation on reddit](https://www.reddit.com/r/jailbreakdevelopers/comments/464gbh/how_can_i_load_all_ckconversations_from_another/), which had exactly what I needed: instructions on how to connect directly to imagent on iOS. If you don't want to read the whole thread, the tl;dr is that the imagent daemon on iOS checks permissions for each process that tries to access IMCore or ChatKit classes/methods, and returns the values or nothing (depending on the permissions). If you'd like to get full permissions to use anything in ChatKit & IMCore, all you have to do is the following:

```objectivec
%hook IMDaemonController

- (unsigned)_capabilities {
	return 17159;
}

%end
```

On iOS 16 and newer, this changed to:
```objectivec
%hook IMDaemonController

- (unsigned long long)processCapabilities {
    return 4485895;
}

%end
```

&nbsp;&nbsp;&nbsp;&nbsp; This allows every process on the phone to use IMCore & ChatKit methods. You can also check the process that's calling that to conditionally return `17159` or `%orig`, e.g. in libsmserver I use:

```objectivec
%hook IMDaemonController

- (unsigned)_capabilities {
	NSString *process = [[NSProcessInfo processInfo] processName];
	if ([process isEqualToString:@"SpringBoard"] || [process isEqualToString:@"MobileSMS"])
		return 17159;
	else
		return %orig;
}

%end
```

&nbsp;&nbsp;&nbsp;&nbsp; Since I run my tweak in SpringBoard and `MobileSMS.app`, this gives me the permissions I want without creating a security hole for 3rd-party apps to access your data.

&nbsp;&nbsp;&nbsp;&nbsp; Now, once you've hijacked `IMDaemonController` to give you full access, you still need to connect to the daemon to actually requests through it. For this, I used the following:

```objectivec
/// Get the sharedController
IMDaemonController* controller = [%c(IMDaemonController) sharedController];

/// Attempt to connect directly to the daemon
if ([controller connectToDaemon]) {
	/// Do your IMCore/ChatKit stuff here
	/// e.g. send a text, send a tapback, create a new conversation, etc
} else {
	/// If it failed to connect to the daemon for whatever reason
	NSLog(@"Couldn't connect to daemon :(");
}
```

Now, this is the basic context that you need to interface with IMAgent and run your IMCore/ChatKit code. Now that we've got that out of the way, just assume that the rest of the code in this article is being run within the `if` block above unless it explicitly states otherwise.

## Sending a text
&nbsp;&nbsp;&nbsp;&nbsp; This was, by far, the most crucial function to get working. A web interface to simply browse your texts but not send any wouldn't get anyone's interest. This also did take me a significantly long time to get working, but that was due mainly to my own inability to read. [This page](https://iphonedevwiki.net/index.php/ChatKit.framework) on the iPhoneDevWiki has basic information about how to send a text to a pre-existing conversation, but when I read through it, I missed the section that said *"Note that this will still only work from the MobileSMS process, even though `[CKConversationList sharedConversationList]` is NOT always null in other processes."* Once I set up IPC with libmryipc, I was able to send information through the IPC to the MobileSMS process to send the text. Now, however, since I connect directly to the IMDaemon, I don't need to inject into the MobileSMS process to send a text. \
&nbsp;&nbsp;&nbsp;&nbsp; Now, the above linked page in the iPhoneDevWiki does have instructions on how to send a text through ChatKit, but in case that page gets taken down or the instructions are no longer available there for whatever reason, here's the code:

```objectivec
/// The [CKMediaObjectManager mediaObjectWithFileURL:] method must be run on the main thread or the process crashes
dispatch_async(dispatch_get_main_queue(), ^{
	//Get the shared conversation list
	CKConversationList* conversationList = [%c(CKConversationList) sharedConversationList];
	//Get the conversation for an address; "1111111" would be the receiver's phone number
	CKConversation *conversation = [conversationList conversationForExistingChatWithGroupID:@"11111111"];

	//Make a new composition. the text and subject must be an NSAttributedString.
	NSAttributedString* text = [[NSAttributedString alloc] initWithString:@"Hello friend"];
	CKComposition* composition = [[%c(CKComposition) alloc] initWithText:text subject:nil];

	//Add attachment (if desired).
	NSURL* fileUrl = [NSURL URLWithString:@"file:///var/mobile/Media/DCIM/100APPLE/IMG_0001.JPG"];

	CKMediaObjectManager* objManager = [%c(CKMediaObjectManager) sharedInstance];
	CKMediaObject* object = [objManager mediaObjectWithFileURL:fileUrl filename:nil transcoderUserInfo:nil attributionInfo:@{} hideAttachment:NO];
	composition = [composition compositionByAppendingMediaObject:object];

	//A new message from the composition
	CKMessage* message = [conversation messageWithComposition:composition];
	//And finally, send the message in the conversation
	[conversation sendMessage:message newComposition:YES];
});
```

&nbsp;&nbsp;&nbsp;&nbsp; If you'd prefer not to use ChatKit, but rather pure IMCore, I've found another way to send a message using only IMCore, but I haven't yet figured out how to attach attachments to the message. I'll update this document once I do figure that out (if I ever do). Here's the code:

```objectivec
__NSCFString *address = (__NSCFString *)@"+11231231234"; /// Must have the full phone number. just "1231234" wont work.
IMChatRegistry* registry = [%c(IMChatRegistry) sharedInstance];
IMChat* chat = [registry existingChatWithChatIdentifier:address];

if (chat == nil) { /// If you havent yet texted them
	/// Get your own account; must use it to register their conversation in your phone
	IMAccountController *sharedAccountController = [%c(IMAccountController) sharedInstance];
	IMAccount *myAccount = [sharedAccountController mostLoggedInAccount];

	/// Create their handle
	IMHandle *handle = [[%c(IMHandle) alloc] initWithAccount:myAccount ID:address alreadyCanonical:YES];

	/// Use the handle to get the IMChat
	chat = [registry chatForIMHandle:handle];
}

IMMessage *message;

/// iOS 14 requires the 'threadIdentifier' parameter, iOS13- doesn't support it.
if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 14.0)
	message = [%c(IMMessage) instantMessageWithText:text flags:1048581 threadIdentifier:nil];
else
	message = [%c(IMMessage) instantMessageWithText:text flags:1048581];

/// Send the message :)
[chat sendMessage:message];
```

&nbsp;&nbsp;&nbsp;&nbsp; If you haven't yet texted the person in the previous block of code (the one using `CKConversation`), the `CKConversation` from `[CKConversationList conversationForExistingChatWithGroupID]` will be nil. Just check for that, then execute the code in the most previous block, assuming that `chat` is nil (so that you have to go into the if and create a new chat).

## Typing indicators
&nbsp;&nbsp;&nbsp;&nbsp; This isn't the most elegant solution, but it isn't that bad either. In a nutshell, whenever `isCancelTypingMessage` returns `YES`, someone stopped typing, and whenever `isIncomingTypingMessage` returns `YES`, someone started typing. So if you check their values every time, you can do whever you want each time they return `YES`. The full code is below.

```objectivec
%hook IMMessageItem

- (bool)isCancelTypingMessage {
	bool orig = %orig;

	/// if `orig` is true here, someone stopped typing.
	if (orig) {

		/// warning: the `IMMessageItem` that this function is running in gets deallocated very quickly after when
		/// this function is called and returns `YES`, so if you do any dispatch_async stuff here and try to call
		/// anything on `self` in the block, it will crash the process this is running in.
		__block NSString* sender = [self sender];
		/// do whatever you want with the conversation in which the other party stopped typing
	}

	return orig;
}

- (bool)isIncomingTypingMessage {
	bool orig = %orig;

	/// if `orig` is true here, somebody started typing.
	if (orig) {

		__block NSString* sender = [self sender];
		/// `sender` is the chat identifier of the conversation in which the other party started typing.
		/// you can do whatever you want with it here.
	}
	return orig;
}

%end
```

&nbsp;&nbsp;&nbsp;&nbsp; The `IMTypingChatItem` is the typing indicator that appears in a conversation when they start typing. This init method is called even when the app is in the background, so as long as MobileSMS is running, this method theoretically works.  \
&nbsp;&nbsp;&nbsp;&nbsp; The method to notify someone else that you're typing is also pretty simple. All you'll need is the chat identifier of the conversation that you want to set as typing. Personally, I get the chat identifier from the sqlite database (`/var/mobile/Library/SMS/sms.db`), but I'm certain there are other ways. Here's the code:

```objectivec
CKConversationList* sharedList = [%c(CKConversationList) sharedConversationList];
/// Get the conversation relating to the phone number.
/// Change +11231231234 to whatever chat identifier you want
CKConversation* convo = [sharedList conversationForExistingChatWithGroupID:@"+11231231234"];

/// Change "YES" to "NO" if you want to set yourself as not typing
[convo setLocalUserIsTyping:YES];
```

## Sending a tapback
&nbsp;&nbsp;&nbsp;&nbsp; Although this one took me a while to figure out, I finally cracked it and included this feature in version 0.6.2 of SMServer. For this to work, you need the `chat_identifier` of the conversation which you want to send the tapback in, the `guid` of the message to which the tapback is reacting (so to speak), and the corresponding int value to the type of tapback you want to send (see code snippet below for specifics). In this case, `address` is the `chat_identifier`, `guid` is the `guid`, and `tapback` is the tapback value. Assuming that everything works perfectly the first time you call it, here's the code you need:

```objectivec
NSString *address = @"11231231234";
NSString *guid = @"12345678-1234-1234-1234-123456789012";
/*
Love tapback: send: 2000, remove: 3000
Thumbs up: 2001, 3001
Thumbs down: 2002, 3002
Haha: 2003, 3003
Exclamation: 2004, 3004
Question: 2005, 3005
*/
long long int tapback = 2000;

IMChat *chat = [[%c(IMChatRegistry) sharedInstance] existingChatWithChatIdentifier:address];

[chat loadMessagesUpToGUID:guid date:nil limit:nil loadImmediately:YES];
IMMessageItem *item = [chat messageItemForGUID:guid];

IMTextMessagePartChatItem *pci = [[%c(IMTextMessagePartChatItem) alloc] _initWithItem:item text:[item body] index:0 messagePartRange:NSMakeRange(0, [[item body] length]) subject:[item subject]];

if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 14.0)
	[chat sendMessageAcknowledgment:tapback forChatItem:pci withAssociatedMessageInfo:@{@"amc": @1, @"ams": [[item body] string]}];
```

&nbsp;&nbsp;&nbsp;&nbsp; Many of the methods in the above snippet regularly return `nil`, most commonly `[chat messageItemForGUID:]` (which you sometimes have to run up to 100 times before it actually gives you the message item), and `[chat loadMessagesUpToGUID:...]` (which sometimes just doesn't load the messages). If you are to make this reliable, you'll probably need to add quite a lot of error checking.

&nbsp;&nbsp;&nbsp;&nbsp; I don't yet know how to send tapbacks on iOS 13-, since the last method used in the code snippet above (`[chat sendMessageAcknowledgment: forChatItem: withAssociatedMessageInfo:]`) doesn't exist in anything before iOS 14. However, there are very similar methods, so I'm fairly certain that it wouldn't be too hard to figure it out. Let me know if you do, and I can add the information here (if you'd like).

## Getting pinned chats
&nbsp;&nbsp;&nbsp;&nbsp; I am actually fairly certain this is the best way to get the list of pinned chats. It's not as short and easy as I previously thought, but it's not too bad either. The following code gets us an array of all the chat identifiers that correspond to the currently pinned chats.

```objectivec
/// Pinned chats are only available for iOS 14+, so check that first
if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 14.0) {
	IMPinnedConversationsController* pinnedController = [%c(IMPinnedConversationsController) sharedInstance];
	NSArray* pins = [[pinnedController pinnedConversationIdentifierSet] array];

	CKConversationList* list = [%c(CKConversationList) sharedConversationList];
	NSMutableArray* convos = [NSMutableArray arrayWithCapacity:[pins count]];

	/// So `pins` contains an array of pinning identifiers, not chat identifiers. So we have to iterate through
	/// and get the chat identifiers that correspond with the pinning identifiers, since SMServer parses them by chat identifier.
	for (id obj in pins) {
		CKConversation* convo = (CKConversation *)[list conversationForExistingChatWithPinningIdentifier:obj];
		if (convo == nil) continue;
		NSString* identifier = [[convo chat] chatIdentifier];
		[convos addObject:identifier];
	}

	return convos;
}

return [NSArray array];
```

## Receiving texts
&nbsp;&nbsp;&nbsp;&nbsp; There are two easy methods to achieve this: they both should work just fine, but the second requires you to leave the MobileSMS app running, while the first one doesn't. The first one simply requires adding a notification listener for the name `__kIMChatReceivedNotification`. Here's a code snippet to demonstrate:

```objectivec
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(receivedText:) name:@"__kIMChatMessageReceivedNotification" object:nil];

- (void)receivedText:(NSConcreteNotification *)notif {
	IMMessage *msg = [[notif userInfo] objectForKey:@"__kIMChatValueKey"];
	NSString* guid = [msg guid];
	/// I use the guid of the message that was just received, but since you can get an
	/// IMMessage, you can use any of the details of that that you'd like.
}
```

&nbsp;&nbsp;&nbsp;&nbsp; Once again, this second method requires that the MobileSMS be running at least in the background the entire time, so I wouldn't recommend it. However, if you would like to use it regardless, here it is:

```objectivec
%hook SMSApplication

- (void)_messageReceived:(id)arg1 {
	%orig;

	IMChat* chat = (IMChat *)[(NSConcreteNotification *)arg1 object];
	NSString* identifier = chat.identifier;
	/// Do whatever you want with the identifier; it's the `chat_identifier`
	/// of the conversation that just sent the text
}

%end
```

## Setting a conversation as read
&nbsp;&nbsp;&nbsp;&nbsp; I'm also fairly certain that I know the best way to do this; it's very straightforward (you just need the chat identifier for the conversation which you want to mark as read), so here's the code:

```objectivec
/// Get the conversation
IMChat* imchat = [[%c(IMChatRegistry) sharedInstance] existingChatWithChatIdentifier:(__NSCFString *)@"+11231231234"];
/// mark it as read!
[imchat markAllMessagesAsRead];
```

You can also mark a single message as read, though you have to load the messages in the IMChat object first like this (otherwise messageForGUID returns nil sometimes):

```objectivec
for (int x = 0; x < 4 && !msg; x++) {
	[imchat loadMessagesUpToGUID:full_guid date:nil limit:0 loadImmediately:YES];
	for (int i = 0; i < 500 && !msg; i++)
		msg = [imchat messageForGUID:full_guid];
}
[imchat markMessageAsRead:msg];
```

&nbsp;&nbsp;&nbsp;&nbsp; That's all it is :&#12;) Though you should note that marking the latest message as read this way marks all older messages in the conversation read as well. This method also automatically handles sending read receipts for you (if you have them turned on; obviously it doesn't send them if you have them turned off for this conversation), so you don't have to worry about manually doing that.

## Thanks for reading :&#12;)
&nbsp;&nbsp;&nbsp;&nbsp; I would appreciate anything that anyone could add to this document. I'd love to compile as much information about these frameworks as I can.
