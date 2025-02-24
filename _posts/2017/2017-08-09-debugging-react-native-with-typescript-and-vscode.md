---
title: "Debugging React Native with TypeScript and Visual Studio Code"
categories:
  - "React Native"
tags:
  - Debugging
  - TypeScript
---

One of the things I really miss from [React Native](https://facebook.github.io/react-native/) was the support for [TypeScript](http://www.typescriptlang.org/). TypeScript helps me immensely, but it really comes into its own with React programming as the PropTypes are specified for you (no more propTypes static). I’m also getting into [MobX](https://mobx.js.org/) as a flux implementation and that uses decorators, which is native in TypeScript. There is lots to love in TypeScript.

However, every single guide I saw for implementing TypeScript within React Native was flawed. Specifically, there was a need for a separate build step, so I could not just use Visual Studio Code to run a debug instance of my app.

Well, I’ve solved that and this is how I did it.

## Create an App

Use either `react-native init` to create your app. You can use `create-react-native-app` to create your app but there are a couple of extra steps that I am not going to cover here. If you use `create-react-native-app`, check [the transformer repo](https://github.com/ds300/react-native-typescript-transformer).

## Add TypeScript Packages

Use the following to add the appropriate packages:

{% highlight bash %}
yarn add --dev react-native-typescript-transformer typescript @types/react @types/react-native
{% endhighlight %}

The [first package](https://github.com/ds300/react-native-typescript-transformer) is the glue code that makes this all possible. After that, you should recognize the other packages as they are common for all TypeScript projects.

## Configure TypeScript

TypeScript has a [configuration file](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html) called `tsconfig.json`. The major parts of this are as follows:

{% highlight json %}
{
    "compilerOptions": {
        "target": "es2015",
        "module": "es2015",
        "jsx": "react-native",
        "moduleResolution": "node",
        "allowSyntheticDefaultImports": true
    }
}
{% endhighlight %}

You can add whatever other options you need to your TypeScript configuration at this point. I like things like `noImplicitAny` here, for example.

## Configure the React Native Packager

This (along with the `react-native-typescript-transformer` package) is the bit that does the magic – compiling your TypeScript files on the fly! Create a file called `rn-cli.config.js` with the following contents:

{% highlight js %}
module.exports = {
    getTransformModulePath() {
        return require.resolve('react-native-typescript-transformer');
    },
    getSourceExts() {
        return [ 'ts', 'tsx' ]
    }
};
{% endhighlight %}


You will also want to update the start script definition within `package.json`:

{% highlight json %}
"scripts": {
    "start": "react-native start --transformer node_modules/react-native-typescript-transformer/index.js --sourceExts ts,tsx",
    "test": "jest"
},
{% endhighlight %}


This will add the transformer when you run `npm start` instead of using the IDE.

## Write a TypeScript React Native component

I’ve created a folder called `src` that holds my TypeScript files. I’ve placed a component in `src/index.tsx` as follows:

{% highlight jsx %}
import React from 'react';
import { StyleSheet, Text, TextStyle, View, ViewStyle } from 'react-native';

interface Props {
}

interface State {
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  } as ViewStyle,
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  } as TextStyle,
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  } as TextStyle,
});

export default class App extends React.Component<Props, State> {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          {'To get started, edit src/index.tsx'}
        </Text>
      </View>
    );
  }
}
{% endhighlight %}

Yes, this is just plain old TypeScript+JSX, but using React Native components.

## Wire up your TypeScript component

Before we continue, there is a bunch of code in the `index.ios.js` and `index.android.js` files. Unless you want to hack the iOS and Android platform code, it’s best to leave these as JavaScript code. However, you can make them minimal. Both of mine are identical:

{% highlight js %}
import { AppRegistry } from 'react-native';
import App from './src';

AppRegistry.registerComponent('masterdetailtemplate', () => App);
{% endhighlight %}

## Add React Native Launch Controls and Debug

Click into the Debug area of Vistual Studio code and create the React Native launch.json file. Select the appropriate target (I use **Debug iOS** to run the iOS simulator). Then click on the green start button.
