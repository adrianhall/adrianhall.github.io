---
title: "Converting types with Room and Kotlin"
categories:
  - Mobile
tags:
  - android
  - kotlin
---

I’ve been working on a personal project, trying to get to grips with the various [Android Architecture Components](https://developer.android.com/topic/libraries/architecture/) and Kotlin. One of the things I came up with was the requirement to deal with type conversion when using a SQLite database and the [Room persistence library](https://developer.android.com/training/data-storage/room/). Room is a nice abstraction to the internal SQLite database that converts models to tables within SQLite. It’s nice because it works alongside LiveData and RxJava to provide observable objects — when the database changes, the observable changes as well.

## Write a Room data access layer the normal way

Let me explain my problem with type conversion with an example. I’ve got a nice model:

{% highlight kotlin %}
import android.arch.persistence.room.ColumnInfo
import android.arch.persistence.room.Entity
import android.arch.persistence.room.PrimaryKey
import java.time.Instant
import java.util.*

/**
 * Definition of an Album.
 */
@Entity(tableName = "albums")
data class Album(
        @PrimaryKey
        @ColumnInfo(name = "id")         var id: String = UUID.randomUUID().toString(),
        @ColumnInfo(name = "created")    var created: Instant = Instant.now(),
        @ColumnInfo(name = "modified")   var modified: Instant = Instant.now(),
        @ColumnInfo(name = "deleted")    var deleted: Boolean = false,
        @ColumnInfo(name = "album_name") var name: String = "New Album",
        @ColumnInfo(name = "hidden")     var hidden: Boolean = false,
        @ColumnInfo(name = "pinned")     var pinned: Boolean = false
)
{% endhighlight %}

This is a fairly simple model for the Room persistence layer. However, I’m using the (relatively) new java.time package that is available in Java 1.8. Specifically, the `Instant` type (which represents a time zone agnostic moment in time) is not recognized by SQLite or Room.

Continuing on, I have a DAO:

{% highlight kotlin %}
import android.arch.paging.DataSource
import android.arch.persistence.room.*

@Dao
interface AlbumDao {
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun addAlbum(album: Album)

    @Update
    fun updateAlbum(album: Album)

    @Delete
    fun reallyDeleteAlbum(album: Album)

    @Query("SELECT * FROM albums WHERE NOT(deleted) AND NOT(hidden) ORDER BY pinned,album_name")
    fun listAlbumsByName(): DataSource.Factory<Int, Album>

    @Query("SELECT * FROM albums WHERE id = :id")
    fun getAlbumById(id: String): Album
}
{% endhighlight %}

This DAO does the normal CRUD operations. I have a funky custom select statement for dealing with the ordering of the albums. I want “pinned” albums to appear first and then in alphabetical order. I’m also using a data source as a return value here so I can deal with the paging adapter. Finally, here is my app database class:

{% highlight kotlin %}
import android.arch.persistence.room.Database
import android.arch.persistence.room.Room
import android.arch.persistence.room.RoomDatabase
import android.content.Context

@Database(entities = [ Album::class ], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {
    abstract fun albumDao(): AlbumDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
                INSTANCE ?: synchronized(this) {
                    INSTANCE ?: buildDatabase(context).also { INSTANCE = it }
                }

        private fun buildDatabase(context: Context) =
                Room.databaseBuilder(context.applicationContext, AppDatabase::class.java, "app.db").build()
    }
}
{% endhighlight %}

Again, this is a fairly normal — even boilerplate — implementation of the database class. It deals with building the database and is a synchronized singleton. I took this code directly from the [Google sample](https://github.com/googlesamples/android-architecture-components/blob/master/BasicRxJavaSampleKotlin/app/src/main/java/com/example/android/observability/persistence/UsersDatabase.kt).

## Add Converters

Compiling this, I get the following errors:

{% highlight text %}
error: Cannot figure out how to save this field into database. You can consider adding a type converter for it.
    private final java.time.Instant created = null;
error: Cannot figure out how to save this field into database. You can consider adding a type converter for it.
    private java.time.Instant modified;
{% endhighlight %}

This is not unexpected. As I mentioned earlier, SQLite doesn’t understand the `Instant` type, so it needs to be converted before being stored. Fortunately, the Room persistence library has provided a mechanism for this. First, create a class with a to/from pair of type converters:

{% highlight kotlin %}
import android.arch.persistence.room.TypeConverter
import java.time.Instant

class Converters {
    companion object {
        @TypeConverter
        @JvmStatic
        fun fromInstant(value: Instant): Long {
            return value.toEpochMilli()
        }

        @TypeConverter
        @JvmStatic
        fun toInstant(value: Long): Instant {
            return Instant.ofEpochMilli(value)
        }
    }
}
{% endhighlight %}

The important thing here is that they are annotated with both the `@TypeConverter` and `@JvmStatic` annotations. This comes from a peculiarity of Kotlin. When you place something in the companion object, it doesn’t appear as a normal static method. You can’t call `Converters.fromInstant()` from within a Java class. Instead, you have to call `Converters.Companion().fromInstant()`. The `Companion` here is the companion object. If, however, you annotate the method with `@JvmStatic` it will get the appropriate treatment to be a true static method of the `Converters` class.

Now that I have a set of type converters, I can add it to the application database class:

{% highlight kotlin %}
import android.arch.persistence.room.Database
import android.arch.persistence.room.Room
import android.arch.persistence.room.RoomDatabase
import android.arch.persistence.room.TypeConverters
import android.content.Context

@Database(entities = [ Album::class ], version = 1, exportSchema = false)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun albumDao(): AlbumDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
                INSTANCE ?: synchronized(this) {
                    INSTANCE ?: buildDatabase(context).also { INSTANCE = it }
                }

        private fun buildDatabase(context: Context) =
                Room.databaseBuilder(context.applicationContext, AppDatabase::class.java, "app.db").build()
    }
}
{% endhighlight %}

