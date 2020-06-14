---
title: "Build a better RecyclerView Adapter"
categories:
  - Android
tags:
  - Kotlin
---

Many of my Android apps end up including listst, which are implemented via a [RecyclerView](https://developer.android.com/guide/topics/ui/layout/recyclerview).  More importantly, I know all the items in the list ahead of time.  Every single blog and tutorial always uses the same methodology. This ends up being a lot of boilerplate code.

* Create a view holder class
* Create a list adapter
* Attach the list adapter to the recyclerview
* Update the view holder class to implement the UI
* Check out the requirements around clicking
* Arrange for the list of items in the adapter to be updated

Even with the reduction in code from writing this in Kotlin, it's still a lot of code spread over three classes.  I'd prefer for it to be all in one class.  Recently, I was building a new app and the same problem surfaced, so I decided to fix it once and for all with an abstract adapter.

## The Abstract Adapter

Here is the theory.  Let's say I have an adapter that is always the same.  The only thing I have to do is specify a method for building out the UI.  Wouldn't this save time?

Here is the abstract adapter class:

```kotlin
package com.github.adrianhall.utils

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.annotation.LayoutRes
import androidx.recyclerview.widget.RecyclerView

abstract class SimpleListAdapter(@LayoutRes private val rowLayout: Int)
  : RecyclerView.Adapter<SimpleListAdapter.VH>()
{
  private val items = ArrayList<Any>()

  fun setItems(newItems: List<Any>) {
    items.clear()
    items.addAll(newItems)
    notifyDataSetChanged()
  }

  abstract fun onBindData(position: Int, viewHolder: RecyclerView.ViewHolder, data: Any)

  override fun onCreateViewHolder(parent, ViewGroup, viewType: Int): VH 
    = VH(LayoutInflater.from(parent.context).inflate(rowLayout, parent, false))

  override fun getItemCount(): Int
    = items.size

  override fun onBindViewHolder(holder: VH, position: Int) {
    this.onBindData(position, holder, items[position])
  }  

  class VH(itemView: View): RecyclerView.ViewHodler(itemView)
}
```

It's not a big class and it can be re-used again and again.  The essence is:

* Call `setItems()` to replace the current list with a new list of items.
* Implement `onBindData()` to bind the data within a single element of the list to the UI.
* Everything else is taken care of for you.

## Creating a list

When does the code necessary to create a list become?  It is also (in my opinion) simplified.  I only have to create the list adapter and define one method:

```kotlin
class MyFragment: Fragment() {
  private val vm by viewModel<MyViewModel>()

  override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
    val view = inflater.inflate(R.layout.fragment_my, container, false)

    // Initialize the recyclerview
    val list = view.findViewById<RecyclerView>(R.id.my_list)
    list.adapter = object : SimpleListAdapter(R.layout.my_list_item) {
      override fun onBindData(position: Int, viewHolder: RecyclerView.ViewHolder, data: Any) {
        this@MyFragment.bindDataToRow(viewHolder.itemView, data as MyData)
      }
    }

    // Populate the list from something in the view-model
    vm.listItems.observe(this, Observer { items -> 
      (list.adapter as SimpleListAdapter).setItems(items)
    }) 
  }

  private fun bindDataToRow(view: View, data: MyData) {
    val title = view.findViewById<TextView>(R.id.list_item_title)
    
    title.text = data.title
  }
}
```

This is a really simple example, but you can see the initialization is straight forward.  I pass the layout of an individual row as an argument, then I override the `onBindData()` method.  This is called for every row to bind the data to the row.  The new adapter doesn't know what sort of data is in your list, nor does it know what is in your layout.  You provide the binding.

To populate the list, you call `setItems()`.  This will wipe out the data that is there and replace it with the new data.

## Handling row clicks

You can easily handle row clicks by registering a click-listener within `bindDataToRow()` method:

```
  private fun bindDataToRow(view: View, data: MyData) {
    //...

    view.setOnClickListener { this.onClick(data) }
  }
```

## Why not a library?

There are a lot of RecyclerView replacements out there that implement better semantics for list building.  I find it isn't worth it.  The code needed to implement a simple list is small, and I would rather not bring in another dependency for a small amount of code.  It's difficult to keep up to date, and you never know what else is in the library.  Always consider the effects of the dependency before agreeing to it.  Some dependencies are good, but some will just cause issues.

## Next steps

Swiping left and right still takes a major amount of work to implement an `ItemTouchHelper`, so I will tackle that next as a reusable component.
