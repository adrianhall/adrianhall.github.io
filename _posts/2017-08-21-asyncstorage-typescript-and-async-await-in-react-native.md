---
title: "AsyncStorage, TypeScript, and Async/await in React Native"
categories:
  - "React Native"
tags:
  - "Async Programming"
  - TypeScript
---

In the last few posts, I’ve been working on a Notes app in a master-detail pattern for React Native using TypeScript. This is the last post about that project. So far, I’ve:

* [Worked out how to handle orientation changes]({% post_url 2017-07-26-handling-orientation-changes-in-react-native %}).
* [Worked out how to implement swipe-to-delete]({% post_url 2017-08-07-implementing-swipe-right-on-a-react-native-flatlist %}).
* [Figured out the best way to use TypeScript]({% post_url 2017-08-09-debugging-react-native-with-typescript-and-vscode %}).
* [Integrated MobX for the Flux pattern]({% post_url 2017-08-11-integrating-react-native-typescript-mobx %}).
* [Fixed up the project for universal iOS apps]({% post_url 2017-08-14-universal-ios-apps-with-react-native %}).
* [Finally written the Master-Detail pattern]({% post_url 2017-08-16-building-a-master-detail-pattern-in-react-native %}).

That’s a lot of stuff. This last post is about storage. React Native provides a method of storing data on the device called [AsyncStorage](https://facebook.github.io/react-native/docs/asyncstorage.html). It follows the [Storage](https://developer.mozilla.org/en-US/docs/Web/API/Storage) API that is fairly well known in the JavaScript world. It’s promise driven and generally backed by SQLite, which is a nice performant on-device storage that is common on both iOS (as Core Data) and Android.

Some housekeeping first. I wired up a new icon that looks like a plus sign to the following event handler in `MasterDetail.tsx`:

{% highlight typescript %}
/**
 * Event handler called when the user clicks on the Add Item button
 * @memberof MasterDetail
 */
onAddItem() {
    let item = {
        noteId: uuid.v4(),
        title: '',
        content: '',
        createdAt: Date.now(),
        updatedAt: 0
    };
    this.props.noteStore.saveNote(item);
    this.props.noteStore.setActiveNote(item);
}
{% endhighlight %}

This creates a new note and then sets it as active. Once the user enters some information, it will be stored in the store. Now, onto the AsyncStorage stuff. The best practice suggested is to wrap the AsyncStorage class with your own class. I can use the wrapping to implement some logic for storing the data locally. Here is the `LocalStorage.ts` file:

{% highlight typescript %}
import { AsyncStorage } from 'react-native';
import Note from '../models/Note';

/**
 * Deals with the local storage of Notes into AsyncStorage
 *
 * @class LocalStorage
 */
class LocalStorage {
    /**
     * Get a single item
     *
     * @param {string} noteId
     * @returns {Promise<Note>}
     * @memberof LocalStorage
     */
    async getItem(noteId: string): Promise<Note> {
        return AsyncStorage.getItem(`@note:${noteId}`)
        .then((json) => {
            return JSON.parse(json) as Note;
        });
    }

    /**
     * Save a single item
     *
     * @param {Note} item
     * @returns {Promise<void>}
     * @memberof LocalStorage
     */
    async setItem(item: Note): Promise<void> {
        return AsyncStorage.setItem(`@note:${item.noteId}`, JSON.stringify(item));
    }

    /**
     * Deletes a single item
     *
     * @returns {Promise<void>}
     * @memberof LocalStorage
     */
    async deleteItem(noteId: string): Promise<void> {
        return AsyncStorage.removeItem(`@note:${noteId}`);
    }

    /**
     * Get all the items
     *
     * @returns {Promise<Note[]>}
     * @memberof LocalStorage
     */
    async getAllItems(): Promise<Note[]> {
        return AsyncStorage.getAllKeys()
        .then((keys: string[]) => {
            const fetchKeys = keys.filter((k) => { return k.startsWith('@note:'); });
            return AsyncStorage.multiGet(fetchKeys);
        })
        .then((result) => {
            return result.map((r) => { return JSON.parse(r[1]) as Note; });
        });
    }
};

const localStorage = new LocalStorage();
export default localStorage;
{% endhighlight %}

This introduces a new concept for JavaScript: async and await. These are basically markers for a Promise. Marking a method as “async” says “this returns a Promise”. Since this is TypeScript, I’m specifying the return type anyway and it’s obvious it returns a Promise.

There is a flip side to this, which is to make the calling method use await, like this:

```typescript
const note = await localStorage.getItem(noteId);
```

You can only use await inside of an async method, so the promise bubbles up to the top. I’ve got three items – `getItem()`, `deleteItem()` and `saveItem()` to do the normal CRUD elements. I’ve also got a `getAllItems()` that fetches all the notes from the store. I go to some lengths to ensure that only JSON objects for notes end up in the notes table, and I don’t deal with exceptions (I should do this!).

In my `noteStore.ts`, I use this LocalStorage class like this:

{% highlight typescript %}
initializeNotes(notes: Note[]) {
    if (this.notes.length > 0) {
        this.notes.splice(0, this.notes.length);
    }
    this.notes.push(...notes);
}

saveNote(note: Note) {
    console.log(`NoteStore:saveNote(${note.noteId})`);
    const idx = this.notes.findIndex((n) => note.noteId === n.noteId);
    if (idx < 0) {
        this.notes.push(note);
    } else {
        this.notes[idx] = note;
    }
    localStorage.setItem(note);
}

deleteNote(note: Note) {
    console.log(`NoteStore:deleteNote(${note.noteId})`);
    const idx = this.notes.findIndex((n) => n.noteId === note.noteId);
    if (idx < 0) {
        throw new Error(`Note ${note.noteId} not found`);
    } else {
        this.notes.splice(idx, 1);
        localStorage.deleteItem(note.noteId);
        if (note.noteId === this.activeNoteId) {
            this.activeNoteId = null;
        }
    }
}
{% endhighlight %}

Note that I don’t deal with the promise – it just stores the data asynchronously and I move about my day. If I were to do this “properly”, I would have a queue of data and a queue processor that was based on a service worker that processed the queue. This would prevent a race condition within the code where the app shuts down before the save to storage happens. However, the volume of data that is being stored is so low, I’m viewing this as a very low probability.

The store now stores the notes to local storage, but I don’t have anything to read the local storage on startup. I do have the `initializeNotes()` method that cleans out the notes array and replaces it with another notes array. Note that I edit the array in-situ rather than creating a new array. I’m honestly not sure this is worth it in the observable world, but it can’t hurt anything. My notes initialization is done at the bottom of the `notesStore.ts` file:

```typescript
const observableNoteStore = new NoteStore();
localStorage
    .getAllItems()
    .then(items => observableNoteStore.initializeNotes(items));
```

This is a standard promise pattern. The `getAllItems()` method resolves to the list of notes from the local storage, and I use that to populate the notes in my in-memory store.

That’s it for this series.  Now I’m going to use this knowledge to produce a prettier version of the Notes app!