The important line here is line 8 — the @TypeConverters annotation. You can put as many converters as you want. Just include a to/from pair for the custom type.

## Fix the model class

There is another warning that creeps up:

{% highlight text %}
warning: There are multiple good constructors and Room will pick the no-arg constructor. You can use the @Ignore annotation to eliminate unwanted constructors.
{% endhighlight %}

If you have done any Room development with Kotlin, the likelihood is that you have run into this. This is because the de-facto advice is to use a data class as the model, such as I have done above. You can easily get rid of this warning by switching to a normal class. This is my converted class:

{% highlight kotlin %}
import android.arch.persistence.room.ColumnInfo
import android.arch.persistence.room.Entity
import android.arch.persistence.room.PrimaryKey
import java.time.Instant
import java.util.*

@Entity(tableName = "albums")
class Album {
    @PrimaryKey
    @ColumnInfo(name = "id")
    var id: String = UUID.randomUUID().toString()

    @ColumnInfo(name = "created")
    var created: Instant = Instant.now()

    @ColumnInfo(name = "modified")
    var modified: Instant = Instant.now()

    @ColumnInfo(name = "deleted")
    var deleted: Boolean = false

    @ColumnInfo(name = "album_name")
    var name: String = "New Album"

    @ColumnInfo(name = "hidden")
    var hidden: Boolean = false

    @ColumnInfo(name = "pinned")
    var pinned: Boolean = false

    override fun equals(other: Any?): Boolean {
        if (other == null)
            return false // null check
        if (javaClass != other.javaClass)
            return false // type check

        val mOther = other as Album
        return id == mOther.id
            && created == mOther.created
            && modified == mOther.modified
            && deleted == mOther.deleted
            && name == mOther.name
            && hidden == mOther.hidden
            && pinned == mOther.pinned
    }

    override fun hashCode(): Int {
        return Objects.hash(id, created, modified, deleted, name, hidden, pinned)
    }
}
{% endhighlight %}

Two of the things that the data class provides are the `equals()` and `hashCode()` methods. Since I am switching to a non-data class, I now need to provide those. This is actually not really a problem for me because I am going to be doing a `RecyclerView` with a `PagedListAdapter`. The `PagedListAdapter` requires me to provide a `Diffutil.ItemCallback` to compare two objects in the list. The best place to compare two objects is within the model class itself, so I end up extending the data class for this purpose.

Now my code compiles without warnings and I have bi-directional type conversion when storing data in SQLite. I can move on to my UI.

{% include links.md %}
