---
title: "Building a Master-Detail Pattern in React Native"
categories:
  - "React Native"
tags:
  - TypeScript
---

I’m in the middle of writing a simple Notes app in React Native. Thus far, I’ve:

* [Worked out how to handle orientation changes]({% post_url 2017/2017-07-26-handling-orientation-changes-in-react-native %}).
* [Worked out how to implement swipe-to-delete]({% post_url 2017/2017-08-07-implementing-swipe-right-on-a-react-native-flatlist %}).
* [Figured out the best way to use TypeScript]({% post_url 2017/2017-08-09-debugging-react-native-with-typescript-and-vscode %}).
* [Integrated MobX for the Flux pattern]({% post_url 2017/2017-08-11-integrating-react-native-typescript-mobx %}).
* [Fixed up the project for universal iOS apps]({% post_url 2017/2017-08-14-universal-ios-apps-with-react-native %}).

Now it’s time to get to the master-detail pattern itself. Master-Detail is a basic pattern that incorporates a list (the master) and a detail page. On phones (and tablets in portrait mode), this is normally rendered as two separate screens. On tablets in landscape mode, it is rendered as a side-by-side arrangement.

I’m using a container component for this. The premise is that I will detect what size device and what sort of orientation it is in and then produce the right output. Let’s get started with the `index.tsx` file:

{% highlight typescript %}
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
        <MasterDetail/>
      </Provider>
    );
  }
}
{% endhighlight %}

I’m going to wrap all the logic inside of the `src/components/MasterDetail.tsx` component:

{% highlight typescript %}
@inject('noteStore')
@observer
export default class MasterDetail extends React.Component<MasterDetailProperties, MasterDetailState> {
    constructor(props: MasterDetailProperties) {
        super(props);
        this.state = {
            isLandscape: this.isLandscape()
        };

        // Not in typings - see https://github.com/DefinitelyTyped/DefinitelyTyped/pull/18885
        Dimensions.addEventListener('change', () => {
            this.setState({ isLandscape: this.isLandscape() });
        });
    }

    // ...
}
{% endhighlight %}

The MasterDetail object has two properties. One is the `noteStore` and is injected from MobX. The other is `forceTwoPane` – a boolean that can be used to force the two-pane mode if you need to. I don’t use it normally, but you can wire a button so that you can use two-pane mode in portrait mode on a tablet. The state contains the current orientation, and I wire up an event handler to adjust this so that the component will be re-rendered if the orientation changes. Note that the `addEventListener()` and `removeEventListener()` methods were not included in the react-native typings, so I published a pull request for these. Hopefully they will make it into an official npm package by the time you read this.

{% highlight typescript %}
isLandscape(): boolean {
    const dim = Dimensions.get('screen');
    return dim.width >= dim.height;
}

isTablet(): boolean {
    const msp = (dim: ScaledSize, limit: number): boolean => (dim.scale * dim.width) >= limit || (dim.scale * dim.height) >= limit;
    const dim = Dimensions.get('screen');
    return msp(dim, dim.scale < 2 ? 960 : 1800);
}

useTwoPane(): boolean {
    return this.props.forceTwoPane || (this.isTablet() && this.isLandscape());
}
{% endhighlight %}

The next set of methods determine if the interface should be in two-pane mode or not. These are taken directly from my work on [detecting orientation changes]({% post_url 2017/2017-07-26-handling-orientation-changes-in-react-native %}).

{% highlight typescript %}
onSelectItem(item: Note) {
    this.props.noteStore.setActiveNote(item);
}

onDeleteItem(item: Note) {
    this.props.noteStore.deleteNote(item);
}

onChangeItem(item: Note) {
    this.props.noteStore.saveNote(item);
}

onClearSelection() {
    this.props.noteStore.clearActiveNote();
}
{% endhighlight %}

I added an `activeNote` variable to my `noteStore` implementation. This is altered by `setActiveNote()` and `clearActiveNote()`. These event handlers adjust things in the store, which will then filter their way through the rest of the interface.

Finally, let’s look at the `render()` method. There are three cases to deal with:

* The app is in two-pane mode.
* The app is in one-pane mode and is displaying the master list.
* The app is in one-pane mode and is displaying the details page.

Each of these are a case. Theoretically, it would make for a better user experience if I used a `Navigator` object and `react-native-navigation` instead of three screens. If I did that, then the back button would be dealt with for me and the scenes would change by a swipe animation. However, I want to do things in the toolbar, which the `Navigator` pattern does not allow, so I’m happy to avoid the animation for now.

