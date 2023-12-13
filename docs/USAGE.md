# How to use MynahUI
In this document you'll find what functions `mynah-ui` provides and how to use them. Additionally you'll see some examples about handling the tabs and tab data, adding chat items and streams.

#### Important notes before you start:
If you didn't already, please take a look to the **[Startup Guide](./STARTUP.md)** for how to generate an instance of `mynah-ui`. All below examples and usage guides will assume that you have a constant called **`mynahUI`** to reach those functions.

Another important information about handling the events can be found under **[Constructor Properties](./PROPERTIES.md)** document.

To see how to configure statics for MynahUI please refer to **[Configuration](./CONFIG.md)** document.

Lastly before you start reading here, you can find more details on the **[Data Model](./DATAMODEL.md)** document. That document also contains visuals related with each type of the chat message in detail.


#### All publicly available functions
```typescript
mynahUI.addChatItem(...);
mynahUI.addToUserPrompt(...);
mynahUI.getSelectedTabId();
mynahUI.notify(...);
mynahUI.updateLastChatAnswer(...);
mynahUI.updateStore(...);
```

<p><br/></p>

---

<p><br/></p>

# Handling tabs
Even though you have the chance to partially handle the tabs, you should still want to keep track of the tabs in your app for various reasons like providing different controls/answers for different type of tabs. Since `MynahUI` doesn't have tab types, you should handle these kind of things on your own. (Also to keep the history and providing it back to the tab when you need.) **Since tab data flows can be asynchronous, you may want to keep track of which tab is doing what.** 

### Important Note:
You cannot close the tabs programmatically, but you can create new ones without user interaction. Closing a tab can only be done by a user interaction with the close button of the tab or middle clicking to the tab title. 

---

## Adding a new tab on runtime
Assume that you didn't set `tabs` property during the initialization of `mynahUI` and you want to add a new tab after the initialization completes. Or at some point, you want to create a new tab instead of using an existing one. Since everything on MynahUI runs through data store, you need to update the store by provind an empty string as the tabId. MynahUI will understand that there is no such tabId for an empty string and will create you a new one and return the newly created tab's id.  

```typescript
const mynahUI = new MynahUI({...});

const newTabId = mynahUI.updateStore('', {
  tabTitle: 'My new tab!'
});
```
But there is also one more important thing needs to be considered while generating a tab on the runtime without a user interaction, which is the `maxTabs` limit. (Details can be found under [Configuration](./CONFIG.md) document).

So if it reached to the max number of tabs can be opened, it will return undefined instead of returning the new tabId.

```typescript
const mynahUI = new MynahUI({...});

const newTabId: sring | undefined = mynahUI.updateStore('', {
  tabTitle: 'My new tab!'
});
if(typeof newTabId !== string){
  console.warn(`Couldn't open a new tab, max tabs reached.`);
} else {
  // ...
}
```
**Please refer to the [Data Model](./DATAMODEL.md) documentation for details on which object on tab data store refers to what and how they are working**

---

## Getting the selected tab id
To get the selected tab id, you can use the function described here. However if there is no (selected) tab, it will return `undefined`. Since if there are tab or even just a single tab, there should be at least one tab selected.

```typescript
const mynahUI = new MynahUI({...});

const selectedTabId = mynahUI.getSelectedTabId();
if(typeof selectedTabId !== string){
  console.warn(`No selected tab, which means that there is no tab.`);
} else {
  // ...
}
```
---

## Updating tab store (everything on that tab) on runtime
Assume that you didn't set `tabs` property during the initialization of `mynahUI` and you want to add a new tab after the initialization completes. Or at some point, you want to create a new tab instead of using an existing one. Since everything on MynahUI runs through data store, you need to update the store by provind an empty string as the tabId. MynahUI will understand that there is no such tabId for an empty string and will create you a new one and return the newly created tab's id.  

```typescript
const mynahUI = new MynahUI({...});

const selectedTabId = mynahUI.getSelectedTabId();
if(typeof selectedTabId !== string){
  console.warn(`No selected tab, which means that there is no tab.`);
} else {
  mynahUI.updateStore(selectedTabId, {
    tabTitle: 'Updated tab title',
    ...
  });
}
```
### Please refer to the [Data Model](./DATAMODEL.md) documentation for details for which object on tab data store refers to what and how they are working

<p><br/></p>

---

<p><br/></p>

# Handling chat items

There are two ways of adding chat items to the desired tab. However one of them is mainly being used to clean the chat items completely. So let's take a look to them both below. In additon to adding chat items, we'll also cover how you can stream a chat answer and handle the spinners during the stream.

---

## Adding Chat Items

### using `addChatItem` function
With addChatItem function, you're able to add a chat item onto the tab with given tabId. 

```typescript
const mynahUI = new MynahUI({
  tabs: {
    'tab-1': {
      isSelected: true,
      store: {}
    }
  }
});

mynahUI.addChatItem('tab-1', {
  type: ChatItemType.ANSWER,
  body: 'Body of the message',
  ...
});
```

### using `updateStore` function
With this method you can also append new chat items at once with the order they're given inside the chatItems array. But as defined, they will not replace the list, they will be appended. If you want to clear the whole chat items and add new ones, you need to send an empty list first. And send the new items through `addChatItem` one by one of updating the store with the new list.

```typescript
const mynahUI = new MynahUI({
  tabs: {
    'tab-1': {
      isSelected: true,
      store: {}
    }
  }
});

