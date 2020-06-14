---
title: "Handle location and geocoding with SwiftUI"
categories:
  - Swift
tags:
  - SwiftUI
---

I'm continuing on my Weather app research in SwiftUI.  My next problem is this: 

How do I get the current location (as a longitude / latitude) and a place (like a city) in my app such that the UI is updated when location updates happen?

It turns out that this is a fairly easy problem, but it does come with some knowledge that you just have to know.  Here is the list:

* How can I update the UI based on programmatic changes to the value?
* How can I listen for location updates?
* How can I convert a location into a place (also known as geocoding)?

Let's tackle this one by one.  

## Update the UI with an ObservableObject

The first is handling updates to the UI based on programmatic changes to the value.  I have three values I need to deal with:

* The location
* The state of the location service
* The geocoding of the location

There are multiple ways to do this, including the ubiquitous environment object.  However, I'm going to use an `ObservableObject` for this function.  This isn't my app state.  I want to inject the observation wherever I need it, and I don't mind if the data is duplicated since the location is the location.

Here is an observable object:

{% highlight swift %}
import Foundation
import Combine

class LocationManager: NSObject, ObservableObject {
  let objectWillChange = PassthroughSubject<Void, Never>()

  @Published var someVar: Int = 0 {
    willSet { objectWillChange.send() }
  }

  override init() {
      super.init()
  }
}
{% endhighlight %}

This only has one variable.  However, when I change `someVar`, all the views that are hooked into it will be updated too.  That's the point of the `objectWillChange` and the `willSet` call. The import of `Combine` is what makes all this possible. 

For example, here is a `ContentView` that hooks into it:

{% highlight swift %}
import SwiftUI

struct ContentView: View {
    @ObservedObject var lm = LocationManager()

    var someVar: String  { return("\(lm.someVar? ?? 0)") }

    var body: some View {
        VStack {
            Text("someVar: \(self.someVar)")
            Button(action: { self.lm.someVar = self.lm.someVar + 2 }) {
              Text("Add more")
            }
        }
    }
}
{% endhighlight %}

When you click on the button, the `someVar` within the location manager will increment, which will in turn update the UI.  This gives us the basis for updating the UI programatically.

## Get the location with CLLocationManager

Next, let's update our `LocationManager` so that it actually handles the location.  This is done in two parts:

1. Set up a `CLLocationManagerDelegate` that will update the location.
2. Start listening for updates.

First, you have to add an entry into the `Info.plist`.  I'm going to ask for "when-in-use" permission to use the location.  You should always only ask for permissions you absolutely need, otherwise you risk losing your customers to privacy concerns.  The permission you want is labelled "Privacy - Location When In Use Usage".  Underneath, it's called `NSLocationWhenInUseUsageDescription`.  Add it to the `Info.plist` and fill in the value (as a string) with the reason for needing permission.

Now the preliminaries are over, let's look at that delegate:

{% highlight swift %}
extension LocationManager: CLLocationManagerDelegate {
    func locationManager(_ manager: CLLocationManager, didChangeAuthorization status: CLAuthorizationStatus) {
        self.status = status
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        self.location = location
        self.geocode()
    }
}
{% endhighlight %}

When the `CLLocationManager` (that we set up next) receives an authorization update, it will set the status.  When it receives a location update, it will set the location and kick off a geocode of the current location.  

Turning our attention to the `LocationManager` itself now, we need to remove `someVar` (since we don't need it any more), and replace it with the following:

{% highlight swift %}
import Foundation
import CoreLocation
import Combine

class LocationManager: NSObject, ObservableObject {
  private let locationManager = CLLocationManager()
  let objectWillChange = PassthroughSubject<Void, Never>()

  @Published var status: CLAuthorizationStatus? {
    willSet { objectWillChange.send() }
  }

  @Published var location: CLLocation? {
    willSet { objectWillChange.send() }
  }

  override init() {
    super.init()

    self.locationManager.delegate = self
    self.locationManager.desiredAccuracy = kCLLocationAccuracyBest
    self.locationManager.requestWhenInUseAuthorization()
    self.locationManager.startUpdatingLocation()
  }

  private func geocode() {
    // For later
  }
}
{% endhighlight %}

Together with the preceding listing (which I place in the same swift file), the location and status are updated.  The interesting stuff is in the `init()` code.  It should be self-explanatory.  The only difficult part is "where is the delegate?" - that's the extension methods we wrote earlier.

## Use the geocoder to turn location into a placemark

The final piece is to turn the location into the placemark.  I've already produced a dummy call to `geocode()` which is called whenever the location changes.  All I need to do now is to fill it in:

{% highlight swift %}
class LocationManager: NSObject, ObservableObject {
  private let geocoder = CLGeocoder()
  
  // Rest of the class

  @Published var placemark: CLPlacemark? {
    willSet { objectWillChange.send() }
  }

  private func geocode() {
    guard let location = self.location else { return }
    geocoder.reverseGeocodeLocation(location, completionHandler: { (places, error) in
      if error == nil {
        self.placemark = places?[0]
      } else {
        self.placemark = nil
      }
    })
  }
}
{% endhighlight %}

The `completionHandler` is an async return value.  The geocoder method will return immediately, but it will continue to work in the background.  When it is complete, it will call the completion handler with the results.  We then set the new placemark and the view observing this value will get the update.

## Use it in a view

Here is the view I use to test:

{% highlight swift %}
struct CityListScene: View {
    @ObservedObject var lm = LocationManager()

    var latitude: String  { return("\(lm.location?.latitude ?? 0)") }
    var longitude: String { return("\(lm.location?.longitude ?? 0)") }
    var placemark: String { return("\(lm.placemark?.description ?? "XXX")") }
    var status: String    { return("\(lm.status)") }

    var body: some View {
        VStack {
            Text("Latitude: \(self.latitude)")
            Text("Longitude: \(self.longitude)")
            Text("Placemark: \(self.placemark)")
            Text("Status: \(self.status)")
        }
    }
}
{% endhighlight %}

I'm still getting the values in the same way, but now I have access to the location and the placemark.  I like "promoting" the latitude and longitude to the be properties on the `CLLocation`.  By default, they are buried inside the `coordinate` property.  To do the promotion, I have the following code:

{% highlight swift %}
extension CLLocation {
    var latitude: Double {
        return self.coordinate.latitude
    }
    
    var longitude: Double {
        return self.coordinate.longitude
    }
}
{% endhighlight %}

This code is placed in the same place as the `LocationManager`.

## Final notes

The location manager doesn't work inside the preview window.  For this reason, I'm going to be switching to a "fake" location manager that is attached when running in debug mode.  This will allow me to inject a fake location by constructing the three variables I need.  You will always need to check the status (ensure the value is `.authorizedWhenInUse` or `.authorizedAlways`) and don't add the "current location" when you are not authorized.

When you run it in the simulator, you will be prompted for the permission to use location, then see the following:

![]({{ site.baseutl }}/assets/images/2019/2019-11-05-image1.png)

This is my first peek into the application state (and outside the UI), and it turned out to be really easy.