{% highlight typescript %}
render(): JSX.Element {
     if (this.useTwoPane()) {
         /*
          * BEGIN: Two-Pane Mode where the list is on the left and the details on the right
          */
         const activeNote = this.props.noteStore.getNote();
         const activeNoteTitle = activeNote === null ? <Text/>
             : <Text style={styles.onePaneHeaderTitle}>{activeNote.title}</Text>;
         const activeNoteView = activeNote === null ? <View/>
             : <NoteDetails item={activeNote} onChangeItem={(item: Note) => this.onChangeItem(item)} />

         return (
             <View style={styles.twoPaneContainer}>
                 <View style={styles.twoPaneLeft}>
                     <View style={styles.onePaneHeader}>
                         <View style={styles.onePaneHeaderLeftIconContainer}>
                         </View>
                         <View style={styles.onePaneHeaderTitleContainer}>
                             <Text style={styles.onePaneHeaderTitle}>Notes</Text>
                         </View>
                     </View>
                     <NoteList
                         items={this.props.noteStore.notes}
                         onSelectItem={(item: Note) => this.onSelectItem(item)}
                         onDeleteItem={(item: Note) => this.onDeleteItem(item)}
                     />
                 </View>
                 <View style={styles.twoPaneRight}>
                     <View style={styles.onePaneHeader}>
                         <View style={styles.onePaneHeaderLeftIconContainer}>
                         </View>
                         <View style={styles.onePaneHeaderTitleContainer}>
                             {activeNoteTitle}
                         </View>
                         <View style={styles.onePaneHeaderRightIconContainer}>
                             <TouchableHighlight onPress={() => this.onClearSelection()}>
                                 <Text style={styles.onePaneHeaderBackButton}>Done</Text>
                             </TouchableHighlight>
                         </View>
                     </View>
                     <View style={styles.onePaneContent}>
                         {activeNoteView}
                     </View>
                 </View>
             </View>
         );
         /*
          * END: Two-pane mode
          */
     }

     if (this.props.noteStore.activeNoteId === null) {
         /*
          * BEGIN: One-pane mode where the list is displayed
          */
         return (
             <View style={styles.onePaneContainer}>
                 <View style={styles.onePaneHeader}>
                     <View style={styles.onePaneHeaderLeftIconContainer}>
                     </View>
                     <View style={styles.onePaneHeaderTitleContainer}>
                         <Text style={styles.onePaneHeaderTitle}>Notes</Text>
                     </View>
                 </View>
                 <View style={styles.onePaneContent}>
                     <NoteList
                         items={this.props.noteStore.notes}
                         onSelectItem={(item: Note) => this.onSelectItem(item)}
                         onDeleteItem={(item: Note) => this.onDeleteItem(item)}
                     />
                 </View>
             </View>
         );
         /*
          * END: One-pane mode where the list is displayed
          */
     } else {
         /*
          * BEGIN: One-pane mode where the details are displayed
          */
         const activeNote = this.props.noteStore.getNote();
         return (
             <View style={styles.onePaneContainer}>
                 <View style={styles.onePaneHeader}>
                     <View style={styles.onePaneHeaderLeftIconContainer}>
                         <TouchableHighlight onPress={() => this.onClearSelection()}>
                             <Icon style={styles.onePaneHeaderBackButton} name="ios-arrow-back"/>
                         </TouchableHighlight>
                     </View>
                     <View style={styles.onePaneHeaderTitleContainer}>
                         <Text style={styles.onePaneHeaderTitle}>{activeNote.title}</Text>
                     </View>
                 </View>
                 <View style={styles.onePaneContent}>
                     <NoteDetails item={activeNote} onChangeItem={(item: Note) => this.onChangeItem(item)} />
                 </View>
             </View>
         );
         /*
          * END: One-pane mode where the details are displayed
          */
     }
 }
{% endhighlight %}

This is a longer code set, so I’ve marked the beginning and end of each section. Yes, a Master-Detail pattern is just an if-then-else statement with the appropriate logic at each step and distinguishable blocks.

I have abstracted the NoteList (without chrome) and NoteDetails (again, without chrome) into their own components so that I can re-use the components in both the one-pane and two-pane versions.

There are a lot of fiddly UI pieces in the Master-Detail that make it problematic to convert to a generic component. I haven’t, for example, added an “Add Item” button yet, and there are various “fit-and-finish” type UI changes that I want to do.

## Next Steps

Next on the list is local storage. I want to store the notes in persistent storage so that they are available next time the app is started. That will be the subject of my next blog post. 