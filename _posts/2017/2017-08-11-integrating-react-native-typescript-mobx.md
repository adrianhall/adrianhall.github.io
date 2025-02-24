---
title: "Integrating React Native, TypeScript, and MobX"
categories:
  - "React Native"
tags:
  - TypeScript
---

In my last article, I psted about [getting TypeScript working with React Native]({% post_url 2017/2017-08-09-debugging-react-native-with-typescript-and-vscode %}). I’m building a flexible, best-practices, Notes App in React Native. This means I need a backing store, and it has to be local for offline capabilities. React has a definite way of building data into the UI and the manipulation of that data is an architecture known as [Flux](http://facebook.github.io/flux/). Flux isn’t a concrete implementation, however. Normally, I would use [Redux](http://redux.js.org/) as the concrete implementation. However, I have recently started working with [MobX](https://mobx.js.org/) and I prefer it. This article is about integrating MobX into my application for the storage of the Notes data.

## Step 1: Install Packages

MobX splits its functionality for React and React Native across two packages – the mobx package contains all the non-specific stuff and the mobx-react package contains the bindings for React and React Native:

{% highlight bash %}
yarn add mobx mobx-react
{% endhighlight %}

## Step 2: Enable Decorators

MobX uses [JavaScript decorators](https://github.com/wycats/javascript-decorators) to specify how the store is linked up to the components in your React tree. TypeScript supports decorators, which is a good thing. However, you have to enable it. Edit the tsconfig.json file and add the appropriate line:

{% highlight json %}
{
    "compilerOptions": {
        "target": "es2015",
        "module": "es2015",
        "jsx": "react-native",
        "moduleResolution": "node",
        "allowSyntheticDefaultImports": true,
        "experimentalDecorators": true,
        "noImplicitAny": true
    }
}
{% endhighlight %}

Once this is done, you may want to restart Visual Studio Code if you are using it. Visual Studio Code does not generally pick up changes in the `tsconfig.json` file so you may notice some red squiggly lines for decorators until you restart.

## Step 3: Write a Model

I’m using a small model file to define the shape of my data. Create a file called `src/models/Note.ts` with the following content:

{% highlight typescript %}
/**
 * Model for the Note
 */
export default interface Note {
    noteId: string,
    title: string,
    content: string,
    createdAt: number,
    updatedAt: number
}
{% endhighlight %}

## Step 4: Write a Store

The observable store is the MobX version of the Flux state store. We can use TypeScript to add type annotations and use the MobX decorators to make the store observable. This is my `src/stores/noteStore.ts` file:

{% highlight typescript %}
import { observable } from 'mobx';
import Note from '../models/Note';

class NoteStore {
    @observable notes: Note[] = [];

    saveNote(note: Note) {
        const idx = this.notes.findIndex((n) => note.noteId === n.noteId);
        if (idx < 0) {
            this.notes.push(note);
        } else {
            this.notes[idx] = note;
        }
    }

    deleteNote(note: Note) {
        const idx = this.notes.findIndex((n) => n.noteId === note.noteId);
        if (idx < 0) {
            throw new Error(`Note ${note.noteId} not found`);
        } else {
            this.notes.splice(idx, 1);
        }
    }

    getNote(noteId: string): Note {
        const idx = this.notes.findIndex((n) => n.noteId === noteId);
        if (idx < 0) {
            throw new Error(`Note ${noteId} not found`);
        } else {
            return this.notes[idx];
        }
    }
}

const observableNoteStore = new NoteStore();

const newNote = (title: string, content: string) => {
    const note = {
        noteId: uuid.v4(),
        title: title,
        content: content,
        updatedAt: Date.now(),
        createdAt: Date.now()
    };
    observableNoteStore.saveNote(note);
}

newNote('First Note', 'some content');
newNote('2nd Note', 'some content');
newNote('3rd Note', 'some content');
newNote('4th Note', 'some content');

export default observableNoteStore;
{% endhighlight %}

## Step 5: Write some container components

Since this is going to be a master-detail template, I want to write some common pages. For example, I’m going to write a NoteList component that takes a set of items and displays them, and I’m going to create a NoteListPage that wraps the Note List appropriately for a one-pane view of the NoteList. I’ve previously posted about [the NoteList component]({% post_url 2017/2017-08-07-implementing-swipe-right-on-a-react-native-flatlist %}). The `NoteListPage` looks like the following:

{% highlight jsx %}
import React from 'react';
import { Platform, StyleSheet, View, ViewStyle } from 'react-native';
import { observer, inject } from 'mobx-react/native';
import { NoteStore } from '../stores/noteStore';
import Note from '../models/Note';
import NoteList from './NoteList';

const styles = StyleSheet.create({
    container: {
        marginTop: Platform.OS === 'ios' ? 20 : 0
    } as ViewStyle
});

interface NoteListPageProperties {
    /**
     * The store reference for the notes store.  Note that this needs to be optional
     * because the <Provider> component adjusts things appropriately, which the
     * code checker won't pick up on.
     *
     * @type {NoteStore}
     * @memberof NoteListPageProperties
     */
    noteStore?: NoteStore
}

@inject('noteStore')
@observer
export default class NoteListPage extends React.Component<NoteListPageProperties> {
    onDeleteItem(item: Note): void {
        this.props.noteStore.deleteNote(item);
    }

    render() {
        return (
            <View style={styles.container}>
                <NoteList
                    items={this.props.noteStore.notes}
                    onDeleteItem={(item: Note) => this.onDeleteItem(item)}
                />
            </View>
        );
    }
}
{% endhighlight %}

Line 26 injects the `noteStore` provided by the Provider object (more on that in a minute) into the props for this component. It will be available as `this.props.noteStore`. Line 27 adds code to re-render the component when the observed store changes. The code inside the container component creates a list and links the `onDeleteItem` (which is the swipe-to-delete) to the stores `deleteNote()` method. If I swipe to delete, it will effect a change in the store that will then cause the container to re-render because the observed element (the notes) drive the list. I could also add an `onSelectItem()` to this, but I haven’t added routing to this application yet, and this would be more of a state change than a store change, so it isn’t germane to the MobX functionality.

## Step 6: Wire the store to the components with the Provider

In my `index.tsx` file, I need to link the `noteStore` to the stack of React components. This is done with the `Provider` component:

{% highlight jsx %}
import React from 'react';
import { StyleSheet, Text, TextStyle, View, ViewStyle } from 'react-native';
import { Provider } from 'mobx-react/native';
import noteStore from './stores/noteStore';
import NoteListPage from './components/NoteListPage';

/**
 * Production Application Component - this component renders the rest of the
 * application for us.
 *
 * @export
 * @class App
 * @extends {React.Component<undefined, undefined>}
 */
export default class App extends React.Component<undefined, undefined> {
  /**
   * Lifecycle method that renders the component - required
   *
   * @returns {React.Element} the React Element
   * @memberof App
   */
  render() {
    return (
      <Provider noteStore={noteStore}>
        <NoteListPage/>
      </Provider>
    );
  }
}
{% endhighlight %}

Note that the Provider has an argument (called noteStore) that is assigned the value noteStore. It is important that the argument name is the same as the string value used in the inject statement from Step 5. Your app will replace the NoteListPage in this example. I use this format to design my page container components. I can replace the NoteListPage with NoteListDetail, for example, to ensure that the display is appropriate for what I am trying to do.

## Next Steps

Now that I have the MobX store working, I am going to move onto getting the two-pane version of the application working. I’ll show this in the next article. 