mynahUI.updateStore('tab-1', {
  chatItems: [
    {
      type: ChatItemType.ANSWER,
      body: 'Body of the message',
      ...
    },
    {
      type: ChatItemType.ANSWER,
      followUp: {
        text: 'Followups',
        options: [
          {
            pillText: 'Followup 1',
            prompt: 'followup 1',
            type: ''
          }
        ]
      }
      ...
    },
    ...
  ]
}); // these chat items will be appended.
```

**Note: if you want to clear all the chat items for that tab, send an empty array**

```typescript
mynahUI.updateStore('tab-1', {
  chatItems: []
});
```
**Please refer to the [Data Model](./DATAMODEL.md) documentation for types of chat items and how they appear differently on screen.**

---

## Creating and updating streaming chat items (`updateLastChatAnswer`)

You can only update the streaming chat items on the runtime and you can only have one streaming chat item at once. First let's create a streaming chat item. We'll also set the tab in loading state to be sure that we have streaming animations before and during the stream.

```typescript
const mynahUI = new MynahUI({
  tabs: {
    'tab-1': {
      isSelected: true,
      store: {}
    }
  }
});

// set the tab to loading state. It will automatically animate the streaming card
// Also disable the prompt field to avoid the user enter a new prompt before the stream ends.
// You can also inform the user with the placeholder of the prompt input about why they are not able to write a new prompt
mynahUI.updateStore('tab-1', {
  loadingChat: true,
  promptInputDisabledState: true,
  promptInputPlaceholder: `Please hold, i'm generating your answer`
});

mynahUI.addChatItem('tab-1', {
  type: ChatItemType.ANSWER_STREAM,
  body: '',
  ...
});
```

Now on UI, we have a streaming chat item. Before we start to stream, **there is an important thing to know that you should send the whole body everytime you update the stream**. MynahUI doesn't make it appended for you, it will just show the given content each time.

```typescript
mynahUI.updateLastChatAnswer('tab-1', {
  body: 'Hello there',
});

...
// After a moment
mynahUI.updateLastChatAnswer('tab-1', {
  body: `Hello there, I'm MynahUI.`,
});

...
// After a moment
mynahUI.updateLastChatAnswer('tab-1', {
  body: `Hello there, I'm MynahUI. I am a data and event driven`,
});

...
// After a moment
mynahUI.updateLastChatAnswer('tab-1', {
  body: `Hello there, I'm MynahUI. I am a data and event driven chat interface for web.`,
});

...
// After a moment
mynahUI.updateLastChatAnswer('tab-1', {
  followup: {
    text: 'What you can do more?',
    options: [
      {
        pillText: 'Show me examples',
        type: 'Show MynahUI examples',
      },
      {
        pillText: 'Which projects use MynahUI as their interface?',
        prompt: 'Which projects MynahUI is being used',
      }
    ],
  }
});

// Since the stream is ended, stop the spinner and enable the prompt input again.
mynahUI.updateStore('tab-1', {
  loadingChat: false,
  promptInputDisabledState: false,
  promptInputPlaceholder: 'Type your question here.'
});
```

Except setting the loading state back to false and enabling the prompt input back you don't need to do anything more to end a streaming. When you add a new chat item MynahUI will automatically release the last streaming chat answer and convert it to a normal answer. Which means that it will not be updated anymore. 

As you can update the body of a streaming card, you can also update the other information related with the card. Like adding the related sources etc. 
**Please refer to the [Data Model](./DATAMODEL.md) documentation for types of chat items and how they appear differently on screen.**

---

## Adding code attachments to prompt field (`addToUserPrompt`)

You can add code attachments under the prompt field of the desired tab. When user fills the prompt field and sends it, the attached code block will be appended at the end of the prompt text. It accepts max 4000 chars however you don't need to worry about it. MynahUI will automatically crop it depending on the available chars left from the prompt field itself.

```typescript
mynahUI.addToUserPrompt('tab-1', `
\`\`\`typescript
const a = 5;
\`\`\`
`);
```

<p align="center">
  <img src="./img/code-attachment.png" alt="mainTitle" style="max-width:500px; width:100%;border: 1px solid #e0e0e0;">
</p>

And when user sends the prompt attached code will be appended to the bottom.

<p align="center">
  <img src="./img/prompt-with-code-attached.png" alt="mainTitle" style="max-width:500px; width:100%;border: 1px solid #e0e0e0;">
</p>

<p><br/></p>

---

<p><br/></p>

# Notifications (`notify`)

You can give notifications through mynahUI which will appear inside the mynah ui root container block.

```typescript
mynahUI.notify({
  content: 'Hello there! Here is a test notification',
  duration: 5,
  title: 'This is a notification',
  type: NotificationType.INFO,
  onNotificationClick: () => {
    // when notification itself is clicked
    // do not forget that when notification is clicked, it closes automatically
  },
  onNotificationHide: () => {
    // when notification is closed
  },
});
```
<p align="center">
  <img src="./img/notification.png" alt="mainTitle" style="max-width:500px; width:100%;border: 1px solid #e0e0e0;">
</p>

To see the different notification types and how they look **please refer to the [Data Model](./DATAMODEL.md) documentation for details of the `NotificationProps`**
